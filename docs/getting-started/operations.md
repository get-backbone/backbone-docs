---
title: "Operations"
summary: "Operational tasks that sit outside day-to-day service development."
---

Tasks that sit outside day-to-day service development: local metrics (Prometheus and Grafana), CDK against AWS or Floci, and one-time GitHub Actions OIDC bootstrap.

For first-time setup (repository, licence, bootstrap), see [Onboarding](/docs/onboarding).  
For running Quarkus locally, see [Development](/docs/development).  
For a comprehensive task index, see [cheatsheet.md](/docs/cheatsheet).

For step-by-step operational runbook procedures (for example ECS native vs JVM, self-hosted runners for image builds, and other topics as they are added), see [RUNBOOK.md](/docs/runbook).

## Table of contents

- [GitHub Setup](#github-setup)
- [AWS Setup](#aws-setup)
  - [Development Environment `backbone-sandbox` profile](#1-development-environment-backbone-sandbox-profile)
  - [Domain DNS configuration](#2-domain-dns-configuration)
  - [Managed observability](#3-managed-observability)
- [CDK and AWS](#cdk-and-aws)
- [CDK and Floci](#cdk-and-floci)
- [Local metrics](#local-metrics)

---

## GitHub Setup

Backbone pipelines run on free tier GitHub plans. Most workflows complete in a few minutes, with all workflows currently completing in under 10 minutes. Paid business plans will allow you to optimize runners and reduce build times further.

### 1. Setup AWS IAM resources for GitHub Actions OIDC

Each environment account (INT, STAGE, PROD) needs `GitHubRoleStack` once before GitHub Actions can assume `backbone-github-actions-ci` via OIDC. There is no CDK pattern that avoids this: **something** must authenticate to AWS before the role exists. From a machine already logged into that account (SSO, IAM user, or similar):

```bash
BACKBONE_STAGE_ENV=INT task cdk:synth
BACKBONE_STAGE_ENV=INT task aws:deploy-github-role
# Repeat with BACKBONE_STAGE_ENV=STAGE and PROD against those AWS accounts when required
```

OIDC trust is scoped to `github.organization` in `platform-config.yml` (set via `task bootstrap:platform-config`; see [Onboarding](/docs/onboarding#5-client-platform-infrastructure-configuration) if you have not run it yet). The role allows Actions from `backbone-*` repos in that org (classic and immutable `sub` forms). After changing the organization, re-run the deploy above for each account.

The INT role includes ECR push to the central registry. STAGE/PROD roles can force-deploy ECS and run CDK in their accounts, but do not push images. Paid GitHub plans can later add deployment protection (e.g. Environment required reviewers on PROD) without changing this bootstrap.

### 2. Setup GitHub Actions Variables and Secrets

**Required repository variables** (Settings > Secrets and variables > Actions > Variables):

- `AWS_REGION` - region for INT / STAGE / PROD (same region for all promotion stages)
- `INT_AWS_ACCOUNT_ID` - AWS account ID for the INT stage (central ECR registry; image push and INT deploys)
- `STAGE_AWS_ACCOUNT_ID` - AWS account ID for the STAGE stage
- `PROD_AWS_ACCOUNT_ID` - AWS account ID for the PROD stage
- `BACKBONE_ECS_RUNTIME_MODE` (optional) - `jvm` (faster image build; slower container startup) or `native` (long builds; smaller images; better cold start).
- `USER_BACKBONE_DEPLOY` - GitHub username for the account that owns `secrets.PAT_BACKBONE_DEPLOY` (workflows pass it as `GITHUB_MAVEN_USERNAME` to `configure-maven-github.sh`)
- `BACKBONE_LICENCE_TIER` - `foundation`, `growth`, or `enterprise`. Must match the tier in your signed licence. CDK synth uses this for entitlement checks.

Workflows select the target account from those vars (e.g. INT for ECR push; INT/STAGE/PROD from a `stage_env` choice). CDK reads the same names from the job environment for central ECR ARNs and the pull allow-list.

**Required GitHub Actions Secrets** (Settings > Secrets and variables > Actions > Secrets > Repository secrets):

Backbone-related secrets:
- `PAT_BACKBONE_DEPLOY` - GitHub token for deployment
- `BACKBONE_LICENCE` - Contents of the signed client licence file (received from the vendor; not generated in this repo). Required at CDK synth to populate the Secrets Manager secret consumed by ECS tasks.
- `SMALLRYE_CONFIG_SECRET_KEY` (base64 encoded) - Encryption key for secrets.properties
- `GPG_PRIVATE_KEY` - GPG private key for signing artifacts during releases; see [03-release-bump.yml](https://github.com/get-backbone/backbone-platform/blob/main/.github/workflows/03-release-bump.yml)
- `GPG_PASSPHRASE` - GPG passphrase for signing artifacts during releases

External third-party secrets:
- `NVD_API_KEY` - NVD API key for vulnerability scanning; see [NIST - Request an API Key](https://nvd.nist.gov/developers/request-an-api-key)
- `OSS_INDEX_API_KEY` - Sonatype Guide / OSS Index API token for weekly CI dependency scanning only (local OWASP skips OSS Index by default); see [Sonatype Guide](https://guide.sonatype.com/) and [OSS Index analyzer](https://dependency-check.github.io/DependencyCheck/analyzers/oss-index-analyzer.html)
- `CODECOV_TOKEN` - Codecov token for coverage reporting; see [Codecov](https://codecov.io/)

Required for Login with Google (and OAuth refresh-token storage):
- `GOOGLE_OAUTH2_CLIENT_ID` - Google OAuth 2.0 Web client ID; see [Google Cloud Console credentials](https://console.cloud.google.com/apis/credentials)
- `GOOGLE_OAUTH2_CLIENT_SECRET` - Google OAuth 2.0 Web client secret
- `BACKBONE_OAUTH2_REFRESH_TOKEN_ENCRYPTION_KEY` - Encryption key for OAuth2 refresh token encryption (`openssl rand -base64 32`)

  Optional secrets: If you wish to retain frontend ability to 'Login with LinkedIn' you must specify:
- `LINKEDIN_OAUTH2_CLIENT_ID` - LinkedIn Oauth2 client ID; see [LinkedIn - OAUTH 2.0 Overview](https://learn.microsoft.com/en-gb/linkedin/shared/authentication/authentication)
- `LINKEDIN_OAUTH2_CLIENT_SECRET` - LinkedIn Oauth2 client secret

---

## AWS Setup

Because AWS environments vary widely across clients — especially in enterprise contexts — this project relies on a minimal, profile-based configuration (access key, secret, and region) to avoid imposing assumptions about account structure, IAM policies, or organizational setup.

For greenfield teams or startups without established AWS conventions, it is recommended to bootstrap your AWS environments using [Superwerker](https://github.com/superwerker/superwerker), developed by AWS Advanced Partners. Superwerker provides a well-architected baseline with sensible defaults around multi-account structure, security boundaries, and blast radius management.

Backbone does not subscribe to or replace an AWS landing zone model. Every client account layout differs. When you already operate a landing zone (Superwerker, Control Tower, or a custom security OU with a central log archive), map Backbone into that foundation rather than reshaping the organization around Backbone.

Backbone local development uses Floci (Docker) on `:4566` for AWS APIs including Cognito.
Deploy Cognito and Domain DNS into real AWS only for INT/DEV/PROD (or when you need live Cognito outside Floci);
that path is free-tier eligible — see [Domain DNS configuration](#2-domain-dns-configuration) and `task aws:deploy-cognito` under [CDK and AWS](#cdk-and-aws).

> 💡 **Note:** Backbone currently supports deployment within a single AWS region per environment, including multi-AZ infrastructure patterns for high availability inside that region.
>
> Multi-region deployment, replication, and failover patterns are planned roadmap areas and are expected to evolve alongside enterprise operational requirements. The current single-region posture is intentionally pragmatic for most early-stage and mid-scale deployments, where operational simplicity is typically more valuable than cross-region complexity.

### 1. Development Environment `backbone-sandbox` profile

In `~/.aws/credentials`, add a profile named `backbone-sandbox` with the following contents:

```bash
[backbone-sandbox]
aws_access_key_id=********************
aws_secret_access_key=********************
region=us-west-2
```

### 2. Domain DNS configuration

`DomainStack` provisions a Route 53 public hosted zone for the environment subdomain (`<env>.<domainRoot>`) plus an ACM certificate validated by DNS.
ACM writes its validation records into the new subdomain hosted zone, but those records are only publicly resolvable once the root domain (`domainRoot` in `config/src/main/resources/platform-config.yml`) delegates the subdomain.
Until then the certificate stays in `PENDING_VALIDATION` and the deploy blocks.
When deploying `DomainStack` into an environment for the first time, delegate by adding an `NS` record for the subdomain on the root domain, using the four name servers Route 53 assigned to the subdomain hosted zone:

```bash
# List the delegation name servers for the subdomain hosted zone
aws route53 list-hosted-zones-by-name --dns-name "<env>.<domainRoot>" \
  --query 'HostedZones[0].Id' --output text \
| xargs -I {} aws route53 get-hosted-zone --id {} \
  --query 'DelegationSet.NameServers' --output text
```

At the root domain's DNS provider, create an `NS` record named for the subdomain label (e.g. `int`) whose value is those four name servers. When the parent domain is itself a Route 53 hosted zone, add the record there.
After delegation propagates, ACM completes validation automatically and the deploy proceeds.

When DomainStack deployment succeeds, you should receive a notification 'DKIM setup SUCCESS for `<env>.<domainRoot>` in the specified region.'

**References**:
- Route 53 subdomain delegation: <https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/CreatingNewSubdomain.html>
- ACM DNS validation: <https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html>

### 3. Managed observability

Skip when managed observability is disabled for that environment. When enabled, CDK deploys `ObservabilityStack` (AMP + AMG). Identity Center is only required for Grafana sign-in, not for CDK or other Backbone stacks.

#### Identity Center

Set up IAM Identity Center in the account that hosts AMG so operators can authenticate to the Grafana workspace.

1. Enable **IAM Identity Center** (built-in directory is fine for sandbox).
2. Create a user or group.
3. Create a permission set (e.g. `AdministratorAccess`).
4. Assign user/group to the workload account; wait until the assignment completes.
5. Note the **access portal** URL (Identity Center → Dashboard).

AMG needs an **organization-level** Identity Center instance. Account SSO assignment grants AWS Console access only — not Grafana workspace access (see [Workspace access](#workspace-access)).

#### X-Ray OTLP bootstrap

Run once per account and region so Backbone apps can export OTLP traces (otherwise export returns HTTP 400).

```bash
task aws:bootstrap-xray
```

Idempotent when destination is already `CloudWatchLogs` / `ACTIVE`. Operator IAM needs `logs:PutResourcePolicy` and `xray:UpdateTraceSegmentDestination`.

#### Deploy

Deploy AMP and AMG so metrics and dashboards are available. No dependency on Runtime or hibernated stacks.

```bash
BACKBONE_STAGE_ENV=INT task cdk:synth
BACKBONE_STAGE_ENV=INT task aws:deploy-observability
```

Use `backbone-sandbox` or any profile with deploy permissions; Identity Center is not required for this step.

#### Workspace access

After deploy, grant AMG workspace roles so users can open Grafana (empty `list-permissions` causes redirect loops). CDK does not assign Grafana users. **Re-run after every ObservabilityStack recreate** (hibernate/redeploy creates a new workspace ID; permissions do not carry over).

```bash
task aws:grant-observability -- you@example.com
```

Optional: `BACKBONE_STAGE_ENV` (default `INT`), `AMG_ROLE` (`ADMIN` | `EDITOR` | `VIEWER`). List workspaces and permissions: `./scripts/aws/amg-grant-workspace-access.sh list`.

Open the workspace URL (`Backbone-OBSERVABILITY-AMG-WORKSPACE-URL`) → **Sign in with AWS IAM Identity Center** (Identity Center password, not `backbone-sandbox` access keys).

After first deploy, complete the AMG datasource and X-Ray plugin checklist in [Runbook §8](/docs/runbook#8-amg-post-deploy-checklist).

---

## CDK and AWS

### Standard CDK lifecycle

```bash
task cdk:build
task cdk:test
task cdk:synth
task cdk:diff
```

The client mirror does not include `infra/cdk.context.json`; run `task cdk:synth` once with AWS credentials in your account so CDK can cache account-specific lookups (availability zones, managed prefix lists) locally.

### AWS environment

GitHub Actions needs GitHubRoleStack in the target account first; bootstrap it manually once per INT / STAGE / PROD account (see [GitHub Setup](#1-setup-aws-iam-resources-for-github-actions-oidc)).

DEV is the usual stage for local CDK defaults; CI/CD runs with `BACKBONE_STAGE_ENV=INT` (see [01-infra-bootstrap.yml](https://github.com/get-backbone/backbone-platform/blob/main/.github/workflows/01-infra-bootstrap.yml)).

Use this section’s `task aws:*` tasks for AWS CDK deploys — the same shape as CI. Use [CDK and Floci](#cdk-and-floci) for Floci-only CDK development.

```bash
task cdk:install                                   # npm install in infra/ (CI and real AWS)
task aws:bootstrap                                 # CDKToolkit in workload region (backbone-sandbox profile) + us-east-1

task aws:deploy-all                                # deploy all stacks (AWS)
task aws:deploy-ecr                                # deploy ECR stack (INT)
task aws:deploy-cognito                            # deploy Cognito (INT needs GitHubRole first)
task aws:deploy-domain                             # deploy Domain stack (requires manual DNS updates)
```

---

## CDK and Floci

CDK work is usually done against AWS free tier. Floci `cdklocal` is supported for development and testing.

```bash
task cdk:install                                   # install Node dependencies and verify cdklocal
task cdk:bootstrap                                 # Floci cdklocal bootstrap

task cdk:deploy-all                                # deploy all stacks (Floci)
task cdk:deploy-ecr
task cdk:deploy-cognito
task cdk:deploy-domain
task cdk:deploy-network
task cdk:deploy-datastore
task cdk:deploy-security
task cdk:deploy-runtime
```

## Local metrics

Prometheus and Grafana run in the same local Docker stack as Floci, Jaeger, and Postgres (not emulated AWS services).

**IaC for Metrics is a priority on the internal release roadmap.**

```bash
task metrics:restart                               # restart Prometheus + Grafana; regenerate configs and dashboards
task metrics:start
task metrics:stop
```

Next: [Infrastructure](/docs/infrastructure) for AWS topology, datastores, and cost.
