---
title: "Developer Setup Guide"
summary: "A comprehensive guide for setting up and working with the Backbone platform."
---

A comprehensive guide for setting up and working with the Backbone platform.

For GitHub and AWS setup, CDK in both LocalStack (development) and AWS (upstream environments) and local Prometheus and Grafana, see [OPERATIONS.md](/docs/operations).

For a compact task index, see [CHEATSHEET.md](/docs/cheatsheet).

For **verified** platform capabilities see [FEATURES.md](/docs/features); for how-to guides see [Welcome - Start here](/docs/welcome#start-here). For the ADR index see [ADRs.md](/docs/adrs).

## Table of Contents

- [Environment Setup](#environment-setup)
- [Prerequisites](#prerequisites)
- [Build & Deploy](#build--deploy)
- [Troubleshooting](#troubleshooting)
- [Useful Debugging Commands](#useful-debugging-commands)
- [Operations (GitHub, IaC, metrics)](/docs/operations)

---

## Environment Setup

### Prerequisites

Backbone shell scripts use `#!/usr/bin/env bash` and require **bash 5 or later**. Check your version:

```bash
bash --version
```

The major version number must be at least 5.

- **macOS:** the system `/bin/bash` is 3.2 and is not sufficient. Install Homebrew bash (`brew install bash`) and ensure it is first on your `PATH` (for example via your `.zshrc`).
- **Linux / Amazon Workspaces:** bash 5.x on Amazon Linux 2023 and Ubuntu 22.04+ meets this requirement.

We do not enforce the version in scripts; upgrading bash is left to each developer so existing shell setups are not disrupted.

For shell script layout and scaffolding, see [Runbook §4 — How to add a new bash script](/docs/runbook#4-how-to-add-a-new-bash-script).

### 1. Required Tooling

Backbone is an opinionated platform. It has mature operational tooling, and all operations are exposed as taskfile tasks. Install **task** first, then the remaining CLI toolchain before continuing.

#### macOS (Homebrew)

```bash
brew install go-task
task bootstrap:toolchain
```

#### Linux (Amazon Linux)

*Added 2026-06-22. This install path has not yet been tested on a live Amazon Linux or Workspaces host.*

On Amazon Linux 2023 or AL2 (Amazon Workspaces), install **task** then run the same command:

```bash
sh -c "$(curl -fsSL https://taskfile.dev/install.sh)" -- -d -b /usr/local/bin
task bootstrap:toolchain
```

Other Linux distributions are not supported; `task bootstrap:toolchain` exits on non-Amazon Linux.

### 2. Maven settings

To allow consumption of GitHub packages published by the public `get-backbone/forge-kit` repo, configure local `~/.m2/settings.xml` with GitHub credentials. The script prompts for your GitHub username and a classic (only) PAT token named `PAT_BACKBONE_DEPLOY` that you will need to create. The script will back up any existing `~/.m2/settings.xml` file first.

```bash
task bootstrap:mvn
```

See [Authenticating with a personal access token][github-pat-auth].

[github-pat-auth]: https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-apache-maven-registry#authenticating-with-a-personal-access-token

### 3. Git Hooks

Install git hooks using lefthook:

```bash
task lefthook:install
```

This installs git hooks configured in `.config/lefthook.yml`.

### 4. Install and secure the licence file

Create the config directory and place the file at the fixed path:

```bash
task bootstrap:licence-install
```

Secure both the client host backbone group/user and the licence file and directory, so only the backbone group (and root) can read it.

```bash
task bootstrap:licence-secure
```

Add each user who will run the app to the backbone group, then have them **log out and back in** (a new terminal is not enough on macOS). See the message printed by the script for the exact commands.

After that, running the app as that user is enough; the process can read the licence via group membership.

### 5. Client platform configuration

To override any default platform configuration values in `config/src/main/resources/platform-config.yml` run:

```bash
task bootstrap:platform-config
```

This script sets up overall:
- **AWS region** for local Quarkus development (only): defaults to `us-west-2`
- **Root domain**: You must set this to your own domain name where you control the DNS.

And baseline values for:
- **NAT Gateway enabled**: Set to false everywhere for cost optimization (backbone uses VPC endpoints internally); you only need this enabled if your platform architecture dictates outbound internet access.
- **ECS desired task count**: Set to 1 for cost optimization.

You can override these values on a per-environment basis. By convention, backbone treats DEV/INT as non-production environments and TEST/PROD as production-like environments.

### 6. Development environment variables (.envrc.local)

Secrets and local overrides live in `.envrc.local` (gitignored). The committed `.envrc` sources `.envrc.local` and watches it for changes so direnv reloads automatically.

Node.js for CDK is scoped to `infra/.envrc` (see `.nvmrc`).

Create an initial set of environment variables for API keys and secrets:

```bash
task bootstrap:dotenvrc
```

This script sets up, and you will be prompted for:
- **API Keys**: NVD, OSS Index, LinkedIn
- **LocalStack**: Auth Token
- **Quarkus SmallRye Config**: A secret for property encryption; generate using e.g. `openssl rand -base64 32`

**References**:
- OSS Index API key: <https://ossindex.sonatype.org/doc/auth-required>
- NVD API key: <https://nvd.nist.gov/developers/request-an-api-key>
- LocalStack Auth Token: <https://docs.localstack.cloud/aws/getting-started/auth-token/>
- LinkedIn Authentication [optional]: <https://learn.microsoft.com/en-gb/linkedin/shared/authentication/authentication>

### 7. AWS Cognito

LocalStack does not fully support AWS Cognito. You will need to provision AWS Cognito resources in your development sandbox AWS account. This enables localhost Quarkus to talk to Cognito while other dependencies are LocalStack/Docker. This is fully supported in the AWS free tier:

```bash
task cdk:synth
task aws:deploy-cognito
```

Update `.envrc.local` with AWS Cognito params from SSM, created by the Cognito stack you just deployed:

```bash
task bootstrap:cognito
```

### 8. Domain DNS delegation

`task aws:deploy-cognito` has a dependency on the `DomainStack`: a Route 53 public hosted zone for the environment subdomain (`<env>.<domainRoot>`) plus an ACM certificate validated by DNS. ACM writes its validation records into the new subdomain hosted zone, but those records are only publicly resolvable once the root domain (`domainRoot` in `config/src/main/resources/platform-config.yml`) delegates the subdomain. Until then the certificate stays in `PENDING_VALIDATION` and the deploy blocks.

Delegate by adding an `NS` record for the subdomain on the root domain, using the four name servers Route 53 assigned to the subdomain hosted zone:

```bash
# List the delegation name servers for the subdomain hosted zone
aws route53 list-hosted-zones-by-name --dns-name "int.<domainRoot>.software" \
  --query 'HostedZones[0].Id' --output text \
| xargs -I {} aws route53 get-hosted-zone --id {} \
  --query 'DelegationSet.NameServers' --output text
```

At the root domain's DNS provider, create an `NS` record named for the subdomain label (e.g. `int`) whose value is those four name servers. When the parent domain is itself a Route 53 hosted zone, add the record there. After delegation propagates, ACM completes validation automatically and the deploy proceeds.

When DomainStack deployment succeeds, you should receive a notification 'DKIM setup SUCCESS for `int.<domainRoot>` in the specified region.'

**References**:
- Route 53 subdomain delegation: <https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/CreatingNewSubdomain.html>
- ACM DNS validation: <https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html>

### 9. LocalStack Docker Services

Start required Docker services for local development:

#### Service management

Services can be stopped, started, and restarted individually or all at once:

```bash
task docker:restart -- localstack jaeger postgres
task docker:start -- localstack jaeger postgres
task docker:stop -- localstack jaeger postgres
```

Service status can be queried with:

```bash
task docker:status
```

Example output:

```terminaloutput
task: [docker:status] scripts/docker/status.sh localstack jaeger postgres redis prometheus grafana

Docker Container Status

SERVICE      | STATE      | PORTS

localstack   | running    | 4566:4566
jaeger       | running    | 16686:16686,4317:4317
postgres     | running    | 5432:5432
redis        | running    | 6380:6380
prometheus   | running    | 9090:9090
grafana      | running    | 3000:3000
```

#### Provision LocalStack resources

Provision LocalStack AWS resources (S3 buckets, DynamoDB tables, etc.), and seed development data:

```bash
task dev:localstack
task seed:localstack
```

### 10. Integration test fixtures

Once AWS Cognito is provisioned, seed IT fixtures into Cognito and sync actor identities into Postgres `actor.actors` (Cognito holds credentials; Postgres holds actor metadata):

```bash
task seed:it:up -- --target local
```

**Prerequisites**: Service `actor-service` needs to have been started to create the `ACTORS` table (via Flyway) in Postgres.

```bash
# run all quarkus dev services
mert start
# run actor-service only
task dev:actor
```

To teardown **perf** data only (ephemeral `testuser*` from k6, `loaduser*` from perf TSV, audit truncate, S3)—**not** alice/bob/eve:

```bash
task seed:perf:down
```

---

## Build & Deploy

### Build Tasks

Taskfile provides convenient wrappers around the standard Maven commands:

```bash
# Display all available tasks
task

task build:nuke                                    # Delete the Maven build cache + stop mvn daemons
task build:clean
task build:compile
task build:package
task build:install

task test:unit
task test:integration
task test:integration MODULE=services/auth-service # Run a specific module's integration tests
task test:all                                      # Run all unit and integration tests
task test:static                                   # OWASP, PMD, SpotBugs, Checkstyle static code analysis
```

### Development Runtime

Run services in Quarkus dev mode:

#### Services/Applications/UIs

```bash
mert start                                         # Run all quarkus dev services locally

task dev:audit                                     # Run a single quarkus dev service locally
task dev:auth
task dev:actor
task dev:document
task dev:notification
task dev:bff                                       # actor-bff BFF application
task dev:web                                       # web-actor static/stateless ui
```

#### Quarkus Servers

Check the status of running Quarkus servers:

```bash
task quarkus:status
```

Kill any running Quarkus server processes:

```bash
task quarkus:kill
```

For the complete task index, see [CHEATSHEET.md](/docs/cheatsheet).

## Troubleshooting

### Basic Checks

1. **Docker services not starting**
   - Check Docker/OrbStack is running
   - Verify ports are not in use: `task docker:status`

2. **Quarkus port conflicts**
   - Check running Quarkus servers: `task quarkus:status`
   - Kill conflicting processes: `task quarkus:kill`

### Maven Build Errors

1. **Build cache**
   - The [Maven build cache extension](https://maven.apache.org/extensions/maven-build-cache-extension/) is implemented to improve build times
   - This can be unreliable and produce maven build errors, however
   - It is generally resolved with the following clean build command, which stops the maven daemon and clears the cache:

```bash
task build:nuke build:clean build:install
```

## Useful Debugging Commands

### LocalStack

1. notification-service:      ➜ `awslocal ses verify-email-identity --email hello@backbonehq.io`
2. actor-service:             ➜ `docker exec -it postgres psql -U postgres -d backbone -c "SELECT * FROM actor.actors;"`
3. document-service:          ➜ `awslocal dynamodb describe-table --table-name DOCUMENTS 2>&1 | grep -A 10 "KeySchema"`
4. authenticate Docker to ECR ➜

```bash
aws ecr get-login-password --region "$AWS_REGION" \
| docker login --username AWS --password-stdin "$(echo $ECR_URI | cut -d/ -f1)"
```
