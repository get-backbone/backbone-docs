---
title: "Runbook"
summary: "Operational procedures for Backbone infrastructure and delivery."
---

## Table of contents

- [1) Native → JVM escape hatch](#1-native--jvm-escape-hatch)
- [2) Self-hosted Mac runner](#2-self-hosted-mac-runner)
- [3) Non-prod infrastructure hibernate](#3-non-prod-infrastructure-hibernate)
- [4) Add a bash script](#4-add-a-bash-script)
- [5) Scaffold a new domain service](#5-scaffold-a-new-domain-service)
- [6) Query private Postgres (CloudShell VPC)](#6-query-private-postgres-cloudshell-vpc)
- [7) Password reset against Floci (local)](#7-password-reset-against-floci-local)

---

## 1) Native → JVM escape hatch

When native image fails (startup, crashes, memory), set GitHub Actions variable `BACKBONE_ECS_RUNTIME_MODE` = `jvm`.

- Same deploy tag `<serviceName>`; ECR also has `<serviceName>-jvm` / `<serviceName>-native`
- Runtime swap only — no infra changes
- Rollback: set `BACKBONE_ECS_RUNTIME_MODE` = `native`

---

## 2) Self-hosted Mac runner

When native builds are slow or time out on GitHub-hosted runners.

1. GitHub → **Settings → Actions → Runners → New self-hosted runner** → **macOS** → run the provided install commands.
2. Labels: `self-hosted`, `macOS`, `X64`, `backbone-ecr`. Keep Docker/OrbStack running; machine awake.
3. Repo variable `BACKBONE_ECR_RUNS_ON` = `["self-hosted","macOS","X64","backbone-ecr"]`.
4. Stop: Ctrl+C on the runner; remove `BACKBONE_ECR_RUNS_ON` to use `ubuntu-latest`.

Uses existing OIDC role `backbone-github-actions-ci`. Native builds still run in Linux Docker.

---

## 3) Non-prod infrastructure hibernate

**Hibernate:** workflow **`11 💤 Infra hibernate`** — deletes `RuntimeStack`, `SecurityStack`, `ObservabilityStack`, `DatastoreStack`, `NetworkStack`. Keeps DNS, static edge, CI, ECR, Cognito.

**Wake:** workflow **`10 🐿️ Infra deploy`**, then app deploys as needed.

Optional schedule: uncomment `schedule` in `.github/workflows/11-infra-hibernate.yml`.

---

## 4) Add a bash script

```bash
task bootstrap:script-template -- quarkus/my-script            # executable
task bootstrap:script-template -- --module aws/cognito/helpers # module
```

- **Executable:** `main()` + `validate_dependencies()`; end with `main "$@"`
- **Module:** functions only; header `Type: module`
- Templates: `scripts/lib/template.sh`, `scripts/lib/template-module.sh`
- Use `require_cli` from `scripts/lib/common.sh`; ShellCheck via lefthook/CI

---

## 5) Scaffold a new domain service

Use the golden `services/template-service` module (in the Maven reactor, not deployed) to mint a new ECS-ready service with platform wiring.

```bash
task dev:scaffold -- quote-engine
task dev:scaffold -- search-service --with-rds
```

Only specify `--with-rds` if your domain service requires Flyway / Hibernate / Postgres persistence.

### Naming rules

| Input            | Module / artifactId | Java package                  | Port env var          | Dev task alias | Internal ALB path |
|------------------|---------------------|-------------------------------|-----------------------|----------------|-------------------|
| `search-service` | `search-service`    | `io.backbone.services.search` | `SEARCH_SERVICE_PORT` | `dev:search`   | `/search/*`       |
| `quote-engine`   | `quote-engine`      | `io.backbone.services.quote`  | `QUOTE_ENGINE_PORT`   | `dev:quote`    | `/quote/*`        |

### Explicitly out of scope (do manually when needed)

- `.mertrc` layout (add `task dev:<alias>` when you want it in `mert start`)
- BFF routes and `libs/domain-clients` REST clients
- Grafana JVM dashboard panels
- CI Floci IT coverage (only when the service adds Floci ITs)
- CDK deploy / first environment rollout

### Local run your domain service

```bash
task dev:quote          # after scaffolding quote-engine
task dev:search         # after scaffolding search-service
```

---

## 6) Query private Postgres (CloudShell VPC)

RDS is private — laptop `psql` cannot reach it. Do not open the RDS SG to your Mac.

Use a [CloudShell VPC environment](https://docs.aws.amazon.com/cloudshell/latest/userguide/creating-vpc-environment.html):

1. CloudShell → **+** → **Create VPC environment** (same region as the stage).
2. **VPC** = `Backbone-VPC-ID`; **Subnet** = private (ECS/RDS); **SG** = `Backbone-ECS-SERVICE-SG-ID` (not an ALB SG).
3. In that shell:

```bash
export PGHOST="$(aws cloudformation list-exports --query "Exports[?Name=='Backbone-DATASTORE-RDS-ENDPOINT'].Value | [0]" --output text)"
export PGPASSWORD="$(aws secretsmanager get-secret-value \
  --secret-id "$(aws cloudformation list-exports --query "Exports[?Name=='Backbone-DATASTORE-RDS-SECRET-ARN'].Value | [0]" --output text)" \
  --query SecretString --output text | jq -r .password)"
psql "host=$PGHOST port=5432 dbname=backbone user=backbone sslmode=require"
```

```sql
SELECT * FROM audit.audit_events WHERE method_name = 'login' ORDER BY event_timestamp DESC LIMIT 10;
```

---

## 7) Password reset against Floci (local)

Floci accepts SES sends but does not deliver mail. After **Forgot password** in web-actor, pull the reset link from the local SES mailbox and open it to continue:

```bash
curl -s http://localhost:4566/_aws/ses
```

Use the `reset-password.html?token=…` URL from the message body (text or HTML). Clear captured mail with `curl -X DELETE http://localhost:4566/_aws/ses` if the list gets noisy.
