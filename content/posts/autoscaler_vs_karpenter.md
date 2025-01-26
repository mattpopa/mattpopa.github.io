---
title: "Cluster Autoscaler VS Karpenter"
date: 2025-01-26
author: "Matt Popa"
---

![microservices_vs_serverless](/images/microservices_vs_serverless.jpg)

# Proposal: Migration from Cluster Autoscaler to Karpenter for EKS Cluster

## Overview

The most common approach to scaling Kubernetes clusters in AWS is to use **Cluster Autoscaler (CA)**, which automatically adjusts the number of nodes based on pod resource requirements. However, CA has limitations when managing mixed-instance Auto Scaling Groups (ASG), leading to inefficiencies in scaling down nodes. To address these issues, we propose migrating to **Karpenter**, a Kubernetes-based node provisioning solution that offers advanced features like direct EC2 provisioning and customizability.
This document outlines the technical steps, pros and cons, and migration plan for transitioning from **Cluster Autoscaler (CA)** to **Karpenter** in an EKS environment. The change aims to address inefficiencies in scaling down mixed-instance Auto Scaling Groups (ASG) and improve provisioning flexibility by leveraging Karpenter’s advanced capabilities.

---

## Problem with Current Setup (Cluster Autoscaler)

The main issue stems from using **AWS ASG with a mixed-instance policy** alongside CA. While the scaling up works seamlessly, **scaling down does not honor CA’s selection of nodes**, leading to:
- **Uncoordinated node termination**: AWS ASG terminates nodes arbitrarily, ignoring CA preferences.
- **Resource wastage**: CA scales up new nodes despite underutilized existing nodes.
- **Operational friction**: Reliance on ASGs for scaling introduces limited control over granular provisioning.

Additionally, while CA is easy to configure and supports rolling updates, its limitations in handling mixed-instance scaling and reliance on ASG behavior create inefficiencies for dynamic workloads.
This is a known issue on CA's community and a bug has been opened to address it.

---

## Why Switch to Karpenter?

Karpenter eliminates dependency on ASGs, directly provisioning EC2 instances based on pod requirements and cluster state. This resolves the ASG misalignment issue during scaling down while introducing advanced features:
- **Direct EC2 provisioning**: Dynamically provisions nodes to meet specific pod requirements, bypassing ASGs.
- **Efficient scaling**: Automatically balances Spot and On-Demand instances, prioritizing cost-efficiency.
- **Customizability**: Supports CRDs for fine-grained control, allowing node-level customization.
- **Faster scheduling**: Reduces provisioning latency by optimizing scaling decisions.
- **Consolidation**: Removes underutilized nodes to optimize cluster resources.

### Notable Features:
- **Capacity-type fallback**: Prioritizes Spot Instances; falls back to On-Demand when Spot capacity is unavailable.
- **AWS Node Termination Handling**: Integrates features similar to [AWS Node Termination Handler](https://github.com/aws/aws-node-termination-handler) for gracefully handling Spot instance interruptions.
- **Rolling Updates**: Allows for seamless integration of new node types or configurations.

---

## Comparison: Cluster Autoscaler vs. Karpenter

| Feature                       | Cluster Autoscaler         | Karpenter                       |
|-------------------------------|----------------------------|----------------------------------|
| **Dependency on ASGs**        | Required                   | Not required                    |
| **Spot and On-Demand fallback** | Partial (via ASG strategy) | Native                          |
| **CRD Support**               | No                         | Yes                             |
| **Scaling Granularity**       | Limited by ASG policies    | Per pod                         |
| **Scaling Speed**             | Moderate                  | Faster                          |
| **Operational Overhead**      | Lower                      | Higher (CRDs)          |
| **Rolling Updates**           | Supported                 | Supported                       |
| **Integration with AWS APIs** | Indirect (ASG-level)       | Direct                          |
| **Flexibility**               | Lower                      | Higher                          |

---

## Implementation Plan

### Prerequisites
1. **Verify Subnet and Security Group Tags**:
    - Add `karpenter.sh/discovery: <cluster-name>` to:
        - Subnets used for node groups.
        - Security Groups associated with cluster resources.

2. **IAM Role for Karpenter**:
    - Create a **Karpenter Controller Role** with permissions to manage EC2 instances.
    - Create a **Karpenter Node Role** for instances provisioned by Karpenter.

3. **OIDC Provider**:
    - Ensure an OIDC provider is associated with the EKS cluster.

---

### Steps to Integrate Karpenter Using `terraform-aws-eks` Module

1. **Add Karpenter Module**:
    - Use the Karpenter submodule in the `terraform-aws-eks` module. This includes preconfigured templates for the controller, NodePools, and CRDs.

2. **Configure Karpenter**:
    - Define a **default NodePool** specifying:
        - Instance types: `m7i.2xlarge`, `m7i-flex.2xlarge`, `c7i.2xlarge`.
        - Capacity type preferences: `Spot` as primary with fallback to `On-Demand`.
        - Expiration policy: For temporary workloads, set `expireAfter`.

3. **Tagging Resources**:
    - Add required tags to all subnets and security groups as per the Karpenter documentation.

4. **Deploy Karpenter**:
    - Configure and deploy Karpenter using Helm or the provided Terraform module.
    - Include the `settings.clusterName` and `settings.interruptionQueue` parameters.

5. **Migrate Critical Workloads**:
    - Use nodeAffinity rules to restrict workloads like `coredns` and `metrics-server` to existing ASG nodes during testing.

6. **Testing and Rollout**:
    - Test with specific workloads to validate node provisioning and behavior.
    - Gradually scale down ASG-managed nodes to validate Karpenter’s efficiency.

---

### Migration Steps
1. **Configure Cluster Autoscaler**:
    - Restrict CA to manage only one ASG during the migration period.
    - Set the ASG minimum size to retain critical nodes.

2. **Deploy Karpenter in Parallel**:
    - Run Karpenter alongside CA, ensuring proper namespace isolation and resource tagging.

3. **Monitor and Validate**:
    - Use `kubectl logs -n karpenter` to monitor provisioning behavior.
    - Verify node scaling using `kubectl get nodes`.

4. **Gradual Transition**:
    - Slowly scale down CA-managed ASGs while relying more on Karpenter for node provisioning.
    - Disable CA once all workloads are verified under Karpenter.

---

## Risks and Mitigation
1. **Risk**: Misconfigured NodePool leading to resource mismatch.
    - **Mitigation**: Use specific tags for node requirements and validate configuration with dry runs.

2. **Risk**: Spot capacity shortages causing delays.
    - **Mitigation**: Enable On-Demand fallback in the NodePool CRD.

3. **Risk**: Resource contention during migration.
    - **Mitigation**: Use nodeAffinity to protect critical workloads.

---

## Conclusion

Migrating to Karpenter provides:
- Greater control over instance provisioning.
- Improved cost-efficiency and scaling speed.
- Elimination of ASG misalignment during scale-down.

While Karpenter introduces operational complexity, the benefits outweigh the limitations, particularly for dynamic workloads. The proposed Terraform-based approach ensures a smooth transition while allowing testing alongside Cluster Autoscaler.

---
