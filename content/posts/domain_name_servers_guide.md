---
title: "A Beginner's Guide to DNS"
date: 2024-05-26
author: "Matt Popa"
---

![DNS](/images/dns2.jpg)

### A Beginner's Guide to DNS - From Domain Purchase to Establishing Your Online Presence

#### The Common Query: What Happens After Buying a Domain?

As an IT professional with a sysadmin background and cloud infrastructure focus, I often get DNS-related
questions from programmers and colleagues who are just starting out with their own websites.
One common confusion that arises post-purchase is about the role and setup of DNS hosting. This post 
aims to clarify these concepts, using my own experience with setting up `https://cloudcat.digital` 
as a practical example.

#### What is DNS, and Why is DNS Hosting Important?

The Domain Name System (DNS) is fundamentally the mechanism that translates human-friendly domain
names (like `cloudcat.digital`) into IP addresses that computers use to identify each other on the 
network. Think of DNS as the internet's yellow pages book (remember phone books?). When you type a 
URL into your browser, DNS servers take that domain name and translate it to the IP address of the 
web server hosting that site.

**DNS hosting** is a service that helps manage this process. It stores and manages the instructions 
that guide how your website’s name is translated into numbers (IP addresses) and how people find your 
website. These instructions are known as DNS records. They direct web traffic, route email and more. 
Good DNS hosting is important for several reasons: 

- **Reliability**: It makes sure that your website is always available to visitors, without interruptions.
- **Speed**: It controls how quickly your website can be found and accessed when someone types in your address.
- **Security**: Protects against DNS attacks and ensures your domain's records are safely managed.

#### Setting Up DNS Hosting: A Step-by-Step Example

To illustrate, let's look at how I set up DNS hosting for `cloudcat.digital` using Namecheap(.com,
not affiliated nor advertising -- just used as an example) for domain purchase and AWS Route 53 for 
DNS hosting:

1. **Purchase the Domain**:
    - I bought `cloudcat.digital` from Namecheap. You can use any domain registrar. Some popular
options include GoDaddy, Namecheap, dedicated country code registrars (for example, `.co.uk` or `.ro`).

2. **Choose a DNS Hosting Provider**:
    - Buying a domain name doesn't include DNS zone hosting. It's like buying a car -- doesn't come
with a parking space or a garage. Although Namecheap offers DNS hosting, I opted for AWS Route 53
because I already use AWS for other services.

3. **Configure DNS Records in AWS Route 53**:
    - **Create a Hosted Zone**: In AWS Route 53, I created a new hosted zone for `cloudcat.digital`. 
This zone holds all the DNS records for the domain.
    - **Set Name Server (NS) Records**: Route 53 automatically assigns NS records to your hosted zone. 
Update these NS records in Namecheap to point to AWS Route 53, effectively transferring DNS management
responsibilities. This is important if you want to manage your DNS records outside of where you bought
the domain.
    - **Create A type Records**: For GitHub Pages hosting, set up A records pointing to GitHub’s servers. 
These records ensure that when someone types `cloudcat.digital`, they are directed to the GitHub page 
hosting the site’s content. It's very similar to linking your own service or some 3rd party like Shopify,
Wix, Firebase, etc. I used terraform to create the records, but it can be done manually in the Route 53
or the UI of your DNS hosting provider too. Here's an infrastructure as code [example](https://github.com/mattpopa/cloudcat_infra/blob/main/cloudcat.tf)
of DNS records for `cloudcat.digital`.

4. **Deploy Your Website**:
    - I used Hugo with the PaperMod theme to generate the site's content, pushing it to a GitHub repository
set up to serve GitHub Pages (free hosting for static sites). If you're interested in how to host a 
static website for free and relatively fast, check out [GitHub Pages](https://pages.github.com/).

#### Conclusion: Simplifying DNS for Your Projects

DNS is like a yellow pages book -- a mapping of names and numbers. Understanding DNS and setting up 
DNS hosting are foundational to establishing a reliable and effective online presence. By choosing the 
right DNS hosting provider and configuring your DNS records correctly, you can ensure that your domain 
not only points to your website but also handles traffic efficiently and securely. This guide should 
demystify some of the complexities around DNS and help you smoothly transition from domain buyer to 
website owner.
