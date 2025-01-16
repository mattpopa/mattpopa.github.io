---
title: "How to build a wordpress-as-a-service platform on AWS with Terraform"
date: 2025-01-08
description: "A practical guide to hosting a PHP blog on AWS using Terraform. Automate infrastructure, manage certificates, secure and scale."
author: "Matt Popa"
tags:
  - PHP
  - WordPress
  - AWS
  - Terraform
---
![wordpress hello](/images/wp_hello.webp)

Manually setting up PHP applications can be tedious and error-prone. By using AWS services and Terraform, we can automate our infrastructure setup, providing a scalable and secure environment for our PHP blog.

This guide will walk we through a basic architecture, considerations, and common pitfalls. At the end, we'll be able to host our blog with minimal effort and manage it like a pro.

---

### The Architecture

The setup consists of the following components:

1. **Application Load Balancer (ALB)**: Acts as the entry point for traffic, terminating HTTPS and forwarding requests to the backend servers.
2. **EC2 Instances**: Hosts our PHP application (e.g., WordPress).
3. **Amazon ACM**: Handles SSL/TLS certificates for HTTPS.
4. **Amazon Route 53**: Provides DNS management for our domain.
5. **Amazon RDS (optional)**: For databases if you’re building a larger, scalable blog.

---

### Why Terraform?

Using Terraform, we can define our infrastructure as code, enabling we to:

- Automate deployments and updates.
- Version control our infrastructure.
- Reuse and scale configurations.

---

### Key Steps

#### 1. Set Up the ALB
The ALB is configured to redirect HTTP traffic to HTTPS and route traffic based on hostnames. If we have multiple subdomains (e.g., `www.example.com` and `example.com`), we can also set up redirection rules to consolidate traffic.

#### 2. Install and Configure PHP
Use EC2 user data scripts to automate the installation of PHP, Nginx, and MariaDB. Configure the PHP-FPM process to work under the correct user (`nginx`).

#### 3. Enable HTTPS with ACM
AWS ACM manages our SSL certificates, which are automatically integrated with the ALB. This ensures our site is secure without manual certificate management.

#### 4. Automate Deployment with Terraform
Terraform provisions the ALB, listeners, EC2 instances, and security groups, so our infrastructure is consistent across environments.

---

### Pitfalls and Troubleshooting

#### Mixed Content Issues
Ensure our application properly sets the `HTTPS` flag when running behind the ALB. Update the PHP configuration to handle the `X-Forwarded-Proto` header from the ALB.

#### Redirect Loops
Avoid redirect conflicts by configuring only one layer to handle HTTP-to-HTTPS redirection. In this case, the ALB should handle it.

#### Directory Permissions
WordPress requires writable directories for uploads. Ensure the permissions are set correctly to avoid errors.

#### Instance Configuration Drift
Use user data scripts and configuration management tools to ensure consistent setup for EC2 instances.

---

### Final Thoughts

Hosting a PHP blog on AWS doesn’t have to be complex. By leveraging AWS services like ALB, ACM, and EC2, combined with Terraform, we can create a reliable, automated, and scalable hosting environment.

For the full Terraform configuration files and scripts, check out the [GitHub repository](https://github.com/mattpopa/cloudcat_infra). This repository contains everything we need to get started, from ALB configurations to user data scripts.

As a side note, hosting a PHP blog on AWS could become expensive if not managed properly. If redundancy isn't a concert, consider stop using a multi AZ setup, replace or don't use the NAT gateway, use a self-hosted DB to start with and size our EC2 instance well.

Happy hosting!
