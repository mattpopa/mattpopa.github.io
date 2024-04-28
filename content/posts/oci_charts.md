---
title: "Embracing OCI for Helm Charts"
date: 2024-04-24
author: "Matt Popa"
---

![containers](/images/containers.jpg)

## A New Perspective on Helm Charts

Whale hello there (docker pun) DevOps enthusiasts! As someone who's been navigating the waters of 
AWS and Helm for years, I've largely relied on the tried-and-tested methods like ChartMuseum and S3 
backends for managing Helm charts. Occasionally, I've also pulled charts directly from version control 
when the stakes weren't too high on audit trails, because it's straightforward and gets the job done 
when conditions allow.

However, not long ago (or at least that's when I caught the wave), I discovered a relatively 
fresh approach that got my attention: using the Open Container Initiative (OCI)[https://opencontainers.org/about/overview/] standards 
for Helm chart repositories. Interesting to find out there's a new way to manage Helm charts, I thought. 

## Why the Shift?

While using AWS, it’s easy to fall into a routine, sticking with S3 because it's there and it works. 
But as projects grow and compliance requirements tighten—especially in environments like AWS GovCloud,
the need for audit and simplicity becomes apparent. Enter OCI on AWS Elastic Container Registry (ECR), 
a solution that treats Helm charts as first-class citizens alongside container images. 
This method isn’t just a fancy alternative; it's a streamlined, secure way to manage your deployments.

## The OCI Approach with Helm and Terraform

I enjoy working with helm provider within terraform, and happy to see that the tf provider supports
the OCI feature without any McGyver stuff required and integrating OCI with Helm and Terraform is streamlined.

Here's an example of how to use helm charts w/ OCI:

### Step 1: Set Up Your AWS ECR for Helm

First things first, let's create a repository in AWS ECR:

```
aws ecr create-repository --repository-name el-testo --region eu-west-2
```
We're using `aws` cli here for simplicity but you might want to use `terraform` for this, on a project.

Docs about pushing charts to ECR are also available [here](https://docs.aws.amazon.com/AmazonECR/latest/userguide/push-oci-artifact.html).

### Step 2: Work with Helm and Terraform

Assuming you've got your Helm charts ready to go, let's authenticate 

```
aws ecr get-login-password --region eu-gov-east-1 | helm registry login \
    --username AWS \
    --password-stdin your-account.dkr.ecr.eu-west-2.amazonaws.com
```

and push them as OCI artifacts:

```
helm chart push your-account.dkr.ecr.eu-west-2.amazonaws.com/el-testo:0.1.0
```

Caveat here is that the chart name is expected to be named the same as the repository name.

Now, integrating these charts into your Terraform workflows using the Helm provider is smoother 
than ever. Just specify your OCI-compatible chart in the Terraform configuration:

(in this example I have an EKS cluster provisioned with terraform and using `aws-iam-authenticator`
for authentication with OIDC, but you can use any other method for authentication with your EKS cluster)

```
provider "helm" {
  kubernetes {
    cluster_ca_certificate = base64decode(module.eltesto-eks-cluster.cluster_certificate_authority_data)
    host                   = module.eltesto-eks-cluster.cluster_endpoint
    exec {
      api_version = "client.authentication.k8s.io/v1"
      command     = "aws-iam-authenticator"
      args        = ["token", "-i", module.eltesto-eks-cluster.cluster_id]
    }
  }
}


resource "helm_release" "el-testo-oci-poc" {
  name       = "el-testo-release"
  namespace  = "poc"
  repository = "oci://your-account.dkr.ecr.eu-west-2.amazonaws.com"
  chart      = "el-testo"
  version    = "0.1.0"
}
```

## Conclusion: Looking Forward with OCI

Adopting OCI for Helm on AWS ECR represents more than just a shift in technology—it's a step towards 
more secure, manageable, and streamlined infrastructure operations. This method aligns particularly 
well with the stringent requirements of environments like AWS GovCloud, where compliance and security 
are paramount.

While the traditional methods of managing Helm charts—via ChartMuseum or directly from S3—are still 
effective, OCI offers a robust alternative that integrates seamlessly with existing AWS services and 
Terraform workflows. It’s especially beneficial for new projects where you can design your infrastructure 
with the best tools from the start.

For those of us in DevOps, it's essential to stay adaptive and be ready to incorporate new, efficient 
technologies that could enhance our systems' security and efficiency. OCI on ECR is a compelling option 
that promises just that, making it a worthwhile consideration for your next Kubernetes project.

So, as you plan your future projects or consider upgrades to your current systems, give OCI a thought. 
It might just be the enhancement you need to simplify and secure your deployments in the ever-evolving 
landscape of cloud infrastructure.
