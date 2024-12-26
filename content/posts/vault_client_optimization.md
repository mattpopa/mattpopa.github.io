---
title: "Hashicorp Vault Client Optimization"
date: 2024-12-26
author: "Matt Popa"
---

![Hashicorp Vault](/images/vault.jpg)

# Optimizing Vault Costs with `serviceaccount_name` for Identity Aliases

Hashicorp Vault SaaS bills on client usage, and a client is everything from a user to a machine. More on this [here](https://developer.hashicorp.com/vault/docs/concepts/client-count). The problem is that when a developer removes a chart and redeploys it, vault will count the Kubernetes ServiceAccount as a distinct new app, which isn't quite true, because it's the same app we had before and we were billed for.

When configuring Vault with Kubernetes authentication, the `alias_name_source` parameter plays a critical role in defining how identity aliases are generated. By default, Vault uses `serviceaccount_uid`, but switching to `serviceaccount_name` can significantly reduce costs and simplify operations in certain scenarios.

Here’s a breakdown of the pros and cons of using `serviceaccount_name` over `serviceaccount_uid` and how it impacts Vault's cost optimization. Ref: [alias_name_source](https://developer.hashicorp.com/vault/api-docs/auth/kubernetes#alias_name_source)

---

## Why `serviceaccount_name` Reduces Costs

1. **Lowered Alias Churn**:
    - Using `serviceaccount_uid` ties identity aliases to unique, system-generated UIDs. These UIDs change whenever a ServiceAccount is recreated, leading to the creation of new aliases in Vault.
    - Each alias incurs storage and operational overhead. Over time, environments with high churn (e.g., CI/CD pipelines or ephemeral workloads) can accumulate significant costs.

   **Solution**: Switching to `serviceaccount_name` binds the alias to the human-readable ServiceAccount name, which remains consistent even after recreation. This avoids unnecessary alias generation, reducing Vault's storage usage and API calls.

2. **Simplified Policy Management**:
    - With `serviceaccount_name`, policies can directly reference stable names, streamlining configuration.
    - This reduces the need for frequent policy updates caused by ServiceAccount UID changes, saving administrative effort and minimizing configuration drift.

3. **Optimized Performance**:
    - Fewer aliases mean faster lookup times during authentication, reducing latency for applications and users.

---

## Pros of `serviceaccount_name`

- **Cost Savings**: Reduces storage and API request overhead by minimizing alias churn.
- **Predictability**: Names are stable and human-readable, improving debugging and policy management.
- **Compatibility**: Aligns better with environments leveraging long-lived ServiceAccounts.

---

## Cons of `serviceaccount_name`

- **Risk of Name Collisions**: If ServiceAccounts share the same name across namespaces, this can introduce conflicts. Ensure namespaces are considered in your configuration or choose unique naming conventions.
- **Less Granular Identity**: UIDs are unique by design, while names might not fully represent individual resources. In environments where fine-grained identity is critical, `serviceaccount_uid` might still be preferable.

---

## Best Practices for Implementation

1. **Evaluate Environment Needs**:
    - Use `serviceaccount_name` for environments with frequent ServiceAccount recreation or high alias churn.
    - Stick with `serviceaccount_uid` for highly secure environments requiring strict uniqueness.

2. **Monitoring and Auditing**:
    - Regularly audit identity aliases in Vault to identify and clean up unused entries, further optimizing costs.

---

Switching to `serviceaccount_name` is a practical choice for many use cases, particularly in cost-conscious or highly dynamic Kubernetes environments. However, ensure that your team understands its implications to balance simplicity, cost-efficiency, and security requirements.

By adopting this approach thoughtfully, you can optimize Vault's usage while maintaining a robust and scalable authentication system.

--- 
