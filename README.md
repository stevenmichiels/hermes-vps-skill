# hermes-vps

Human-facing quick start for the `hermes-vps` skill used by Codex or
another coding agent.

This skill provisions and operates a Hermes Agent VPS on Hetzner using
Terraform and Ansible. It keeps SSH and service ports private by default,
deploys Hermes Agent in Docker Compose, optionally deploys a private Firecrawl
stack, creates a separated agent-workbench directory layout, and includes
operational scripts for status checks, backups, release checks, health alerts,
and Docker cleanup.

For agent instructions, use `SKILL.md`. This README is for humans deciding
whether to use, review, or publish the skill.

This repository is a sanitized deployment template, not a copy of a live
production VPS configuration.

## Before You Fork or Use This

- Read `SKILL.md` before running Terraform or Ansible.
- Review every Terraform plan before applying it.
- Keep local inventory, Terraform variables, state, plans, logs, SSH keys,
  OAuth files, bot tokens, and runtime env files out of Git.
- Set `bootstrap_public_ssh_cidr` to your current public `/32` before first
  public SSH bootstrap. The template default is empty so Ansible fails before
  UFW if you forget.
- Confirm the SSH exposure model matches your current access path before
  enabling UFW or changing Hetzner firewall rules.
- Treat remote install scripts and floating optional service images as
  supply-chain decisions to review, not invisible defaults.

## Design Goals

- Private by default: narrow bootstrap SSH, then Tailscale-only SSH.
- Hardened base setup with firewalling and basic intrusion protection.
- Reviewable infrastructure through Terraform and Ansible.
- Hermes gateway ports bound to loopback unless public exposure is explicitly accepted.
- Optional private Firecrawl stack attached to the Hermes Docker network by alias.
- Separated VPS zones for live app runtimes, staging, operator repos, agent
  workspaces, service installs, and backups.
- Operational checks for status, health, backups, releases, Docker cleanup, and timers.
- Recoverable VPS model with backups and a documented restore path.

## Architecture Overview

```text
Controller machine
|-- Terraform -> Hetzner server + firewall
|-- Ansible   -> VPS hardening + Docker + Hermes + timers
`-- Local ignored config
    |-- templates/infra/terraform.tfvars
    |-- templates/ansible/inventory.ini
    `-- templates/ansible/vars/local.yml

Hetzner VPS
|-- Tailscale-only SSH for operators
|-- Hermes Agent container
|-- /var/lib/hermes persistent Hermes state
|-- optional private Firecrawl stack
|-- /home/<admin_user>/repos and agent-workspaces
`-- /var/backups/hermes-vps backups
```

The controller keeps deployment intent and private local config. The VPS keeps
runtime state, loopback-bound services, timers, and backups. Terraform owns the
server/firewall shape; Ansible owns host configuration and service layout.

## Why This Exists

Running an always-on messaging assistant is not just "start a container". The
useful part is the operational wrapper around it: safe access, repeatable
rebuilds, private secrets, health checks, backups, and a clear recovery path
when the VPS disappears or the runtime changes.

This skill turns that into a reusable workflow. It gives an agent enough
structure to provision the box, harden it, deploy Hermes Agent, keep Firecrawl
private when enabled, and verify that the result is actually usable.

## Why This Matters for AI Agents

AI agents become more useful when they operate inside a stable, recoverable
environment with explicit boundaries. This skill gives the agent a safe
operating envelope: private access, reproducible infrastructure, documented
secret handling, health checks, backups, and recovery steps.

The goal is not to let an agent mutate a production box freely. The goal is to
give agents a reliable workbench where infrastructure changes are explicit,
reviewable, and reversible.

## Added Value

- Repeatable infrastructure instead of click-by-click server setup.
- SSH guardrails: narrow bootstrap access first, then Tailscale-only SSH.
- Private-by-default Hermes gateway/dashboard and Firecrawl API binds.
- Secret-safe runtime setup with placeholder env templates and explicit "do not commit" rules.
- One-command operator checks through `hermes-vps status`.
- Scheduled backups, Docker cleanup, stable release checks, and health transition alerts.
- Restore runbook for a fresh VPS rebuild from backup.
- Source-controlled Hermes runtime skill templates, including a private Firecrawl workflow skill.
- Managed parent directories for co-hosting Claude Code CLI, Codex CLI, OpenClaw,
  Hermes, repos, app runtimes, and backups without mixing them into one
  root-owned workspace.
- Local config examples so the public skill can stay generic while private
  machine/account values stay ignored.

## What This Is Not

This is not a hosted service, a turnkey SaaS product, or a substitute for
reading Terraform/Ansible plans before applying them. It is an opinionated
deployment skill with guardrails, intended for operators who are comfortable
reviewing infrastructure changes.

## Threat Model and Assumptions

This template assumes:

- A single-operator or small trusted-admin VPS, not a multi-tenant host.
- A trusted controller machine for Terraform and Ansible runs.
- A trusted Tailscale tailnet when Tailscale SSH is used for ongoing access.
- Loopback-bound Hermes and Firecrawl service ports unless public exposure is explicitly accepted.
- Secret-bearing runtime files remain on the controller or VPS and are never committed.
- Backups are private artifacts and should be encrypted when copied off the VPS.

This template does not try to protect against a compromised controller machine,
malicious Terraform/Ansible operator, compromised upstream image, hostile
tailnet member with admin privileges, or intentional public exposure of private
services.

## Supply Chain Notes

- The Hermes Agent Docker image is pinned to an explicit release tag by
  default. Do not switch it to `latest` unless you intentionally accept
  floating runtime behavior.
- Optional Firecrawl image variables default to upstream `latest` tags because
  Firecrawl's self-host stack can move across multiple coordinated images. Pin
  them in `templates/ansible/vars/local.yml` before production use if you need
  reproducible rollouts.
- The base role uses upstream remote install scripts for Tailscale and Claude
  CLI. Optional Codex CLI support installs Node/npm packages from the configured
  OS/npm registries plus the distro `bubblewrap` package for Linux sandboxing.
  Review those tasks before use, pin or replace them if your environment
  requires stricter supply-chain controls, and rerun syntax checks after
  changes.
- Terraform provider versions are locked in `templates/infra/.terraform.lock.hcl`;
  keep lockfile changes reviewable.

## What It Includes

- Terraform templates for Hetzner server and firewall setup.
- Ansible roles for base hardening, UFW/fail2ban, Docker, Hermes Agent,
  optional Firecrawl, backups, health checks, release checks, and timers.
- Source-controlled Hermes runtime skill templates under `templates/hermes-skills/`.
- A restore runbook in `references/restore.md`.
- Release notes in `CHANGELOG.md`.
- Example config files for local deployment values.

## VPS Layout

The playbook manages parent directories for a single VPS that can act as an
agent workbench without mixing live apps, staging, repos, and service runtimes
into one shared root-owned directory.

Default managed parent zones:

```text
/srv/apps
/home/<admin_user>/repos
/home/<admin_user>/agent-workspaces
/opt/openclaw
/opt/hermes
/opt/firecrawl
/etc/firecrawl
/var/backups
```

Suggested concrete layout once real app and repo names are known:

```text
/srv/apps/production-site
/srv/apps/staging-site
/home/<admin_user>/repos/project
/home/<admin_user>/agent-workspaces/claude-code
/home/<admin_user>/agent-workspaces/codex-cli
/opt/openclaw
/opt/hermes
/opt/firecrawl
/var/backups
```

Firecrawl is an optional private service zone, not a subdirectory of Hermes.
The Firecrawl API is attached to the Hermes Docker network only so Hermes can
call it privately through the `firecrawl` alias; its compose file and env live
outside Hermes under `/opt/firecrawl` and `/etc/firecrawl`.

Codex CLI can also use the private Firecrawl service from the VPS host through
`http://127.0.0.1:3002`. The preferred integration is a user-local Codex MCP
entry:

```sh
codex mcp add firecrawl --env FIRECRAWL_API_URL=http://127.0.0.1:3002 -- npx -y firecrawl-mcp
```

This writes to the operator's `~/.codex/config.toml`; treat it as local account
state, not a tracked deployment secret. Validate with `codex mcp list` and an
interactive Codex prompt that uses the `firecrawl` MCP server. Keep Firecrawl
loopback-bound unless public exposure is explicitly reviewed and accepted.

Claude Code CLI and Codex CLI are treated as interactive coding tools for Git
repos or disposable workspaces. They should not write directly to production
runtime directories, production env files, or production databases. Use Git,
reviewed diffs, CI/deploy scripts, Ansible, or a staging promotion step for
live changes.

## Prerequisites

You need:

- Terraform
- Ansible
- A Hetzner Cloud account and API token
- SSH access from your local machine for bootstrap
- Optional: Tailscale, strongly recommended for private ongoing SSH access
- Optional: Telegram or another Hermes-supported gateway integration
- Optional: Firecrawl, if you enable private scrape/crawl/PDF extraction workflows

## First-Time Setup

Copy the example files and fill in your local values:

```sh
cp templates/ansible/inventory.ini.example templates/ansible/inventory.ini
cp templates/ansible/vars/local.yml.example templates/ansible/vars/local.yml
cp templates/infra/terraform.tfvars.example templates/infra/terraform.tfvars
```

Example values are intentionally generic. Replace them with your own values locally and keep the resulting files private.

Keep real secrets and private deployment details out of committed files.
Provider tokens and bot/API keys belong in secure shell/env storage or on the
VPS in `/var/lib/hermes/.env` and `/etc/firecrawl/firecrawl.env`. Do not include
runtime env files in unencrypted backups, logs, or support bundles.

## Safe Review Path

You can review the template without creating or changing cloud resources:

```sh
terraform -chdir=templates/infra fmt -check -diff
terraform -chdir=templates/infra init -backend=false -input=false -lockfile=readonly
terraform -chdir=templates/infra validate

cd templates/ansible
ansible-playbook -i inventory.ini.example site.yml --syntax-check
```

This checks formatting, Terraform schema validity, and Ansible syntax without
contacting Hetzner or a live VPS. A real deployment still requires private local
config files, credentials, an inspected Terraform plan, and an explicit apply.

## Typical Flow

1. Copy the example config files.
2. Add local deployment values.
3. Run `terraform plan` locally and review it.
4. Run `terraform apply` locally only if the plan matches your intent.
5. Update the Ansible inventory with the bootstrap VPS IP first, then the Tailscale hostname/IP after private access is enabled.
6. Run the Ansible playbook locally against the VPS.
7. Configure Hermes runtime secrets on the VPS.
8. Enable the Hermes service through Ansible.
9. Verify the deployment on the VPS with `hermes-vps status`.
10. Verify timers for backups, Docker cleanup, release checks, and health checks.
11. Optional: enable Firecrawl and validate the private Hermes-network alias with `hermes-vps status`.

## Recovery Model

The VPS is treated as replaceable infrastructure. Persistent state should live
in backups, Git remotes, or explicitly documented storage paths.

If the VPS is lost, the expected recovery path is:

1. Recreate infrastructure with Terraform.
2. Re-run Ansible.
3. Restore Hermes state from backup.
4. Re-check health, timers, gateway access, and optional Firecrawl access.

## Local Files You Must Not Share

Do not commit or share:

- `templates/ansible/inventory.ini`
- `templates/ansible/vars/local.yml`
- `templates/infra/terraform.tfvars`
- `templates/infra/terraform.tfstate*`
- `templates/infra/tfplan*`
- `templates/infra/.terraform/`
- `templates/ansible/ansible-run.log`
- backup archives, OAuth profiles, pairing state, runtime env files, or SSH keys

These should be ignored by `.gitignore` in this repo. Verify before committing.

## Basic Commands

Terraform commands run locally from this repository. Always inspect the Terraform plan before applying it:

```sh
terraform -chdir=templates/infra init
terraform -chdir=templates/infra plan
terraform -chdir=templates/infra apply
```

Ansible commands run locally and target the VPS:

```sh
cd templates/ansible
ansible-playbook -i inventory.ini site.yml
```

The `hermes-vps ...` operator commands run on the VPS after deployment:

```sh
hermes-vps status
hermes-vps timers
hermes-vps release-check
hermes-vps healthcheck
hermes-vps backup
hermes-vps docker-cleanup
```

## Termius Access

Do not open public SSH just for Termius. Use normal SSH over Tailscale:

- Install and connect Tailscale on the Termius device.
- Add a Termius host with the VPS Tailscale IP or MagicDNS name.
- Use port `22`.
- Use the non-root admin user configured in `templates/ansible/vars/local.yml`.
- Authenticate with the private key matching `admin_authorized_keys`.

For a phone or tablet, prefer a device-specific key:

1. Create a new Ed25519 key inside Termius.
2. Copy only the public key.
3. Add that public key as a separate `admin_authorized_keys` entry in ignored
   `templates/ansible/vars/local.yml`.
4. Rerun Ansible from the controller.
5. Connect Termius to the VPS Tailscale IP/hostname as the non-root admin user
   on port `22`.

Password login stays disabled. Do not copy VPS secrets, runtime env files,
Terraform state, or backups into Termius.

## Sharing Checklist

Before sharing this skill:

```sh
git status --short --ignored .
git grep -n -I -E 'BEGIN .*PRIVATE KEY|OPENAI_API_KEY=.+|ANTHROPIC_API_KEY=.+|GITHUB_TOKEN=.+|GH_TOKEN=.+|TELEGRAM_BOT_TOKEN=.+|SLACK_.*TOKEN=.+|API_SERVER_KEY=.+|FIRECRAWL.*KEY=.+|HCLOUD_TOKEN=.+' -- . || true
git ls-files . | rg '(^|/)(inventory\.ini|terraform\.tfstate|terraform\.tfvars|tfplan|ansible-run\.log|local\.yml)$|\.terraform/' || true
```

The first command should show private deployment files only as ignored. The last two commands should not print real secrets or local deployment files. Add your own usernames, hostnames, account labels, and local path fragments to the grep before publishing publicly.
