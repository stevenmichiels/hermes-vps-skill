# dual-agent-vps

Run agentic coding workflows from a hardened VPS instead of your laptop.

This skill provisions a private Hetzner workbench for Hermes Agent, Claude
Code, Codex CLI, Firecrawl, n8n, MCP tooling, tmux, and SSH/Tailscale-based
remote development.

It is built for people who want their AI coding agents to run close to their
servers, keep working after disconnects, and operate inside a reproducible,
auditable, private-by-default environment.

Under the hood it uses Terraform, Ansible, Docker Compose, system-wide
`uv`/`uvx`, separated workbench directories, health checks, backups, release
checks, Docker cleanup, optional private Firecrawl and n8n deployments, and an
opt-in NoMachine/XFCE remote desktop profile.

The base workbench installs `uv` and `uvx` system-wide by default for fast,
isolated Python tool execution. Poetry is intentionally not a baseline package;
install it only for repos that carry `poetry.lock` or explicitly require
`poetry run`.

For coding agents, use `SKILL.md`.

For humans reviewing or adapting the setup, start here.

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
- Optional private n8n stack bound to loopback or a Tailscale address.
- Optional NoMachine/XFCE remote desktop restricted to the Tailscale access boundary.
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
|-- optional NoMachine/XFCE desktop over Tailscale
|-- Hermes Agent container
|-- /var/lib/hermes persistent Hermes state
|-- optional private Firecrawl stack
|-- optional private n8n stack
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
- Private-by-default Hermes gateway/dashboard, Firecrawl API, and n8n binds.
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
- Loopback-bound Hermes, Firecrawl, and n8n service ports unless public exposure is explicitly accepted.
- Optional remote desktop access through NoMachine over Tailscale, not public GUI ports.
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
- Optional n8n defaults to a pinned Docker image tag and refuses to start until
  `N8N_ENCRYPTION_KEY` is set in `/etc/n8n/.env` on the VPS.
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
  optional NoMachine/XFCE remote desktop, optional Firecrawl, optional n8n,
  backups, health checks, release checks, and timers.
- Source-controlled Hermes runtime skill templates under `templates/hermes-skills/`.
- Source-controlled Codex and Claude skill templates under
  `templates/codex-skills/` and `templates/claude-skills/`.
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
/var/lib/n8n
/etc/n8n
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
/var/lib/n8n
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

n8n is an optional private service zone, not a public webhook endpoint by
default. Its compose file and env live under `/etc/n8n`, persistent data lives
under `/var/lib/n8n`, and access is through an SSH tunnel or an explicit
Tailscale bind until a separate reverse-proxy/TLS design is reviewed.

This writes to the operator's `~/.codex/config.toml`; treat it as local account
state, not a tracked deployment secret. Validate with `codex mcp list` and an
interactive Codex prompt that uses the `firecrawl` MCP server. Keep Firecrawl
loopback-bound unless public exposure is explicitly reviewed and accepted.

For LangChain, LangGraph, and LangSmith documentation lookups from Codex CLI,
add the official hosted docs MCP server as user-local Codex state:

```sh
codex mcp add langchain-docs --url https://docs.langchain.com/mcp
```

If the `codex mcp add` subcommand is not available, add the equivalent
`~/.codex/config.toml` entry:

```toml
[mcp_servers.langchain-docs]
url = "https://docs.langchain.com/mcp"
```

Prefer this official docs MCP for documentation lookups. Do not install
third-party LangChain code-search MCP packages unless their login and credit
model has been explicitly accepted.

## Optional Remote Desktop

The default workbench is headless. A desktop adds packages, memory pressure,
and local session surface, so enable it only when you need an actual GUI.

The supported opt-in profile installs XFCE plus NoMachine. Keep it private by
connecting only over Tailscale:

In ignored `templates/ansible/vars/local.yml`:

```yaml
install_remote_desktop: true
install_nomachine: true
```

Then rerun Ansible after Tailscale SSH is already stable. NoMachine is a
proprietary free-for-personal-use package; keep that supply-chain decision
explicit when enabling this profile.

The playbook installs the official Linux amd64 DEB, verifies the vendor MD5,
disables NoMachine UPnP/NAT-PMP port mapping, writes
`/home/<admin_user>/.nx/config/authorized.crt` from `admin_authorized_keys`,
disables NoMachine automatic firewall changes, removes installer-created public
UFW rules, requires NX private-key authentication, and allows TCP/UDP `4000`
only from the Tailscale CIDR. NoMachine virtual desktops are forced to start
XFCE with `/etc/X11/Xsession startxfce4`.

In the NoMachine client on macOS:

1. Add a new connection.
2. Host: VPS Tailscale IP or MagicDNS hostname.
3. Protocol: `NX`.
4. Port: `4000`.
5. Authentication: key-based/private key.
6. Username: the non-root admin user.
7. Private key: the key matching an `admin_authorized_keys` entry.
8. If NoMachine says it cannot detect a display, choose **Yes** to let it
   create a new virtual display. On a VPS this is expected. You can also enable
   "Always create a new display on this server" for this connection.

Concrete connection values look like this:

```text
Protocol: NX
Host: <vps-tailscale-ip-or-magicdns-name>
Port: 4000
Username: <admin_user>
Authentication: Private key
Private key: ~/.ssh/<key-listed-in-admin_authorized_keys>
```

For the current local deployment, use the ignored `inventory.ini` and
`vars/local.yml` files as the source of truth for `Host`, `Username`, and the
controller SSH key path. Do not copy those private values into the public repo.

Do not sign in to NoMachine Network/cloud for this VPS. Use the direct
Tailscale IP/hostname connection.

Resource guidance:

- Minimum for light desktop use: 2 vCPU / 4 GB RAM.
- Preferred for browsers, IDEs, and agent tools: 4 vCPU / 8 GB RAM or more.
- Avoid public GUI ports. Keep access on Tailscale and validate UFW plus the
  Hetzner firewall after enabling the profile.

Claude Code CLI and Codex CLI are treated as interactive coding tools for Git
repos or disposable workspaces. They should not write directly to production
runtime directories, production env files, or production databases. Use Git,
reviewed diffs, CI/deploy scripts, Ansible, or a staging promotion step for
live changes.

For second opinions, keep one agent as the implementer and the other as a
read-only reviewer. The base role installs the `claude-review` Codex skill into
the operator's `~/.codex/skills` and exposes `codex-claude-review`, which lets
Codex ask Claude Code to review the current `git diff HEAD` plus untracked
files, include recent `.codex/plan/*.md` context, and write a Markdown report
without giving Claude edit tools. This makes review workflows explicit and
auditable instead of relying on an agent that both changes and judges the same
code:

```sh
codex-claude-review "Review the current diff as a strict senior engineer."
codex-claude-review -o claude-review.md "Check whether this is overengineered."
```

The base role also installs the reverse `codex-review` Claude skill into the
operator's `~/.claude/skills` and exposes `claude-codex-review`, which lets
Claude ask Codex CLI for a read-only review through Codex's native
`codex review` command:

```sh
claude-codex-review "Review the current diff as a strict senior engineer."
claude-codex-review --commit HEAD -o codex-review.md
claude-codex-review --base main "Review this branch against main."
```

The helpers never stage or commit. If a review report should be versioned,
inspect it first and commit only the Markdown report separately.

## Prerequisites

You need:

- Terraform
- Ansible
- A Hetzner Cloud account and API token
- SSH access from your local machine for bootstrap
- Optional: Tailscale, strongly recommended for private ongoing SSH access
- Optional: Telegram or another Hermes-supported gateway integration
- Optional: Firecrawl, if you enable private scrape/crawl/PDF extraction workflows
- Optional: n8n, if you enable private workflow automation experiments
- Optional: NoMachine client, if you enable the remote desktop profile

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
VPS in `/var/lib/hermes/.env`, `/etc/firecrawl/firecrawl.env`, and
`/etc/n8n/.env`. Do not include runtime env files in unencrypted backups, logs,
or support bundles.

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
11. Optional: enable NoMachine/XFCE remote desktop only after Tailscale SSH is stable.
12. Optional: enable Firecrawl and validate the private Hermes-network alias with `hermes-vps status`.
13. Optional: enable n8n only after setting `N8N_ENCRYPTION_KEY` on the VPS, then access it through an SSH tunnel or Tailscale bind.

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

Do not open public SSH just for Termius. Use normal OpenSSH over the Tailscale
network:

- Install and connect Tailscale on the Termius device.
- Add a Termius host with the VPS Tailscale IP or MagicDNS name.
- Use port `22`.
- Use the non-root admin user configured in `templates/ansible/vars/local.yml`.
- Authenticate with the private key matching `admin_authorized_keys`.

If Tailscale SSH is enabled on the VPS, it may intercept port `22` before
OpenSSH sees `authorized_keys`, causing Termius to hang on `Authenticating`.
For Termius, keep Tailscale connected but disable the SSH intercept:

```sh
sudo tailscale set --ssh=false
```

UFW and the Hetzner firewall should still keep SSH private to the Tailscale
network.

For a phone or tablet, prefer a device-specific key:

1. Create a new Ed25519 key inside Termius.
2. Copy only the public key.
3. Add that public key as a separate `admin_authorized_keys` entry in ignored
   `templates/ansible/vars/local.yml`.
4. Rerun Ansible from the controller.
5. Connect Termius to the VPS Tailscale IP/hostname as the non-root admin user
   on port `22`.

After adding the key, label it clearly, record its public-key fingerprint, and
verify presence without printing all authorized keys:

```sh
ssh-keygen -l -f <(printf '%s\n' '<public-key>')
grep -F '<public-key-body>' ~/.ssh/authorized_keys >/dev/null && echo present
```

It is normal for `authorized_keys` to contain multiple keys when both controller
and phone/tablet access are enabled. After a successful login, rename the key in
Termius to something clear such as `hermes-vps-phone`.

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
