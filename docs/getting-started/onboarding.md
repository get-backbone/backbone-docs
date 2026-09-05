---
title: "Onboarding"
summary: "A comprehensive guide to setting up the Backbone platform."
---

First-time setup: get a working copy in your GitHub org, install your licence, and run the bootstrap tasks once.

## Table of Contents

- [Create a new repository from Backbone template](#create-a-new-repository-from-backbone-template)
- [Upstream Backbone repository updates](#upstream-backbone-repository-updates)
- [Local workstation setup](#local-workstation-setup)

---

## Create a new repository from Backbone template

Visit [`backbone-platform`](https://github.com/get-backbone/backbone-platform) and click on **Use this template**.

That will create a new private repository in your org. The Backbone vendor has no access to your code or repository (not even read) unless you invite them.

Your organization is added as a Contributor on the Backbone vendor `get-backbone/backbone-platform` so you can `read` the vendor tree and pull updates.

1. Open `get-backbone/backbone-platform`.
2. Click **Use this template** → **Create a new repository**.
3. Create a **private** repository in your org. There is only a single `main` branch, so you can leave **Include all branches** unchecked.

Alternatively, clone `backbone-platform` and push to a new empty private repository. Do not click Fork.

You need a signed licence file (delivered out of band) before the licence bootstrap tasks will succeed. Purchase is on the [Backbone website](https://backbonehq.io/#pricing). Install it after you have a working copy; the tasks live in the repository.

### 1. Remotes

On your machine:

```bash
git remote add origin   git@github.com:<your-org>/<your-platform>.git
git remote add upstream git@github.com:get-backbone/backbone-platform.git
git fetch origin
git fetch upstream
```

`origin` is your repository. `upstream` is the Backbone vendor tree you pull updates from.

---

## Upstream Backbone repository updates

After `git fetch upstream`:

| Intent                                   | Typical command                               |
|------------------------------------------|-----------------------------------------------|
| Take everything new from the vendor tree | `git merge upstream/main` (or rebase onto it) |
| Take one commit                          | `git cherry-pick <sha>`                       |
| Take a specific release                  | `git merge <vX.Y.Z>`                          |

The first merge after **Use this template** needs either a merge of unrelated histories or a squash merge:

```bash
git fetch upstream
git merge --allow-unrelated-histories upstream/main
```

or

```bash
git fetch upstream
git merge --squash --allow-unrelated-histories upstream/main
git commit -m "Take Backbone <vX.Y.Z>"
```

This is because GitHub templates start with a single new commit, so your history is not the same as the vendor’s until that merge.

If you extended a service (for example MFA on `auth-service`), git will naturally conflict in files you changed. Resolve those on **your** repository. Paths you never touched can fast-forward.

---

## Local workstation setup

### Quick start

To get started, minus the explanatory commentary in further reading below, issue the following commands:

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

### 1. Required tooling

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

To allow consumption of GitHub packages published by the public `get-backbone/backbone-kit` repo, configure local `~/.m2/settings.xml` with GitHub credentials. The script prompts for your GitHub username and a classic (only) PAT token named `PAT_BACKBONE_DEPLOY` that you will need to create. The script will back up any existing `~/.m2/settings.xml` file first.

```bash
task bootstrap:mvn
```

See [Authenticating with a personal access token][github-pat-auth].

[github-pat-auth]: https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-apache-maven-registry#authenticating-with-a-personal-access-token

### 3. Git hooks

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

Add each user who will run the app to the backbone group:

```bash
# macOS
sudo dscl . -append /Groups/backbone GroupMembership $(whoami)

# Linux
sudo usermod -aG backbone $(whoami)
```

Then have them **log out and back in** (a new terminal is not enough on macOS).
You can verify your installed licence file with:

```bash
task bootstrap:licence-verify
```

After that, running the app as that user is enough; the process can read the licence via group membership.

### 5. Client platform infrastructure configuration

Before deploying any CDK infrastructure, configure Backbone platform defaults for your organisation and licence tier. The wizard only prompts for options allowed on that tier (Foundation never offers BYOK CMKs, HA endpoints, etc.). Allowed values per tier are in [Platform features](/docs/features#licence-tiers-and-platform-configuration).

To create or update platform configuration run:

```bash
task bootstrap:platform-config
```

You can override values on a per-environment basis. By convention, backbone treats DEV/INT as non-production environments and STAGE/PROD as production-like environments.

When you are ready to deploy, continue with [Operations](/docs/operations).

### 6. Development environment variables

Secrets and local overrides live in `.envrc.local` (gitignored). The committed `.envrc` sources `.envrc.local` and watches it for changes so direnv reloads automatically.

Node.js for CDK is scoped to `infra/.envrc` (see `.nvmrc`).

Create an initial set of environment variables for API keys and secrets:

```bash
task bootstrap:dotenvrc
```

This script sets up, and you will be prompted for:
- **API Keys**: NVD (local OWASP); Sonatype Guide / OSS Index is CI-only
- **Google OAuth** [optional]: only when `googleOauthEnabled` is true in platform-config (requires `natEnabled: true`)
- **LinkedIn OAuth** [optional]: only when `linkedInOauthEnabled` is true in platform-config (requires `natEnabled: true`)

**References**:
- NVD API key: <https://nvd.nist.gov/developers/request-an-api-key>
- Google Cloud credentials: <https://console.cloud.google.com/apis/credentials>
- LinkedIn Authentication: <https://learn.microsoft.com/en-gb/linkedin/shared/authentication/authentication>

Next: [Development](/docs/development) to run Docker, Floci, and Quarkus locally.
