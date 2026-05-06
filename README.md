# hermes-vps

Human-facing quick start for the `hermes-vps` skill used by Codex or another coding agent.

This skill provisions and operates a Hermes Agent VPS on Hetzner using Terraform and Ansible. It keeps SSH and service ports private by default, deploys Hermes Agent in Docker Compose, optionally deploys a private Firecrawl stack, and includes operational scripts for status checks, backups, release checks, health alerts, and Docker cleanup.

For agent instructions, use `SKILL.md`. This README is for humans deciding whether to use, review, or publish the skill.

This repository is a sanitized deployment template, not a copy of a live production VPS configuration.

## Before You Fork or Use This

- Read `SKILL.md` before running Terraform or Ansible.
- Review every Terraform plan before applying it.
- Keep local inventory, Terraform variables, state, plans, logs, SSH keys, OAuth files, bot tokens, and runtime env files out of Git.
- Set `bootstrap_public_ssh_cidr` to your current public `/32` before first public SSH bootstrap. The template default is empty so Ansible fails before UFW if you forget.
- Confirm the SSH exposure model matches your current access path before enabling UFW or changing Hetzner firewall rules.
- Treat remote install scripts and floating optional service images as supply-chain decisions to review, not invisible defaults.

## Design Goals

- Private by default: narrow bootstrap SSH, then Tailscale-only SSH.
- Hardened base setup with firewalling and basic intrusion protection.
- Reviewable infrastructure through Terraform and Ansible.
- Hermes gateway ports bound to loopback unless public exposure is explicitly accepted.
- Optional private Firecrawl stack attached to the Hermes Docker network by alias.
- Operational checks for status, health, backups, releases, Docker cleanup, and timers.
- Recoverable VPS model with backups and a documented restore path.

## Why This Exists

Running an always-on messaging assistant is not just "start a container". The useful part is the operational wrapper around it: safe access, repeatable rebuilds, private secrets, health checks, backups, and a clear recovery path when the VPS disappears or the runtime changes.

This skill turns that into a reusable workflow. It gives an agent enough structure to provision the box, harden it, deploy Hermes Agent, keep Firecrawl private when enabled, and verify that the result is actually usable.

## Added Value

- Repeatable infrastructure instead of click-by-click server setup.
- SSH guardrails: narrow bootstrap access first, then Tailscale-only SSH.
- Private-by-default Hermes gateway/dashboard and Firecrawl API binds.
- Secret-safe runtime setup with placeholder env templates and explicit "do not commit" rules.
- One-command operator checks through `hermes-vps status`.
- Scheduled backups, Docker cleanup, stable release checks, and health transition alerts.
- Restore runbook for a fresh VPS rebuild from backup.
- Source-controlled Hermes runtime skill templates, including a private Firecrawl workflow skill.
- Local config examples so the public skill can stay generic while private machine/account values stay ignored.

## What This Is Not

This is not a hosted service, a turnkey SaaS product, or a substitute for reading Terraform/Ansible plans before applying them. It is an opinionated deployment skill with guardrails, intended for operators who are comfortable reviewing infrastructure changes.

## Threat Model and Assumptions

This template assumes:

- A single-operator or small trusted-admin VPS, not a multi-tenant host.
- A trusted controller machine for Terraform and Ansible runs.
- A trusted Tailscale tailnet when Tailscale SSH is used for ongoing access.
- Loopback-bound Hermes and Firecrawl service ports unless public exposure is explicitly accepted.
- Secret-bearing runtime files remain on the controller or VPS and are never committed.
- Backups are private artifacts and should be encrypted when copied off the VPS.

This template does not try to protect against a compromised controller machine, malicious Terraform/Ansible operator, compromised upstream image, hostile tailnet member with admin privileges, or intentional public exposure of private services.

## Supply Chain Notes

- The Hermes Agent Docker image is pinned to an explicit release tag by default. Do not switch it to `latest` unless you intentionally accept floating runtime behavior.
- Optional Firecrawl image variables default to upstream `latest` tags because Firecrawl's self-host stack can move across multiple coordinated images. Pin them in `templates/ansible/vars/local.yml` before production use if you need reproducible rollouts.
- The base role uses upstream remote install scripts for Tailscale and Claude CLI. Review those tasks before use, pin or replace them if your environment requires stricter supply-chain controls, and rerun syntax checks after changes.
- Terraform provider versions are locked in `templates/infra/.terraform.lock.hcl`; keep lockfile changes reviewable.

## What It Includes

- Terraform templates for Hetzner server and firewall setup.
- Ansible roles for base hardening, UFW/fail2ban, Docker, Hermes Agent, optional Firecrawl, backups, health checks, release checks, and timers.
- Source-controlled Hermes runtime skill templates under `templates/hermes-skills/`.
- A restore runbook in `references/restore.md`.
- Example config files for local deployment values.

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

Keep real secrets and private deployment details out of committed files. Provider tokens and bot/API keys belong in secure shell/env storage or on the VPS in `/var/lib/hermes/.env` and `/etc/firecrawl/firecrawl.env`. Do not include runtime env files in unencrypted backups, logs, or support bundles.

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

The VPS is treated as replaceable infrastructure. Persistent state should live in backups, Git remotes, or explicitly documented storage paths.

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

## Sharing Checklist

Before sharing this skill:

```sh
git status --short --ignored .
git grep -n -I -E 'BEGIN .*PRIVATE KEY|OPENAI_API_KEY=.+|ANTHROPIC_API_KEY=.+|GITHUB_TOKEN=.+|GH_TOKEN=.+|TELEGRAM_BOT_TOKEN=.+|SLACK_.*TOKEN=.+|API_SERVER_KEY=.+|FIRECRAWL.*KEY=.+|HCLOUD_TOKEN=.+' -- . || true
git ls-files . | rg '(^|/)(inventory\.ini|terraform\.tfstate|terraform\.tfvars|tfplan|ansible-run\.log|local\.yml)$|\.terraform/' || true
```

The first command should show private deployment files only as ignored. The last two commands should not print real secrets or local deployment files. Add your own usernames, hostnames, account labels, and local path fragments to the grep before publishing publicly.
