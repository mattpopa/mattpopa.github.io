---
title: "GitHub Actions, self-hosted runners with Scale-Sets tips and tricks"
date: 2024-02-29
author: "Matt Popa"
---

![gh actions on k8s](/images/steamboat_willie.jpg)

As someone who's spent a fair bit of time tinkering with CI/CD pipelines and automating infrastructure, 
I've really started to see the value in GitHub Actions, particularly after moving away from Jenkins. 
My journey took an exciting turn when I began experimenting with self-hosted GitHub Actions runners 
on Kubernetes, using the summerwinds actions runner controller. Things got even more interesting when 
I shifted to [scale-sets](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/deploying-runner-scale-sets-with-actions-runner-controller), 
thanks to GitHub embracing the ARC project.

Diving into Docker-in-Docker (DinD) has been mostly a smooth sail for me, bringing more perks than 
pitfalls to the table. It's made things a lot simpler and has been added a nice touch to our overall 
developer experience.

But, sometimes, DinD throws a curveball. Specifically, when using scale sets, Docker would
sometimes drag its feet on startup by a second or two. This tiny delay was enough to throw off a job
with a not-so-fun error message:
```
Checking docker version
  /usr/bin/docker version --format '{{.Server.APIVersion}}'
  Cannot connect to the Docker daemon at unix:///run/docker/docker.sock. Is the docker daemon running?
  '
  Error: Exit code 1 returned from process: file name '/usr/bin/docker', arguments 'version --format '{{.Server.APIVersion}}''.
```
It's not the best of error messages, and I didn't understand the randomness of it.

Here's how I fixed it:

1. Checked the runner's Dockerfile from `ghcr.io/actions/actions-runner` and got the entrypoint script,
called `run.sh`, saved it on my local repo next to my Dockerfiles.

   ![custom_dockerfiles](/images/custom-dockerfiles.png#center)

2. Added a loop function that checks in on docker before starting anything. So, at the top of the `run.sh`
script, I've added a function to give Docker a nudge and check if it's awake before letting anything 
else happen. This bit tries to run docker ps, giving it up to 30 tries before calling it quits. It's 
like saying, "Hey Docker, you up?"
    
    ```bash
    # run.sh file
    #!/bin/bash
    
    # add this function at the top of the file
    wait_for_docker() {
        echo "Waiting for Docker daemon to be ready..."
        retries=30
        while [ $retries -gt 0 ]; do
            docker ps &>/dev/null
            if [ $? -eq 0 ]; then
                echo "Docker daemon is ready"
                return
            else
                echo "Docker not ready yet. Retries left: $retries"
                ((retries--))
                sleep 1
            fi
        done
        echo "Failed to connect to Docker daemon after multiple retries."
        exit 42
    }
    
    # Check Docker readiness before starting anything
    wait_for_docker
    
    # ... rest of the script remains the same
    ```
   
3. Custom Runner Makeover: After editing the `run.sh` script, I gave my custom runner a new entry point 
to include our Docker wake-up call at the end of the Dockerfile.
    
   ```
    FROM ghcr.io/actions/actions-runner:latest
    # We're using the ARC image as a base image on which we install our custom tools
    
    
    # Install org dependencies
    RUN sudo apt-get update \
        && sudo apt-get install -y \
        apt-transport-https \
        ca-certificates \
        curl \
    # ... other dependencies
    
    # At the very end of our Dockerfile, we add our custom entrypoint
    COPY run.sh /home/runner/run.sh
    RUN sudo chmod +x /home/runner/run.sh
    ```

4.  Pushed the changes to my repo, created the new image and updated the runner's deployment to use it.

## Speeding Things Up with a Docker Mirror

Since our setup is like a revolving door with nodes coming and going, because of scaling in and out
based on demand, with an AWS Auto Scaling Group and Karpenter, holding onto a Docker cache is like 
trying to catch smoke with your bare hands. To keep things zippy and dodge some AWS NAT fees, I added 
a private [Docker mirror](https://docs.docker.com/docker-hub/mirror/). Since docker images are the 
bread and butter of our CI/CD pipelines, this was a no-brainer. It's also a great way to keep an eye
on our Docker traffic, since CI/CD pipelines are always pulling and pushing images.

After setting up a Docker registry and its buddy, a registry service in Kubernetes, my custom DinD 
runner image got a little facelift to point at our shiny new mirror:

    FROM docker:dind
    # Starting with the ARC image, we're adding our own special sauce
    
    # Pointing at our Docker registry cache
    ENV REGISTRY_CACHE_IP=10.1.1.10 # Swap this with your own registry address
    ENV REGISTRY_CACHE_PORT=5000
    
    # Making a home for Docker configs
    RUN mkdir -p /etc/docker
    
    # Telling Docker to use the pull-through registry cache
    RUN echo "{\"registry-mirrors\": [\"http://${REGISTRY_CACHE_IP}:${REGISTRY_CACHE_PORT}\"]}" > /etc/docker/daemon.json

## Wrapping Up

With a few tweaks here and there, we've managed to keep the ship sailing smoothly. Our CI/CD pipelines 
are more reliable and I never got the "Is docker ready?" error since this change. We're also looking 
at a lower AWS bill due to significantly lower NAT traffic after using the registry mirror. 
Here's to a better developer experience and a more efficient CI/CD pipeline! 🚢