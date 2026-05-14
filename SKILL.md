---
name: hermes-vps
description: Provision and harden a Hetzner VPS with Terraform and Ansible as a private AI-agent workbench for Hermes Agent, optional Firecrawl, Claude Code CLI, Codex CLI, and service agents, with strict SSH/firewall guardrails, secret-safe env management, backups, health checks, and repeatable service lifecycle steps.
---

# SKILL: hermes-vps (Hetzner + Terraform + Ansible + Hermes Agent)

## Purpose
Provision and harden a Hetzner Cloud VPS in a repeatable, safe-by-default workflow:
- Terraform: server + Hetzner firewall
- Ansible: base config + security (UFW + fail2ban + unattended-upgrades)
- Baseline tools: Docker, Docker Compose v2, jq, gh, ffmpeg, screen, Python 3/pipx
- Deploy Hermes Agent as a Docker Compose gateway on a non-root data directory
- Optional: deploy a private Firecrawl self-host stack for crawl/scrape/PDF extraction workflows
- Support one VPS as an agent workbench for Claude Code CLI, Codex CLI, and service agents while keeping live apps, staging, repos, and agent services in separate zones
- Provide operator commands for status, backups, release checks, health alerts, and Docker cleanup

## Source-of-truth rule
- Treat the official Hermes docs and upstream repo as source of truth:
  - https://hermes-agent.nousresearch.com/
  - https://github.com/NousResearch/hermes-agent
- If this skill drifts from upstream Hermes semantics, update templates before deploy.

## Hermes image release policy
- Prefer the newest stable GitHub Release, but keep deployments pinned to an explicit Docker tag.
- Do not deploy `latest` unless the user explicitly accepts floating-image behavior.
- Before bumping `hermes_image`:
  - check `https://github.com/NousResearch/hermes-agent/releases` for the latest non-draft, non-prerelease release
  - verify the matching Docker Hub tag exists, e.g. `nousresearch/hermes-agent:vYYYY.M.DD`
  - update `SKILL.md`, `templates/ansible/site.yml`, and role defaults together
  - deploy with `hermes_compose_pull=true`, then validate health, ports, Telegram auth, and backups
- Current template default: `nousresearch/hermes-agent:v2026.5.7`.

## Core operating workflow
1) Identify request type first: access, infra change, config change, runtime/service change, or assistant capability change.
2) Load current local artifacts before changing anything:
   - `hermes-vps/SKILL.md`
   - `hermes-vps/LEARNINGS.md`
   - active `.codex/plan/HERMES-*.md` files, if present
   - `hermes-vps/templates/infra/*`
   - `hermes-vps/templates/ansible/site.yml`
   - `hermes-vps/templates/ansible/roles/hermes/*`
3) Confirm whether change is Terraform, Ansible, or both.
4) Execute the smallest safe change first, then validate with concrete health checks.

## Local config and sharing rule
- The tracked skill must stay generic and shareable. Keep machine/account-specific values in ignored local files.
- Copy examples before first use:
  - `templates/ansible/inventory.ini.example` -> `templates/ansible/inventory.ini`
  - `templates/ansible/vars/local.yml.example` -> `templates/ansible/vars/local.yml`
  - `templates/infra/terraform.tfvars.example` -> `templates/infra/terraform.tfvars`
- `templates/ansible/site.yml` automatically loads `templates/ansible/vars/local.yml` from the controller when present.
- Do not commit local inventory, local Ansible vars, Terraform tfvars/state/plans, Ansible logs, backup archives, Hermes `.env`, OAuth profiles, pairing state, or SSH keys.
- Source-controlled Hermes runtime skill templates live under `templates/hermes-skills/` and are safe to copy because this tree must not contain secrets.

## VPS zoning model
- A single Hetzner VPS may host Hermes, OpenClaw, Claude Code CLI, Codex CLI, Docker containers, Git checkouts, and websites, but do not run them all as root in one shared directory.
- Keep live applications, staging, source repos, and long-running agent services as separate zones. Good default paths:
  - `/srv/apps/production-site` for live app runtime
  - `/srv/apps/staging-site` for staging/test runtime
  - `/home/<admin_user>/repos/project` for ordinary Git checkouts where Claude Code CLI, Codex CLI, or an operator edits code
  - `/home/<admin_user>/agent-workspaces/claude-code` for disposable/agent-specific workspaces
  - `/home/<admin_user>/agent-workspaces/codex-cli` for Codex CLI scratch work and separate sessions
  - `/opt/hermes` plus `/var/lib/hermes` for Hermes install and runtime state
  - `/opt/openclaw` for OpenClaw if co-hosted
  - `/opt/firecrawl` plus `/etc/firecrawl` for the optional private Firecrawl stack
  - `/var/backups` for backups
- The Ansible baseline creates the parent zone directories by default: `/srv/apps`, `/home/<admin_user>/repos`, `/home/<admin_user>/agent-workspaces`, `/opt/openclaw`, `/opt/hermes`, and `/var/backups`. The Firecrawl role manages `/opt/firecrawl` and `/etc/firecrawl` only when Firecrawl is enabled. Create concrete app/repo directories only when the real app or repo name is known.
- Prefer separate Unix users for persistent services when co-hosting multiple agents: an admin SSH user for operators, and service users such as `hermes`, `openclaw`, `claude-agent`, and `codex-agent` when their permissions or runtime state need isolation.
- Treat Claude Code CLI and Codex CLI as interactive coding tools by default: run them in SSH/tmux/mosh sessions against Git repos or disposable workspaces, with separate auth/config directories when possible.
- AI agents must not write directly to `/srv/apps/production-*`, `/var/www`, production `.env` files, or production databases. Use Git, reviewed diffs, CI/deploy scripts, Ansible, or a staging promotion step.
- Give agent services dev/staging credentials by default. Use production credentials only after an explicit decision, with the narrowest scope possible and no shared `.env` across Hermes, OpenClaw, Claude Code CLI, Codex CLI, and app runtimes.
- For a multi-agent VPS with Docker builds, browser automation, crawling, or databases, prefer at least 4-8 GB RAM and choose 8 GB when budget allows; enable swap and validate memory pressure before adding another service.

## Controller machine discipline
- When work continues from a different controller machine, rediscover local state instead of assuming previous inventory, SSH keys, shell aliases, Terraform env vars, or `known_hosts` entries exist.
- Prefer `tailscale ssh hermes-vps '<command>'` for read-only operator checks when the controller is already on the tailnet and the VPS has Tailscale SSH enabled.
- Do not create a new SSH key just because ordinary `ssh hermes-vps` fails. If Tailscale SSH works, keep using it unless Ansible or another tool explicitly needs standard SSH.
- Do not edit `known_hosts` to work around a host-key verification failure unless the operator explicitly approves that cleanup; stale host keys can be expected after a rebuild or IP reuse.
- If Ansible must run from the new controller, create or restore ignored local files deliberately (`inventory.ini`, `vars/local.yml`, Terraform vars) and validate connectivity before applying.

## Controller SSH helpers
- SSH helpers are optional controller-local convenience only; do not make the tracked skill depend on machine-specific shell aliases.
- Keep helpers in the controller user's shell config, not in this repo. A minimal zsh pattern is:
  ```bash
  hermes-ssh() { tailscale ssh hermes-vps "$@"; }
  hermes-status() { tailscale ssh hermes-vps 'sudo hermes-vps status'; }
  ```
- Validate helpers before relying on them:
  - `hermes-ssh 'hostname && whoami'`
  - `hermes-status`
- For operator checks, helpers may wrap Tailscale SSH. For normal Ansible applies, still use deliberate ignored local inventory/vars and the documented validation flow.
- If a helper fails but `tailscale ssh hermes-vps '<command>'` works, prefer fixing the local helper over changing VPS SSH configuration.

## Termius and SSH clients
- Termius should use normal SSH over the Tailscale network, not a public SSH exception.
- Requirements:
  - Termius device has Tailscale installed and is logged into the same tailnet.
  - Host is the VPS Tailscale IP or MagicDNS name, e.g. `hermes-vps` when MagicDNS works.
  - Port is `22`.
  - User is the non-root admin user from `admin_user`.
  - Authentication uses the private key matching `admin_authorized_keys`; password login stays disabled.
- For the current local deployment, the admin user is set in ignored `templates/ansible/vars/local.yml`; do not hard-code that private value into the public-cleaned repo.
- Preferred phone/tablet key flow:
  1. Create a new Ed25519 key inside Termius on the phone/tablet.
  2. Copy only the public key.
  3. Add that public key as a separate entry in `admin_authorized_keys` in ignored `templates/ansible/vars/local.yml`.
  4. Rerun Ansible from the controller.
  5. Connect Termius to the VPS Tailscale IP/hostname as the non-root admin user on port `22`.
  This gives the phone/tablet its own revocable key and avoids copying the controller's private key into Termius.
- After adding a phone/tablet key:
  - give the key a stable label such as `termius-phone` in `admin_authorized_keys`
  - record the public-key fingerprint with `ssh-keygen -l -f <(printf '%s\n' '<public-key>')`
  - verify presence without printing all keys: `grep -F '<public-key-body>' ~/.ssh/authorized_keys >/dev/null && echo present`
  - expect `authorized_keys` to contain more than one key when both controller and phone/tablet access are enabled
  - after a successful Termius login, rename the key inside Termius to a clear name such as `hermes-vps-phone`
- Validate from a shell before debugging Termius:
  ```bash
  ssh <admin_user>@<tailscale-ip-or-hostname> 'hostname && whoami'
  ```
- If Termius is on a phone or tablet, install and connect Tailscale on that device first. Do not copy VPS secrets, Hermes env files, Terraform state, or backups into Termius.

## macOS controller Ansible setup
- Prefer Homebrew on macOS when it is already installed:
  - `zsh -lc 'brew install ansible'`
  - `zsh -lc 'ansible --version && ansible-playbook --version'`
- Validate local syntax without contacting a real VPS:
  - `zsh -lc 'cd skills/hermes-vps/templates/ansible && ansible-playbook -i inventory.ini.example site.yml --syntax-check'`
- Real Ansible runs still require ignored local controller files (`inventory.ini`, `vars/local.yml`) and an explicit apply decision.
- If Homebrew is unavailable, ask before installing a package manager or choosing an alternate pipx-based Ansible install.
- Rollback for a Homebrew install is `zsh -lc 'brew uninstall ansible'`; run `brew autoremove` only after checking it will not remove dependencies used by other tools.

## macOS controller Terraform setup
- Terraform is needed only for Hetzner server/firewall work, not for Hermes-only Ansible changes.
- Prefer the official HashiCorp Homebrew tap on macOS when Homebrew is already installed:
  - `zsh -lc 'brew tap hashicorp/tap && brew install hashicorp/tap/terraform'`
  - `zsh -lc 'terraform version'`
- Validate local Terraform syntax without credentials or remote state:
  - `zsh -lc 'terraform -chdir=skills/hermes-vps/templates/infra fmt -check -diff'`
  - `zsh -lc 'terraform -chdir=skills/hermes-vps/templates/infra init -backend=false -input=false -lockfile=readonly'`
  - `zsh -lc 'terraform -chdir=skills/hermes-vps/templates/infra validate'`
- `terraform init -backend=false` may create an ignored `.terraform/` directory in the infra template folder; do not commit it.
- Real plans still require `templates/infra/terraform.tfvars` or equivalent vars plus `TF_VAR_hcloud_token`; never run `apply` unless the plan has been inspected and matches the approved scope.
- Rollback for a Homebrew install is `zsh -lc 'brew uninstall hashicorp/tap/terraform'`; run `brew autoremove` only after checking it will not remove dependencies used by other tools.

## Core principle
- Never expose SSH to the whole internet.
- Bootstrap with a narrow allow-list (your current public IP `/32`) or via Hetzner Console.
- After Tailscale is up, tighten rules to Tailscale-only in both UFW and the Hetzner firewall.

## Access discipline
- Prefer Tailscale-only SSH.
- Avoid root login unless explicitly requested and still enabled for a bootstrap step.
- Do not change SSH/80/443 public exposure without explicit approval.
- A fresh rebuild is not complete until Tailscale SSH is proven and public bootstrap SSH is removed from both UFW and the Hetzner firewall.
- Keep Hermes and Firecrawl service ports loopback-bound unless a public exposure decision is explicit.
- Do not reopen public SSH for Termius or another SSH client. Put the client device on the Tailscale tailnet and connect to the VPS Tailscale IP/hostname on port 22 as the non-root admin user with the configured SSH key.

## Hard safety rules
1) Never leave SSH open to the whole internet. SSH must be restricted to a bootstrap `/32`, then Tailscale-only.
2) Never enable UFW without allow rules that preserve access.
3) Root login disable is gated: validate non-root SSH first, then set `disable_root_login=true`.
4) Never enable password login. Keep `PasswordAuthentication no`.
5) Never print secrets. Never commit API keys, bot tokens, OAuth files, pairing state, or `/var/lib/hermes/.env`.
6) Keep Hermes gateway/dashboard private by default: bind published ports to `127.0.0.1` and access via SSH tunnel or Tailscale.
7) Keep Firecrawl private by default: bind the API to `127.0.0.1` and attach only its API service to the Hermes Docker network.
8) For existing servers, never apply Terraform if plan shows `hcloud_server` destroy/replace unless rebuild is explicitly intended.
9) Keep deterministic artifacts as source of truth: Terraform + Ansible templates in this skill.
10) Keep production apps, staging, repos, and agent workspaces isolated; never let an agent directly mutate production runtime files or share production secrets by default.

## Credential loading (Hetzner token)
- Preferred/default path: export `TF_VAR_hcloud_token` from a secure secret source before Terraform commands.
- If the operator has a shell helper for Hetzner credentials, load that profile first and run the helper in the active shell.
- If the helper exports `HCLOUD_TOKEN`, export Terraform's expected variable from it: `export TF_VAR_hcloud_token="$HCLOUD_TOKEN"`.
- Never assume helper output is token-only; avoid command substitution with helpers unless their behavior is verified.

### Optional macOS `hcloudtoken` helper
- This is optional macOS controller convenience, not a dependency of the skill. Keep it in the operator's shell profile or local secret tooling, never in this repo.
- Example zsh helper shape:
  ```bash
  hcloudtoken() {
    export HCLOUD_TOKEN="$(security find-generic-password -a "$USER" -s hcloud-token -w)"
    export TF_VAR_hcloud_token="$HCLOUD_TOKEN"
    echo "HCLOUD_TOKEN and TF_VAR_hcloud_token exported for this shell"
  }
  ```
- Store the token in macOS Keychain first, outside the repo:
  ```bash
  security add-generic-password -a "$USER" -s hcloud-token -w '<paste-token-here>'
  ```
- Use it by loading the shell profile, running `hcloudtoken`, then running `terraform plan`.
- Do not use `TF_VAR_hcloud_token="$(hcloudtoken)"`; the helper prints status text and is meant to set environment variables in the current shell.

## Hetzner CLI / hcloud
- `hcloud` is optional. Terraform remains the source of truth for managed Hetzner server and firewall changes.
- Install on a macOS controller only when Hetzner read-only inspection or emergency operator checks are useful:
  - `zsh -lc 'brew install hcloud'`
  - `zsh -lc 'hcloud version'`
- Preferred auth for temporary controller sessions: load `HCLOUD_TOKEN` from a secure secret source in the active shell, then run read-only `hcloud` commands.
- Safe read-only checks:
  - `hcloud server list`
  - `hcloud server describe hermes-vps`
  - `hcloud firewall list`
  - `hcloud firewall describe hermes-vps-fw`
- Do not use `hcloud` to mutate Terraform-managed resources unless the operator explicitly approves a manual/emergency change. After any approved manual change, reconcile the intended state back into Terraform before the next apply.
- Do not commit `hcloud` contexts, tokens, command output containing account details, Terraform tfvars, plans, or state.

## GitHub CLI on VPS
- `gh` is installed as baseline tooling, but Hermes does not require GitHub auth for the default Telegram/Firecrawl runtime.
- Do not authenticate `gh`, create VPS GitHub SSH keys, or add deploy keys unless host-side repo operations are explicitly needed.
- If repo operations become required, use a host-side admin-user key and device-flow `gh auth login`; keep runtime secrets and `/var/lib/hermes` state out of Git repositories.
- Do not expose `/root/.ssh` or mount host SSH keys into the Hermes container without an explicit design.

## Hermes skill deployment
- Treat `/var/lib/hermes/skills` as runtime state, not the long-term source of truth for deployment-specific custom skills.
- Reusable public skills should be authored in the Hermes Agent source tree as `skills/<category>/<name>/SKILL.md` and committed there.
- VPS-specific skills should live in a source-controlled config repo or an explicit `hermes-vps` template path before being deployed to `/var/lib/hermes/skills`.
- Current source-controlled VPS-specific Hermes runtime skills:
  - `templates/hermes-skills/web/private-firecrawl/SKILL.md`
- Do not hand-edit runtime skills and call the VPS reproducible unless the same change exists in source control.
- Keep secret-bearing runtime files, OAuth profiles, pairing state, sessions, and `/var/lib/hermes/.env` out of any skills repo.
- The Hermes role copies `templates/hermes-skills/` to `/var/lib/hermes/skills/` when `hermes_runtime_skills_enabled=true`; it does not delete unrelated runtime skills.
- After adding or changing a source-controlled runtime skill, validate discovery in a fresh Hermes session before calling the deployment reproducible.
- For Hermes CLI session tests of local category skills, preload by category path, e.g. `--skills web/private-firecrawl`; `sudo docker exec hermes /opt/hermes/.venv/bin/hermes skills list` may show the display name while direct preload/inspect by display name can be less reliable for local skills. Do not use `/opt/hermes/hermes` for this check unless its Python environment has been verified.

## Hermes Firecrawl crawl skill decision
- A Hermes Firecrawl crawl skill fits Hermes skill semantics: Hermes skills are normal `SKILL.md` files, and existing Hermes skills already reference `web_extract` and Firecrawl-backed PDF/markdown extraction.
- Do not add a runtime-only crawl skill directly under `/var/lib/hermes/skills`.
- Create a source-controlled crawl skill only if it adds behavior beyond existing `web_extract` usage, such as explicit private Firecrawl endpoint workflows, crawl-job handling, batch scrape recipes, or Firecrawl failure recovery.
- The skill should prefer `web_extract` for ordinary URL/PDF extraction and use `http://firecrawl:3002` or `http://127.0.0.1:3002` only for workflows that require the private self-hosted Firecrawl stack.
- Validate any deployed crawl skill in a fresh Hermes session and with a real `/v1/scrape` request before declaring it ready.

## Remaining follow-ups
- Iterate on `templates/hermes-skills/web/private-firecrawl/SKILL.md` only when private-stack workflows need behavior that `web_extract` does not already handle well.
- Keep `hermes-vps status` checking Firecrawl's Hermes-network wiring when Firecrawl is enabled, especially that the API service is attached to the Hermes Docker network with alias `firecrawl`.

## Inputs / knobs (Ansible)
- `admin_user` (default: `admin`)
- `admin_authorized_keys` (default: `[]`; required before `disable_root_login=true`)
- `disable_root_login` (default: `false`)
- `tailscale_ssh_source_cidr` (default: `100.64.0.0/10`)
- `bootstrap_public_ssh_cidr` (repo default: `""`; first public bootstrap must be your current public IP `/32`)
- `bootstrap_public_ssh_cidr_cleanup` (default: `""`; optional)
- `allow_http`, `allow_https` (defaults: `false`)
- `swap_enabled`, `swap_size_mb`, `swap_swappiness`
- `vps_zone_dirs_enabled` (default: `true`)
- `vps_zone_dirs` (default parent zones: `/srv/apps`, `/home/<admin_user>/repos`, `/home/<admin_user>/agent-workspaces`, `/opt/openclaw`, `/var/backups`; Hermes role also manages `/opt/hermes`; Firecrawl role manages `/opt/firecrawl` and `/etc/firecrawl` when enabled)
- `install_hermes` (default: `true`)
- `install_firecrawl` (default: `false`; optional private Firecrawl stack)
- `hermes_enable_service` (default: `false`; set `true` only after `/var/lib/hermes/.env` is configured)
- `hermes_require_telegram` (default: `true`; requires `TELEGRAM_BOT_TOKEN` before service enable)
- `hermes_require_allowed_users` (default: `true`; requires `TELEGRAM_ALLOWED_USERS` or `GATEWAY_ALLOWED_USERS`)
- `hermes_image` (default: `nousresearch/hermes-agent:v2026.5.7`)
- `hermes_gateway_bind_address` (default: `127.0.0.1`)
- `hermes_gateway_port` (default: `8642`)
- `hermes_dashboard_enabled` (default: `false`)
- `hermes_dashboard_port` (default: `9119`)
- `hermes_data_dir` (default: `/var/lib/hermes`; mounted to container `/opt/data`)
- `hermes_compose_file` (default: `/etc/hermes/docker-compose.yml`)
- `hermes_runtime_skills_enabled` (default: `true`; copies `templates/hermes-skills/` to `/var/lib/hermes/skills/`)
- `hermes_runtime_skills_src_dir` (default: `templates/hermes-skills` relative to the skill template root)
- `hermes_runtime_skills_dir` (default: `/var/lib/hermes/skills`)
- `hermes_backup_dir` (default: `/var/backups/hermes-vps`)
- Timer toggles: `hermes_backup_timer_enabled`, `hermes_docker_cleanup_timer_enabled`, `hermes_release_check_timer_enabled`, `hermes_healthcheck_timer_enabled`
- Firecrawl knobs:
  - `firecrawl_enable_service` (default: `false`)
  - `firecrawl_env_dir`, `firecrawl_env_file` (defaults: `/etc/firecrawl`, `/etc/firecrawl/firecrawl.env`)
  - `firecrawl_dir`, `firecrawl_compose_file` (defaults: `/opt/firecrawl`, `/opt/firecrawl/docker-compose.yml`)
  - `firecrawl_api_bind_address` (default: `127.0.0.1`)
  - `firecrawl_api_port`, `firecrawl_internal_port` (defaults: `3002`)
  - `firecrawl_image`, `firecrawl_playwright_image`, `firecrawl_postgres_image`
  - `firecrawl_postgres_db` (must be `postgres` with upstream `nuq-postgres`)
  - `firecrawl_hermes_network` (default: `hermes_default`; exposes only the Firecrawl API service to Hermes)
  - `firecrawl_auto_generate_secrets` (default: `true`)
  - low-concurrency knobs: `firecrawl_num_workers_per_queue`, `firecrawl_crawl_concurrent_requests`, `firecrawl_max_concurrent_jobs`, `firecrawl_browser_pool_size`, CPU/memory limits, `firecrawl_block_media`, `firecrawl_logging_level`

## Inputs / knobs (Terraform)
- `server_name`, `firewall_name`, `image`, `server_type`, `location`
  - for a new default Hermes server, use `server_name = "hermes-vps"`
  - `firewall_name` is optional; when unset, Terraform uses `<server_name>-fw`
- `ssh_key_names` (list)
- `ssh_source_ips` (list; first bootstrap should be your public IP `/32`)
- `allow_http`, `allow_https` (default false)
- Existing-server caution: changing `server_name` or `ssh_key_names` may force replacement.

## Terraform change discipline
- Treat `templates/infra/terraform.tfvars` as the current server definition.
- Always run `terraform -chdir=templates/infra plan` before `apply`.
- If the plan shows `hcloud_server.vps` destroy/create or replacement, stop and ask before applying.
- Treat `server_name` and `ssh_key_names` as immutable unless a rebuild is explicitly intended.
- `firewall_name`, `ssh_source_ips`, `allow_http`, and `allow_https` should normally update the firewall in place; still inspect the plan before applying.
- Do not use `-refresh=false` for real infrastructure changes. It is only a local syntax/sanity shortcut when credentials are unavailable.

## Deployment workflow
1) Configure Terraform variables and bootstrap SSH CIDR.
2) Run:
   - `terraform -chdir=templates/infra init`
   - `terraform -chdir=templates/infra plan`
   - `terraform -chdir=templates/infra apply` only if the plan matches intent
3) Set generated public VPS IP in `templates/ansible/inventory.ini` for the root bootstrap run.
4) First Ansible run with:
   - `disable_root_login=false`
   - `hermes_enable_service=false`
   - `bootstrap_public_ssh_cidr=<your public IP /32>`
5) Validate non-root SSH, then set:
   - inventory user to the admin user
   - `disable_root_login=true`
   - rerun `ansible-playbook -i inventory.ini site.yml`
6) Join Tailscale on the VPS:
   - run `sudo tailscale up --ssh --hostname=hermes-vps`
   - approve the auth link
   - validate SSH through the Tailscale IP before closing bootstrap SSH
   - for operator checks from a tailnet-connected controller, prefer `tailscale ssh hermes-vps 'sudo hermes-vps status'`
7) Close bootstrap SSH to Tailscale-only in both layers:
   - Terraform: set `ssh_source_ips = ["100.64.0.0/10"]`, plan, and apply only if the plan changes the firewall in place
   - Ansible inventory: set `ansible_host` to the Tailscale IP
   - Ansible vars: set `bootstrap_public_ssh_cidr: ""` and `bootstrap_public_ssh_cidr_cleanup: "<old public IP /32>"`
   - rerun Ansible, then verify public SSH fails and Tailscale SSH still works
8) Configure Hermes secrets on the VPS:
   - edit `/var/lib/hermes/.env`, or
   - run `sudo docker compose -f /etc/hermes/docker-compose.yml run --rm hermes setup`
   - set `API_SERVER_KEY` when the local API/health endpoint is enabled
9) For Telegram:
   - create a bot with @BotFather
   - set `TELEGRAM_BOT_TOKEN`
   - set `TELEGRAM_ALLOWED_USERS` to numeric Telegram user IDs, or intentionally use Hermes DM pairing
   - for groups, disable Telegram bot privacy mode or make the bot admin
10) Enable runtime:
   - set `hermes_enable_service=true`
   - rerun `ansible-playbook -i inventory.ini site.yml`
11) Validate:
   - `hermes-vps status`
   - `curl -fsS http://127.0.0.1:8642/health`
   - Telegram `/start` or a direct message from an allowed user
12) Optional Firecrawl:
   - set `install_firecrawl=true` and `firecrawl_enable_service=true`
   - rerun Ansible after Hermes has created the `hermes_default` Docker network
   - validate `curl -fsS http://127.0.0.1:3002`
   - validate from the Hermes network with a scrape request to `http://firecrawl:3002/v1/scrape`

Before Hermes env values are configured and the service is enabled, `hermes-vps status` should be expected to report missing Telegram/allowed-user config and an unreachable gateway. Treat those as expected bootstrap failures, not as a hardened-VPS failure.

## Operator commands on VPS
- `hermes-vps status` - full local status check
- `hermes-vps backup` - immediate backup
- `hermes-vps release-check` - check newer stable Hermes release, no auto-upgrade
- `hermes-vps healthcheck` - status check plus Telegram transition alert when configured
- `hermes-vps docker-cleanup` - prune unused Docker artifacts without pruning volumes
- `hermes-vps timers` - list installed timers

## Validation checklist
- Overall status:
  - `sudo hermes-vps status`
  - `sudo hermes-vps timers`
- Hermes runtime skills:
  - `sudo test -f /var/lib/hermes/skills/web/private-firecrawl/SKILL.md`
  - `sudo docker exec hermes sh -lc 'test -f /opt/data/skills/web/private-firecrawl/SKILL.md'`
  - `sudo docker exec hermes /opt/hermes/.venv/bin/hermes skills list | grep -F private-firecrawl`
- Hermes runtime:
  - `sudo docker compose -f /etc/hermes/docker-compose.yml ps`
  - `curl -fsS http://127.0.0.1:8642/health`
  - `sudo docker compose -f /etc/hermes/docker-compose.yml logs --tail=100 hermes`
- Required env values without printing secrets:
  - `sudo bash -lc 'for k in TELEGRAM_BOT_TOKEN API_SERVER_KEY; do grep -Eq "^${k}=.+" /var/lib/hermes/.env && echo "${k}=set" || echo "${k}=missing"; done; grep -Eq "^(TELEGRAM_ALLOWED_USERS|GATEWAY_ALLOWED_USERS)=.+" /var/lib/hermes/.env && echo "allowed_users=set" || echo "allowed_users=missing"'`
- Swap and disk:
  - `swapon --show`
  - `free -h`
  - `sysctl vm.swappiness`
  - `df -h /`
- Port exposure sanity:
  - `sudo ss -ltnp | grep -E ':22 |:8642 |:9119 |:3002 ' || true`
  - expected: Hermes, dashboard, and Firecrawl ports are loopback-bound; SSH is reachable only through allowed Tailscale/bootstrap sources.
- Firecrawl when enabled:
  - `curl -fsS http://127.0.0.1:3002`
  - `sudo docker compose --env-file /etc/firecrawl/firecrawl.env -f /opt/firecrawl/docker-compose.yml ps`
  - `sudo hermes-vps status` should include `firecrawl_api_on_hermes_network=hermes_default` and `firecrawl_api_alias=firecrawl`
  - `curl -fsS -X POST http://127.0.0.1:3002/v1/scrape -H 'Content-Type: application/json' -d '{"url":"https://docs.firecrawl.dev","formats":["markdown"]}'`
- Backup and restore evidence:
  - `sudo hermes-vps backup`
  - `sudo tar -tzf /var/backups/hermes-vps/latest.tar.gz | grep -E '^(etc/hermes|var/lib/hermes|etc/firecrawl)' | head`
- Terraform safety before infra changes:
  - load `TF_VAR_hcloud_token` through the documented `hcloudtoken` flow
  - `terraform -chdir=templates/infra plan`
  - stop before apply if the plan replaces `hcloud_server.vps`

## Firecrawl private stack
- Use official Firecrawl self-host docs and upstream repo as source of truth:
  - `https://docs.firecrawl.dev/contributing/self-host`
  - `https://github.com/firecrawl/firecrawl`
- Current self-host runtime uses Redis, RabbitMQ, and NUQ on Postgres. Keep `POSTGRES_DB=postgres` with the upstream `nuq-postgres` image.
- Keep Firecrawl private:
  - host endpoint: `http://127.0.0.1:3002`
  - Hermes Docker-network endpoint: `http://firecrawl:3002`
  - laptop tunnel: `ssh -N -L 3002:127.0.0.1:3002 hermes-vps`
- Runtime layout:
  - compose: `/opt/firecrawl/docker-compose.yml`
  - env: `/etc/firecrawl/firecrawl.env` (secret-bearing; never commit)
- `hermes-vps backup` includes `/etc/firecrawl` when present. Firecrawl Docker volumes are not included by default; use a cold volume backup if Firecrawl job history must survive a rebuild.
- The Firecrawl role is gated by `install_firecrawl`; service state is gated by `firecrawl_enable_service`.
- The API service joins `hermes_default` with alias `firecrawl`; Redis, RabbitMQ, and Postgres remain on the private `firecrawl_backend` network.
- Env secrets are generated only when missing/placeholder and `firecrawl_auto_generate_secrets=true`.
- If API restarts with `relation "nuq.queue_*" does not exist`, ensure `POSTGRES_DB=postgres`; if this is a fresh failed init, recreate only Firecrawl volumes with `docker compose --env-file /etc/firecrawl/firecrawl.env -f /opt/firecrawl/docker-compose.yml down -v`, then `up -d`.

## Codex CLI Firecrawl access
- Keep Firecrawl hosted as its own private service stack under `/opt/firecrawl` and `/etc/firecrawl`; do not move it under `/home/<admin_user>` just so Codex CLI can use it.
- Codex CLI runs on the VPS host, so it can reach the private Firecrawl API through `http://127.0.0.1:3002` when Firecrawl is enabled.
- Preferred Codex integration is a local MCP server that points to the private host endpoint:
  ```bash
  codex mcp add firecrawl --env FIRECRAWL_API_URL=http://127.0.0.1:3002 -- npx -y firecrawl-mcp
  ```
- When `install_codex_cli=true`, the base role installs the distro `bubblewrap`
  package so Codex CLI can find `bwrap` on Linux instead of falling back to its
  bundled sandbox helper.
- This adds user-local Codex config in `~/.codex/config.toml`; do not commit it or copy it into the public skill repo as account state.
- Validate without exposing Firecrawl publicly:
  - `codex mcp list`
  - `command -v bwrap && bwrap --version`
  - `curl -fsS http://127.0.0.1:3002`
  - In an interactive Codex session, ask Codex to use the `firecrawl` MCP server to scrape a public test page.
- For non-interactive `codex exec`, MCP tool approval can cancel the tool call unless the run is interactive or explicitly configured to allow the call. Use bypass only for a controlled smoke test in an empty/safe workspace, never as the default operating mode.

## Restore
For recovery from a fresh VPS, read `references/restore.md`.
