---
title: "Developer Setup Guide"
summary: "A comprehensive guide for setting up and working with the Backbone platform."
---

A comprehensive guide for setting up and working with the Backbone platform.

For GitHub and AWS setup, CDK against Floci (local) or AWS (upstream environments), and local Prometheus and Grafana, see [OPERATIONS.md](/docs/operations).

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

### Quick start reference (minus the commentary)

```bash
# bootstrapping
brew install go-task
task bootstrap:toolchain
task bootstrap:mvn
task lefthook:install
task bootstrap:licence-install
task bootstrap:licence-secure
task bootstrap:platform-config
task bootstrap:dotenvrc

# infra
task docker:start -- floci jaeger postgres redis [prometheus grafana]
task dev:floci
task seed:floci

# dev servers
task build:install
mert start

# tests
task test:unit
task seed:it:up
task test:integration
```

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
- **Quarkus SmallRye Config**: A secret for property encryption; generate using e.g. `openssl rand -base64 32`

**References**:
- OSS Index API key: <https://ossindex.sonatype.org/doc/auth-required>
- NVD API key: <https://nvd.nist.gov/developers/request-an-api-key>
- LinkedIn Authentication [optional]: <https://learn.microsoft.com/en-gb/linkedin/shared/authentication/authentication>

### 7. Floci Docker Services

Start required Docker services for local development. Floci emulates AWS on `:4566` (S3, DynamoDB, SES, SSM, Secrets Manager, Cognito, and more).

#### Service management

Services can be stopped, started, and restarted individually or all at once:

```bash
task docker:restart -- floci jaeger postgres
task docker:start -- floci jaeger postgres
task docker:stop -- floci jaeger postgres
```

Service status can be queried with:

```bash
task docker:status
```

Example output:

```terminaloutput
task: [docker:status] scripts/docker/status.sh floci jaeger postgres redis prometheus grafana

Docker Container Status

SERVICE      | STATE      | PORTS

floci        | running    | 4566:4566
jaeger       | running    | 16686:16686,4317:4317
postgres     | running    | 5432:5432
redis        | running    | 6380:6380
prometheus   | running    | 9090:9090
grafana      | running    | 3000:3000
```

#### Provision Floci resources

Create Floci AWS resources (S3, DynamoDB, Cognito pools → SSM/Secrets) and seed development data:

```bash
task dev:floci
task seed:floci
```

### 8. Integration test fixtures

Seed IT fixtures into Floci Cognito and sync actor identities into Postgres `actor.actors` (Cognito holds credentials; Postgres holds actor metadata):

```bash
task seed:it:up
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
task build:clean build:nuke build:install
```

## Useful Debugging Commands

### Floci

1. notification-service:      ➜ `awslocal ses verify-email-identity --email hello@backbonehq.io` · inspect mail: `curl http://localhost:4566/_aws/ses`
2. actor-service:             ➜ `docker exec -it postgres psql -U postgres -d backbone -c "SELECT * FROM actor.actors;"`
3. document-service:          ➜ `awslocal dynamodb describe-table --table-name DOCUMENTS 2>&1 | grep -A 10 "KeySchema"`
4. authenticate Docker to ECR ➜

```bash
aws ecr get-login-password --region "$AWS_REGION" \
| docker login --username AWS --password-stdin "$(echo $ECR_URI | cut -d/ -f1)"
```
