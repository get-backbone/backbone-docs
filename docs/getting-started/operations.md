---
title: "Operations"
summary: "Operational tasks that sit outside day-to-day service development."
---

AWS and GitHub CI bootstrap, then day-2 infra.

For first-time developer workstation setup see [Onboarding](/docs/onboarding).  
To run Quarkus locally, see [Development](/docs/development).  
For a comprehensive task index, see the [Cheatsheet](/docs/cheatsheet).  
[Runbooks](/docs/runbook) include infrastructure hibernation (FinOps), ECS native vs JVM image configuration, and how to configure self-hosted GHA runners.

## Table of contents

- [First-time: AWS and GitHub CI](#first-time-aws-and-github-ci)
  - [1. AWS credentials](#1-aws-credentials)
  - [2. CDK toolkit bootstrap](#2-cdk-toolkit-bootstrap)
  - [3. GitHub Actions OIDC (workstation, once per account)](#3-github-actions-oidc-workstation-once-per-account)
  - [4. GitHub Actions variables and secrets](#4-github-actions-variables-and-secrets)
  - [5. Platform stacks (`01-infra-bootstrap.yml`)](#5-platform-stacks-01-infra-bootstrapyml)
  - [6. Domain DNS delegation](#6-domain-dns-delegation)
  - [7. Remaining stacks](#7-remaining-stacks)
- [Day-2: AWS](#day-2-aws)
  - [CDK lifecycle](#cdk-lifecycle)
  - [Observability](#observability)
- [GitHub Actions reference](#github-actions-reference)
- [Local CDK development with Floci](#local-cdk-development-with-floci)
- [Local metrics](#local-metrics)

---

## First-time: AWS and GitHub CI

INT, STAGE, and PROD are separate AWS accounts. Backbone deploys in a single region per environment (multi-AZ inside that region). Backbone does not install or replace a landing zone; map the accounts into Superwerker, Control Tower, or whatever you already run.

### 1. AWS credentials

In `~/.aws/credentials`, add a profile named `backbone-sandbox` for the account you will use from the workstation (usually INT):

```bash
[backbone-sandbox]
aws_access_key_id=********************
aws_secret_access_key=********************
region=us-west-2
```

For STAGE or PROD, authenticate to that account (SSO, IAM user, or a separate profile) before the workstation commands in step 3.

### 2. CDK toolkit bootstrap

Once per account, in both the workload region defined in `backbone-sandbox` and also in `us-east-1` (CloudFront / WAF):

```bash
task aws:bootstrap
```

Uses the `backbone-sandbox` profile. CDK cannot deploy `GitHubRoleStack` or anything else until `CDKToolkit` exists.

### 3. GitHub Actions OIDC (workstation, once per account)

Each environment account needs `GitHubRoleStack` before GitHub Actions can assume `backbone-github-actions-ci`. There is no CDK pattern that avoids this: something already in the account must authenticate. From a machine logged into that account:

```bash
BACKBONE_STAGE_ENV=INT task cdk:synth
BACKBONE_STAGE_ENV=INT task aws:deploy-github-role
# Repeat with BACKBONE_STAGE_ENV=STAGE and PROD against those AWS accounts
```

OIDC trust is scoped to `github.organization` and `github.repo` in `platform-config.yml`.

The INT GitHub role includes ECR push to the central registry. STAGE/PROD roles can force-deploy ECS and run CDK in their accounts but do not allow image push.

### 4. GitHub Actions variables and secrets

Set the repository variables and secrets listed under [GitHub Actions reference](#github-actions-reference). Workflow `01-infra-bootstrap.yml` cannot assume a role or synth without them.

Hosted GitHub Actions runners have no `/etc/backbone-config`. Publish the same file as repository secret `BACKBONE_LICENCE` (requires `gh auth login`):

```bash
task bootstrap:licence-github
```

### 5. Platform stacks (`01-infra-bootstrap.yml`)

Actions → **01 Infra bootstrap** → Run workflow → choose INT for the first pass, then STAGE, or PROD as required.

| Stage        | Deploys                                      |
|--------------|----------------------------------------------|
| INT          | `EcrStack`, `DomainStack`, `GitHubRoleStack` |
| STAGE / PROD | `DomainStack`, `GitHubRoleStack`             |

`DomainStack` is in this workflow because Route 53 / ACM requires a one-off, mid-flow manual intervention and DNS change at your domain provider. Until that delegation exists, the ACM certificate stays `PENDING_VALIDATION` and CloudFormation does not finish.

Whilst `DomainStack` is waiting on ACM, complete [step 6](#6-domain-dns-delegation), then the job can proceed.

### 6. Domain DNS delegation

`DomainStack` provisions a Route 53 public hosted zone for the environment subdomain (`<env>.<domainRoot>`) plus an ACM certificate validated by DNS. ACM writes validation records into the new subdomain hosted zone. Those records are only publicly resolvable once the root domain (`domainRoot` in `config/src/main/resources/platform-config.yml`) delegates the subdomain.

```bash
aws route53 list-hosted-zones-by-name --dns-name "<env>.<domainRoot>" \
  --query 'HostedZones[0].Id' --output text \
| xargs -I {} aws route53 get-hosted-zone --id {} \
  --query 'DelegationSet.NameServers' --output text
```

At the root domain's DNS provider, create an `NS` record named for the subdomain label (e.g. `int`) whose value is those four name servers. When the parent domain is itself a Route 53 hosted zone, add the record there. After delegation propagates, ACM completes validation and the deploy proceeds.

When `DomainStack` succeeds, you should see an email notification `DKIM setup SUCCESS for <env>.<domainRoot>` for that region.

- Route 53 subdomain delegation: <https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/CreatingNewSubdomain.html>
- ACM DNS validation: <https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html>

### 7. Remaining stacks

After OIDC, Domain, and (INT) ECR exist, run [`10-infra-deploy.yml`](https://github.com/get-backbone/backbone-platform/blob/main/.github/workflows/10-infra-deploy.yml) (`workflow_dispatch`, or a push to `infra/**` on `main` for INT). That workflow synths and executes `task aws:deploy-all`.

---

## Day-2: AWS

### CDK lifecycle

```bash
task cdk:build
task cdk:test
task cdk:synth
task cdk:diff
```

The client mirror does not include `infra/cdk.context.json`. Run `task cdk:synth` once with AWS credentials in the target account so CDK can cache account-specific lookups (availability zones, managed prefix lists) and commit your `cdk.context.json` file.

Local `task aws:*` deploys are for workstation debugging. INT / STAGE / PROD after first-time go through GitHub Actions (`10-infra-deploy.yml`, hibernate via [Runbook](/docs/runbook#3-non-prod-infrastructure-hibernate)).

### Observability

Skip when managed observability is disabled for that environment. When enabled, CDK deploys `ObservabilityStack` (AMP + AMG) as part of `deploy-all`. Identity Center is only required for Grafana sign-in, not for CDK.

#### Identity Center

1. Enable **IAM Identity Center** (built-in directory is fine for sandbox).
2. Create a user or group.
3. Create a permission set (e.g. `AdministratorAccess`).
4. Assign user/group to the workload account; wait until the assignment completes.
5. Note the **access portal** URL (Identity Center → Dashboard).

AMG needs an **organization-level** Identity Center instance. Account SSO assignment grants AWS Console access only, not Grafana workspace access.

#### X-Ray OTLP bootstrap

Once per account and region so apps can export OTLP traces (otherwise export returns HTTP 400):

```bash
task aws:bootstrap-xray
```

Idempotent when destination is already `CloudWatchLogs` / `ACTIVE`. Operator IAM needs `logs:PutResourcePolicy` and `xray:UpdateTraceSegmentDestination`.

#### Workspace access

After `ObservabilityStack` exists, grant AMG workspace roles. Empty `list-permissions` causes redirect loops. CDK does not assign Grafana users. Re-run after every ObservabilityStack recreate (hibernate/redeploy creates a new workspace ID; permissions do not carry over).

```bash
task aws:grant-observability -- you@example.com
```

Optional: `BACKBONE_STAGE_ENV` (default `INT`), `AMG_ROLE` (`ADMIN` | `EDITOR` | `VIEWER`). List workspaces and permissions: `./scripts/aws/amg-grant-workspace-access.sh list`.

Open the workspace URL (`Backbone-OBSERVABILITY-AMG-WORKSPACE-URL`) → **Sign in with AWS IAM Identity Center** (Identity Center password, not `backbone-sandbox` access keys).

After first deploy, complete the AMG datasource and X-Ray plugin checklist in [Runbook §8](/docs/runbook#8-amg-post-deploy-checklist).

---

## GitHub Actions reference

Pipelines run on free-tier GitHub plans. Most workflows finish in a few minutes; all currently finish in under 10 minutes.

| Workflow                                                                                                                         | When                                                    | What                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------|--------------------------------------------------------------------------|
| [`01-infra-bootstrap.yml`](https://github.com/get-backbone/backbone-platform/blob/main/.github/workflows/01-infra-bootstrap.yml) | `workflow_dispatch`                                     | After OIDC exists: Domain (DNS mid-flow), GitHubRole updates, ECR on INT |
| [`10-infra-deploy.yml`](https://github.com/get-backbone/backbone-platform/blob/main/.github/workflows/10-infra-deploy.yml)       | `workflow_dispatch`, or push `infra/**` on `main` (INT) | `cdk synth` + `deploy-all`                                               |

Workflows pick the target account from the variables below (e.g. INT for ECR push; INT/STAGE/PROD from a `stage_env` choice).

**Repository variables** (Settings → Secrets and variables → Actions → Variables):

- `AWS_REGION` - region for INT / STAGE / PROD (same region for all promotion stages)
- `INT_AWS_ACCOUNT_ID` - AWS account ID for the INT stage (central ECR registry; image push and INT deploys)
- `STAGE_AWS_ACCOUNT_ID` - AWS account ID for the STAGE stage
- `PROD_AWS_ACCOUNT_ID` - AWS account ID for the PROD stage
- `BACKBONE_ECS_RUNTIME_MODE` (optional) - `jvm` (faster image build; slower container startup) or `native` (long builds; smaller images; better cold start)
- `USER_BACKBONE_DEPLOY` - GitHub username for the account that owns `secrets.PAT_BACKBONE_DEPLOY` (workflows pass it as `GITHUB_MAVEN_USERNAME` to `configure-maven-github.sh`)

**Repository secrets** (Settings → Secrets and variables → Actions → Secrets):

Backbone-related:

- `PAT_BACKBONE_DEPLOY` - GitHub token for deployment
- `SMALLRYE_CONFIG_SECRET_KEY` (base64 encoded) - Encryption key for secrets.properties
- `GPG_PRIVATE_KEY` - GPG private key for signing artifacts during releases; see [03-release-bump.yml](https://github.com/get-backbone/backbone-platform/blob/main/.github/workflows/03-release-bump.yml)
- `GPG_PASSPHRASE` - GPG passphrase for signing artifacts during releases

External:

- `NVD_API_KEY` - NVD API key for vulnerability scanning; see [NIST - Request an API Key](https://nvd.nist.gov/developers/request-an-api-key)
- `OSS_INDEX_API_KEY` - Sonatype Guide / OSS Index API token for weekly CI dependency scanning only (local OWASP skips OSS Index by default); see [Sonatype Guide](https://guide.sonatype.com/) and [OSS Index analyzer](https://dependency-check.github.io/DependencyCheck/analyzers/oss-index-analyzer.html)
- `CODECOV_TOKEN` - Codecov token for coverage reporting; see [Codecov](https://codecov.io/)

Login with Google (and OAuth refresh-token storage):

- `GOOGLE_OAUTH2_CLIENT_ID` - Google OAuth 2.0 Web client ID; see [Google Cloud Console credentials](https://console.cloud.google.com/apis/credentials)
- `GOOGLE_OAUTH2_CLIENT_SECRET` - Google OAuth 2.0 Web client secret
- `BACKBONE_OAUTH2_REFRESH_TOKEN_ENCRYPTION_KEY` - Encryption key for OAuth2 refresh token encryption (`openssl rand -base64 32`)

Optional, if you wish to keep frontend 'Login with LinkedIn' and have enabled `linkedInOauthEnabled` in `platform-config.yml`:

- `LINKEDIN_OAUTH2_CLIENT_ID` - LinkedIn OAuth 2.0 client ID; see [LinkedIn - OAUTH 2.0 Overview](https://learn.microsoft.com/en-gb/linkedin/shared/authentication/authentication)
- `LINKEDIN_OAUTH2_CLIENT_SECRET` - LinkedIn OAuth 2.0 client secret

---

## Local CDK development with Floci

`cdklocal` and Floci emulation support free CDK development and testing locally.

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
