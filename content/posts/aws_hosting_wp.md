---
title: "How to Host a PHP Blog on AWS with Terraform"
date: 2025-01-08
description: "A practical guide to hosting a PHP blog on AWS using Terraform. Automate infrastructure, manage certificates, and scale seamlessly."
categories:
  - AWS
  - Terraform
  - DevOps
tags:
  - PHP
  - WordPress
  - AWS
  - Terraform
---

## Hosting a PHP Blog on AWS: Automate and Scale with Ease

Manually setting up PHP applications can be tedious and error-prone. By using AWS services and Terraform, you can automate your infrastructure setup, providing a scalable and secure environment for your PHP blog.

This guide will walk you through a basic architecture, considerations, and common pitfalls. At the end, you'll be able to host your blog with minimal effort and manage it like a pro.

---

### The Architecture

The setup consists of the following components:

1. **Application Load Balancer (ALB)**: Acts as the entry point for traffic, terminating HTTPS and forwarding requests to the backend servers.
2. **EC2 Instances**: Hosts your PHP application (e.g., WordPress).
3. **Amazon ACM**: Handles SSL/TLS certificates for HTTPS.
4. **Amazon Route 53**: Provides DNS management for your domain.
5. **Amazon RDS (optional)**: For databases if you’re building a larger, scalable blog.

---

### Why Terraform?

Using Terraform, you can define your infrastructure as code, enabling you to:

- Automate deployments and updates.
- Version control your infrastructure.
- Reuse and scale configurations.

---

### Key Steps

#### 1. Set Up the ALB
The ALB is configured to redirect HTTP traffic to HTTPS and route traffic based on hostnames. If you have multiple subdomains (e.g., `www.example.com` and `example.com`), you can also set up redirection rules to consolidate traffic.

#### 2. Install and Configure PHP
Use EC2 user data scripts to automate the installation of PHP, Nginx, and MariaDB. Configure the PHP-FPM process to work under the correct user (`nginx`).

#### 3. Enable HTTPS with ACM
AWS ACM manages your SSL certificates, which are automatically integrated with the ALB. This ensures your site is secure without manual certificate management.

#### 4. Automate Deployment with Terraform
Terraform provisions the ALB, listeners, EC2 instances, and security groups, so your infrastructure is consistent across environments.

---

### Pitfalls and Troubleshooting

#### Mixed Content Issues
Ensure your application properly sets the `HTTPS` flag when running behind the ALB. Update the PHP configuration to handle the `X-Forwarded-Proto` header from the ALB.

#### Redirect Loops
Avoid redirect conflicts by configuring only one layer to handle HTTP-to-HTTPS redirection. In this case, the ALB should handle it.

#### Directory Permissions
WordPress requires writable directories for uploads. Ensure the permissions are set correctly to avoid errors.

#### Instance Configuration Drift
Use user data scripts and configuration management tools to ensure consistent setup for EC2 instances.

---

### Final Thoughts

Hosting a PHP blog on AWS doesn’t have to be complex. By leveraging AWS services like ALB, ACM, and EC2, combined with Terraform, you can create a reliable, automated, and scalable hosting environment.

For the full Terraform configuration files and scripts, check out the [GitHub repository](https://github.com/mattpopa/cloudcat_infra). This repository contains everything you need to get started, from ALB configurations to user data scripts.

Happy hosting!
