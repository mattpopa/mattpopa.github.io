---
title: "Microservices VS Serverless"
date: 2024-04-14
author: "Matt Popa"
---

![microservices_vs_serverless](/images/microservices_vs_serverless.jpg)

In the dynamic world of software architecture, the debate between microservices and serverless 
functions (like AWS Lambda) is similar to the classic face-off between Ryu and Ken in the Street Fighter 
world. Both have their strengths, their unique moves, and their situational advantages. 
As developers, we're often in the position to make a choice between the two, so let's break down 
the details in a clear, concise, and human manner.

## Microservices: The Containerized Contender

Microservices are independent units of software that communicate over well-defined APIs. 
They are containerized, allowing developers to bundle their application along with all the necessary 
parts such as libraries and dependencies. This makes microservices highly portable, providing a consistent 
environment from local development to testing and production.

Testing microservices locally is straightforward because the container that runs on our machine is 
the same container that will be deployed to production. This predictability and consistency make 
microservices a solid choice for complex applications that require a clear separation of concerns and 
independent scaling.


## Serverless Functions: The Quick and Agile Fighter

Serverless functions, like AWS Lambdas, on the other hand, are event-driven and fully managed by 
cloud providers. They can scale automatically and you only pay for what you use, making them a 
cost-effective solution for many scenarios. However, testing Lambda functions locally can be a 
challenge, as replicating the exact cloud environment on a local machine is not always possible.

The unpredictability arises because Lambda functions depend on the cloud provider's environment, 
which can differ from the local development environment. This can lead to the "It works on my machine" 
syndrome, where code behaves differently in production than it does in development.

## Combining Forces: Serverless Containers

The game-changer arrives when we merge the portability of containers with the agility of serverless 
functions. By deploying Lambda functions as containers, we can bring the predictability of microservices 
to serverless. This hybrid approach allows developers to run their Lambda functions locally in a container, 
ensuring that the function will behave the same way when deployed to the cloud.

This strategy enhances the development experience and increases the predictability of deployments. 
We're using and deploying [Lambdas as containers](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html) 
in our projects for some time now, and it's been great for development and deployment consistency.

## Best of Both Worlds: An Example in Practice

In our projects, we predominantly use microservices for their reliability and scalability. 

But we're not bound by one architecture, and we switch to Lambda functions when we see a clear 
advantage in cost and performance. By deploying these functions as containers, we strike a balance 
between development consistency and cloud-native benefits.

For instance, we might develop a user authentication service as a microservice because of its 
complexity and need for continuous running. Conversely, for a feature like image processing, 
which is event-driven and can have sporadic usage patterns, a Lambda function would be more cost-efficient.

We deploy lambda functions as containers using [terraform](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lambda_function), 
and we're passing the image tag as a variable. Here's an example:

```hcl
locals {
  app_lambda_name = "app-lambda"
  # we're using a remote state to get the image repository
  # this way we minimize terraform risk and blast radius, while keeping the code DRY
  image_repo      = data.terraform_remote_state.main.outputs.app_service_repository_url
  image_uri = (
    # we want to have options in case we're only interested in testing infrastructure
    # or test a specific version of the lambda than the one passed in
    var.app_lambda_version != "" ? "${local.image_repo}:${var.app_lambda_version}" :
    # else try to use the latest deployed version
    try(data.aws_lambda_function.deployed_app_lambda.image_uri, "")
    # only a valid image_uri is required, the above conditional is optional for testing purposes
  )
}

## ##########################################################
## App Lambda Release
## ##########################################################

resource "aws_lambda_function" "app_lambda" {
  function_name = local.app_lambda_name
  role          = aws_iam_role.app_lambda_release_role.arn
  package_type  = "Image"
  image_uri     = local.image_uri
  memory_size   = 3072
  timeout       = 60

  ephemeral_storage {
    size = 2048
  }

  # we're using the same VPC as the EKS cluster
  vpc_config {
    subnet_ids = data.terraform_remote_state.main.outputs.core_private_subnets
    security_group_ids = [
      data.terraform_remote_state.main.outputs.eks_entrypoint
    ]
  }

  environment {
    variables = {
      # passing a key as a an environment variable like we do in containers
      APP_KEY = data.aws_secretsmanager_secret_version.api_context_key.secret_string
    }
  }
}
```

## Conclusion: It's not a zero-sum game

Just like a good fighter knows when to use a heavy punch or a quick kick, developers understand when 
to choose microservices or serverless functions. The winning strategy often lies in combining these 
two powerful architectures: using Lambda functions as containers. 
This approach not only improves development and deployment predictability but also takes advantage 
of serverless benefits when appropriate.

Embrace the strengths of both contenders, and choose your architecture based on the unique needs of 
your project. The goal is not to pick a side but to harness the best qualities of both to create 
robust, efficient, and scalable solutions.

Remember, whether it's Ryu or Ken, the best fighter is the one who adapts to the challenge at hand, 
just as the best architecture is the one that fits the problem you're trying to solve.
