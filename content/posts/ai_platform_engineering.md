---
title: "AI Platform Engineering: what changes when agents become your platform's users"
date: 2026-09-07
description: "Notes from running an LLM agent as a production workload on Kubernetes and letting coding agents operate the platform: what stayed the same, what changed, and the lessons learned along the way."
author: "Matei Popa"
tags:
  - AI
  - Platform Engineering
  - Kubernetes
  - LLM
  - Agents
  - DevOps
images:
  - /images/ai_platform_engineering.png
---
![ai platform engineering](/images/ai_platform_engineering.png)

AI platform engineering is showing up in job titles, conference tracks and vendor decks, and it
means something different in each one. For us it turned out to be two jobs behind one name: running
LLM agents as production workloads, and letting agents operate the platform we already run. We have
been doing both for a while now, with an internal knowledge bot that lives in Slack and coding
agents that debug CI, review pull requests and bump vulnerable images. Here is what changed, what
didn't, and the lessons we learned along the way.

---

### Two jobs behind one title

The first job is building a platform for AI. Somebody writes an agent and it has to run somewhere. It
needs a container, secrets, a model endpoint, access to internal data, logs and a bill. That is a
platform problem, and it lands on the platform team whether or not anyone calls it AI.

The second job is using AI for the platform. Coding agents now read your repos, run your pipelines and
follow your runbooks. They are a new kind of user of your paved roads, one that doesn't get tired,
doesn't skim, and will happily run `terraform destroy` if the road lets it.

Both are the discipline you already know: golden paths, guardrails, and making the safe thing the easy
thing. The user changed, not the job.

---

### What didn't change

An agent is a container. Ours is built in CI, scanned, pushed to a private registry and deployed with a
Helm chart. It gets its secrets from Vault through the Vault Secrets Operator and its cloud access from
a pod identity, not from keys. Logs go to Loki, metrics to Prometheus, alerts to the same on-call
rotation as everything else. If your platform can run a Spring Boot service, it can run an agent.

Our internal bot is a good example of how boring this should be. It answers questions in Slack using the
wiki, the BI tool, the issue tracker and the dbt docs. On the cluster it is a `Deployment` with an init
container that builds the dbt catalog at startup, a `VaultStaticSecret` that syncs its API tokens, and a
config map with the model endpoint and model name. When we switched model providers, the change was two
values in `values.yaml`. Resist the urge to stand up a separate AI platform. The boring parts already
work.

---

### What did change

#### Agents get their own identity

The bot has its own service account, its own Vault role, its own read-only database user, and API
tokens issued to it and scoped to the tools it uses. Never a human's token. The audit trail then points
at the bot rather than at whoever set it up, and a leaked token takes out one bot instead of one
engineer's access. Read-only by default. Anything that writes goes behind an explicit allowlist.

#### Tool boundaries are the security boundary

The system prompt is not a control. Asking the model nicely not to delete things is a suggestion, and
prompt injection will arrive through the data it reads: a ticket description, a wiki page, a pull
request body. Enforce at the tool layer instead. Decide which Slack channels it may speak in, which
commands it may run, and which need a human approval. Our exec policy denies by default and logs every
denial, and those logs have been useful more than once. On the network side, a default-deny
`NetworkPolicy` allows egress only to the model endpoint and the tools it integrates with. Treat
retrieved content as untrusted input, because it is.

#### Context is infrastructure

This was the biggest lesson. When the bot gave a wrong answer, the fix was almost never the model or
the prompt. It was stale or missing knowledge: a table that got renamed, a service that moved, a
process that changed last quarter. So we moved the knowledge into git as small, versioned skills. Each
one is a plain markdown file that says where things live, what the quirks are, and what to do. The
bootstrap prompt shrank to a short pointer at the skills. Skills go through pull requests and deploy
like code. A side effect nobody predicted: our documentation got better, because it finally had a
consumer that fails loudly when it is wrong.

#### Evals are your integration tests

A set of questions with expected answers runs in CI whenever a skill changes. It is not science, but it
catches a forgotten table rename before Slack does. If you would not ship a service without tests, do
not ship a skill without evals.

#### Cost is an SLO dimension

Tokens are the new NAT gateway bill: invisible until they are not. A busy channel, a verbose tool
response or a retry loop turns a cheap bot into a line item. We set per-agent budgets and rate limits,
route simple questions to a smaller model, cache what can be cached, and graph tokens next to CPU. If
it is not on a dashboard, it is not managed.

#### Humans stay in the loop

Everything that changes state pauses for an approval: deploys, database writes, closing tickets. Not
because the agent is bad at it, but because a paved road with no guardrails is a fast way off a cliff.

---

### The other direction: agents as platform users

The second job snuck up on us. Once coding agents started working in our repositories, the platform
team's job became building paved roads for agents. That means repo-level skills that encode our
conventions: where the self-hosted runners live, how the build cache works, what the expiry policy is
for accepted CVEs. It means a pull request review checklist written as a skill, and incident playbooks
for HikariCP pool exhaustion and slow queries written so an agent can follow them.

The payoff is that the agent debugging a failed CI run reads the same runbook the on-call human would.
The CVE bump that used to eat an afternoon now shows up as a pull request with the scan output
attached. Your runbooks are now an API. Write them for a reader who takes every step literally and
never asks whether you really meant it.

Permissions for coding agents follow the same rules as the bot. Read-only tools by default, so `plan`
yes and `apply` no. Work on a branch. Never push to the default branch. Keep secrets out of the context
window entirely.

---

### Gotchas that actually hurt

- Stale docs produce confident nonsense. An out-of-date runbook is worse than none, because the agent
  follows it with total conviction. Version your knowledge, make the bot cite its source, and delete
  docs you don't maintain.
- Logs now contain prompts, and prompts contain data. The question someone asked the bot is in your
  log pipeline, along with whatever the tool returned. Retention and access control on logs matter more
  than they did for a stateless API.
- Cold starts are real. Building a catalog in an init container adds minutes to startup. Set startup
  probes accordingly, or the first rollout will look like an outage.
- Models churn. They get deprecated, prices change, a better one ships. Keep the model name and
  endpoint in configuration, and run your evals against the new one before you flip it.
- One giant token. The first version of every bot has one god token because it was quicker. Split it
  before production. You will not do it after.

---

### Where to start

If your team is being handed an AI project this quarter, this is the order we would do it in:

1. Run it like any other workload: image, chart, Vault, pod identity, Loki, Prometheus.
2. Give it its own identity and read-only credentials. Nothing shared with a human.
3. Put tool and network boundaries in place before the first user talks to it.
4. Move its knowledge into git as versioned skills, with evals in CI.
5. Put tokens on a dashboard and a budget on the tokens.
6. Then turn it around and write your own runbooks so an agent can follow them.

---

### Wrapping up

AI platform engineering is not a new stack. It is the platform engineering you already do, applied to
a user that reads everything, forgets nothing between deploys, and does exactly what the paved road
allows. Identity, boundaries, versioned context and cost visibility are the whole game. Every one of
those investments also makes the platform better for the humans.
