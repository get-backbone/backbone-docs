---
title: "Cheatsheet"
summary: "Step-by-step explanations, prerequisites, and follow-on setup are in DEVELOPMENT.md — Environment Setup."
---

`task --list` · `task --list-all`

## Bootstrap dev env

Step-by-step explanations, prerequisites, and follow-on setup are in [DEVELOPMENT.md — Environment Setup](/docs/development#environment-setup).

### Tooling

```bash
brew install go-task                               # backbone exposes all operational surfaces via taskfiles; install this first
task bootstrap:toolchain                           # install cli toolchain
task bootstrap:mvn                                 # configure maven settings to allow access to GitHub Packages
task lefthook:install                              # install git hooks configured in .config/lefthook.yml
```

### Licensing

```bash
task bootstrap:licence-install                     # install the licence file on your machine
task bootstrap:licence-secure                      # secure the licence file and directory
```

### Configuration

```bash
task bootstrap:platform-config                     # configure interactive platform-config.yml (dev region, DNS, ECS, NAT)
task bootstrap:dotenvrc                            # generate .envrc.local from API keys and secrets
```

## Bootstrap GitHub env

Commentary, prerequisites, and follow-on setup are in [OPERATIONS.md — GitHub Setup](/docs/operations#github-setup).

```bash
BACKBONE_STAGE_ENV=INT task cdk:synth
BACKBONE_STAGE_ENV=INT task aws:deploy-github-role    # deploy GitHub OIDC role
```

## Local development

```bash
task docker:restart -- floci jaeger postgres       # restart listed docker containers
task docker:start -- floci jaeger postgres
task docker:stop -- floci jaeger postgres
task docker:status                                 # docker container status and ports for all services

task dev:floci                                     # create Floci development resources
task seed:floci                                    # seed Floci development resources
task seed:it:up                                    # IT fixtures (Floci Cognito + Postgres)

task build:nuke                                    # delete the Maven build cache + stop mvn daemons
task build:clean
task build:compile
task build:package
task build:install

task test:unit
task test:integration
task test:integration MODULE=services/auth-service # run a specific module's integration tests
task test:all                                      # run all unit and integration tests
task test:static                                   # OWASP, PMD, SpotBugs, Checkstyle static code analysis

mert start                                         # run all quarkus dev services locally

task dev:scaffold -- <service-name> [--with-rds]   # scaffold a new quarkus service [optional rds persistence]
task dev:audit                                     # run a single quarkus dev service locally
task dev:auth
task dev:actor
task dev:document
task dev:notification
task dev:bff                                       # actor-bff BFF application
task dev:web                                       # web-actor static/stateless ui

task quarkus:status                                # local quarkus server port contention
task quarkus:kill                                  # kill all local quarkus servers

task metrics:restart                               # restart Prometheus + Grafana; regenerate configs and dashboards
task metrics:start
task metrics:stop

task metrics:prometheus-start
task metrics:prometheus-stop
task metrics:prometheus-config
task metrics:grafana-start
task metrics:grafana-stop
task metrics:grafana-dashboard
```

## Infra development

### Standard CDK lifecycle

```bash
task cdk:build
task cdk:test
task cdk:synth
task cdk:diff
```

### AWS env development

```bash
task cdk:install                                   # npm install in infra/ (CI and real AWS)
task aws:bootstrap                                 # CDKToolkit in workload region (backbone-sandbox profile) + us-east-1
task aws:bootstrap-xray                            # X-Ray OTLP → CloudWatch Logs (once per account/region)

task aws:deploy-all                                # deploy all stacks (AWS)
task aws:deploy-ecr                                # deploy ECR stack (INT)
task aws:deploy-cognito                            # deploy Cognito (INT needs GitHubRole first)
task aws:deploy-domain                             # deploy Domain stack (requires manual DNS updates)

task aws:deploy-observability                      # deploy Observability stack only (AMP + AMG)
task aws:grant-observability -- you@example.com    # grant AMG workspace Admin (after deploy/hibernate)

task aws:hibernate                                 # delete non-prod runtime + foundation tier CFN stacks (cost-optimisation)
```

### Floci local CDK development

CDK development is predominantly done directly in AWS free tier. Floci `cdklocal` is supported for development and testing.

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

task cdk:destroy-all
task cdk:destroy-runtime
```
