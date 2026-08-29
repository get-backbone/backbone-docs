---
title: "Welcome"
summary: "What Backbone is, what you get out of the box, and where to start."
---

<!-- markdownlint-disable-file MD033 -->
<!-- MD033 off: inline HTML is used for spacing where pure Markdown is insufficient. -->

Backbone is a production-ready starting point for building SaaS (Software as a Service) and distributed systems on AWS with Java and Quarkus.

Backbone is not a framework. It is a production system you fork and run.

You fork the platform, deploy it into your own AWS accounts, and keep full ownership of the infrastructure, services, and data.

### Quick start

<Cards columns={3}>
  <Card title="Development" icon="fa-laptop-code" href="/docs/development">
    Run Backbone locally.
  </Card>
  <Card title="Operations" icon="fa-gears" href="/docs/operations">
    Deploy with GitHub Actions, OIDC, and CDK.
  </Card>
  <Card title="Infrastructure" icon="fa-cloud" href="/docs/infrastructure">
    AWS topology, datastores, and cost.
  </Card>
</Cards>

### What you get

- ECS (Elastic Container Service) Fargate deployment model
- GitHub Actions CI/CD with OIDC (OpenID Connect) based AWS access
- AWS CDK (Cloud Development Kit) infrastructure code
- Floci-powered local development
- Authentication, audit, notification, and document services
- Prometheus and Grafana observability
- A reference web application you can run locally or deploy immediately

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
- Your licence **tier** determines which platform configuration options the bootstrap wizard offers (see [Platform features](/docs/features#licence-tiers-and-platform-configuration)).
- Your organization is added as a Contributor, so you can fork the `backbone-platform` repository.
- You own and develop that forked codebase.
- Provision the provided CI/CD workflows in your GitHub account.
- Deploy into your AWS accounts. Development environments can run within AWS free tier limits.
- Receive future platform updates by syncing your fork to the upstream `backbone-platform` repository.

You retain full ownership and control of infrastructure, services, deployments, and data.

<br />

---

## Start here

These guides cover how Backbone is built, how it runs, and how to extend it.

<Cards columns={3}>
  <Card title="Development" icon="fa-laptop-code" href="/docs/development">
    Local setup, tooling, licence, and Quarkus workflow.
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
