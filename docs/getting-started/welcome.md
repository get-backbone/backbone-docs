---
title: "Welcome"
summary: "What Backbone is, what you get out of the box, and where to start."
---

<!-- markdownlint-disable-file MD033 -->
<!-- MD033 off: inline HTML is used for spacing where pure Markdown is insufficient. -->

Backbone is a production-ready starting point for building SaaS (Software as a Service) and distributed systems on AWS with Java and Quarkus.

Backbone is not a framework. It is a production system you copy into your organization and run.

You take the platform into your own GitHub org, deploy it into your own AWS accounts, and keep full ownership of the infrastructure, services, and data. The vendor has no access to your repository.

### Quick start

<Cards columns={3}>
  <Card title="Onboarding" icon="fa-flag" href="/docs/onboarding">
    Template repository, remotes, licence, and bootstraping.
  </Card>
  <Card title="Development" icon="fa-laptop-code" href="/docs/development">
    Build, test and run Backbone locally.
  </Card>
  <Card title="Operations" icon="fa-gears" href="/docs/operations">
    Deploy to AWS with GitHub Actions, OIDC, and CDK.
  </Card>
</Cards>

### What you get

- Floci-powered local development, with a stateless reference UI and BFF (Backend for Frontend) you can run immediately
- Six domain services (auth, actor, audit, notification, document, template)
- AWS CDK (Cloud Development Kit) on ECS Fargate (private-by-default networking, least-privilege IAM)
- GitHub Actions CI/CD with OpenID Connect (OIDC) to AWS, build-once images, and diffed ECS service deploys
- Metrics, traces, logs, and CloudWatch infrastructure monitoring
- Identity, edge protection, and audit / governance foundations (licence tier gates the higher-assurance options)

For the full capability breakdown, see [Platform features](/docs/features).

<br />

---

## Who Backbone is for

Backbone is the operational stack behind a modern SaaS company. It is built for teams that want to spend their time building product, not rebuilding the same infrastructure every startup eventually needs.

With Backbone, you inherit:

- secure-by-default architecture
- a horizontally scalable foundation
- the cross-cutting operational capabilities you'd expect from a mature distributed system: observability, tracing, monitoring, resilience, caching, and traffic control

Instead of assembling these concerns piece by piece, you start with a coherent production foundation that already has them wired in.

<br />

---

## How Backbone works

- Purchase a Backbone platform licence file from the [Backbone website](https://backbonehq.io/#pricing).
- Your licence tier determines which platform configuration options the bootstrap allows.
- Your organization is added as a Contributor on `backbone-platform` (read access to the vendor tree).
- Copy that tree into a private repository in your GitHub org (**Use this template**, not Fork). The vendor has no access to your repository. Steps: [Onboarding](/docs/onboarding).
- You own and develop that copy.
- Provision the provided CI/CD workflows in your GitHub account.
- Deploy into your AWS accounts. Development environments can run within AWS free tier limits.
- Pull later platform updates from `backbone-platform` into your repository when you choose.

You retain full ownership and control of infrastructure, services, deployments, and data.

<br />

---

## Licence tiers

Every signed licence tier includes the same domain services, CDK, and CI/CD spine. Higher tiers unlock configuration that costs more to run or that regulated operators typically require.

- **Foundation** is the full operational platform listed in [Platform features](/docs/features).
- **Growth** adds Backbone-provisioned customer-managed keys and optional Amazon Managed Prometheus, Grafana, and X-Ray.
- **Enterprise** adds high-availability endpoints, internal HTTPS, bring-your-own-key (BYOK), and governance evidence (CloudTrail, Object Lock, ALB access logs).

Per-environment configuration limits are described in full in [Platform features](/docs/features#licence-tiers-and-platform-configuration).

<br />

---

## Start here

These guides cover how Backbone is built, how it runs, and how to extend it.

<Cards columns={3}>
  <Card title="Development" icon="fa-laptop-code" href="/docs/development">
    Docker, Floci, tests, and Quarkus workflow.
  </Card>
  <Card title="Operations" icon="fa-gears" href="/docs/operations">
    Deployments, GitHub OIDC (OpenID Connect), AWS, Floci, CDK.
  </Card>
  <Card title="Security" icon="fa-shield-halved" href="/docs/security">
    Platform security posture, least privilege, documented trade-offs.
  </Card>
  <Card title="Compliance" icon="fa-clipboard-check" href="/docs/compliance">
    Control mapping for forked deployments and operator responsibilities.
  </Card>
  <Card title="Performance" icon="fa-gauge-high" href="/docs/performance">
    Performance test plan, outputs, phase summaries and conclusions.
  </Card>
  <Card title="Infrastructure" icon="fa-cloud" href="/docs/infrastructure">
    AWS deployment topology - networking, compute, datastores, edge delivery, and cost.
  </Card>
</Cards>

<br />

---

## Next steps

Evaluating? Start with [Security](/docs/security) and [Operations](/docs/operations). They show the platform's architecture, constraints, and operational model in the open.
