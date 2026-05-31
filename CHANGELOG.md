# Changelog

## Unreleased

- Document that controller-local Cloudflare tunnel credentials should be hidden
  from agent reads with local Codex/Claude deny or exclude rules before
  `cloudflared tunnel login`.
- Align the Cloudflare Tunnel site default binary path with package installs.
- Document the locally managed Cloudflare Tunnel bootstrap flow, including
  hostname prerequisites, controller-side `cloudflared tunnel login/create/route
  dns`, sensitive file handling, first-pass hello-world validation, and
  webhook-only steady state.
- Add an opt-in `cloudflared_install_method: apt_repo` path that configures
  Cloudflare's official apt repository before installing the `cloudflared`
  package.
- Add disabled-by-default Cloudflare Tunnel support for n8n public production
  webhooks while keeping the n8n editor/API private and the VPS closed to
  public `80`, `443`, and `5678`.
- Add disabled-by-default n8n SQLite online backup support using a pinned
  SQLite sidecar image, SQLite `.backup`, integrity checks, encrypted age
  artifacts, and an optional systemd timer.
- Add a disabled-by-default Hermes backup off-box gate with rsync-over-SSH
  upload, remote checksum verification, status freshness checks, and a visible
  retention-prune-disabled sentinel until off-box copy is configured.
- Add an off-box retry-pending marker so a temporarily unavailable personal
  target, such as a Mac over Tailscale, does not discard the local archive.
- Restore Hermes backup retention pruning only when a fresh verified off-box
  state file exists.
- Add an opt-in private n8n role that is disabled by default, binds to loopback
  or an explicit Tailscale IP only, persists `/var/lib/n8n`, creates
  `/etc/n8n/.env` without overwriting it, and refuses to start until
  `N8N_ENCRYPTION_KEY` is set on the VPS. Include `/etc/n8n` and
  `/var/lib/n8n` in Hermes backups when present.

## v0.3.5 - 2026-05-22

v0.3.5 adds an opt-in NoMachine/XFCE desktop profile while preserving the
headless, private-by-default VPS posture.

- Add an opt-in NoMachine profile that installs the official NoMachine DEB,
  disables UPnP/NAT-PMP port mapping and NoMachine firewall autoconfiguration,
  removes installer-created public UFW rules, requires NX private-key
  authentication, forces virtual desktops to start XFCE, and allows TCP/UDP
  `4000` only from the Tailscale CIDR.
- Make `install_remote_desktop` and `install_nomachine` a paired profile,
  and remove the previous SSH-desktop package defaults.
- Document concrete NoMachine connection values, the expected virtual-display
  prompt, direct Tailscale-only access, and 4 GB minimum / 8 GB preferred
  resource guidance.
- Rename the public repository and README title to `dual-agent-vps`.

Validation:

- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- `git diff --check`

## v0.3.4 - 2026-05-15

v0.3.4 makes the Codex-to-Claude review helper more robust in sandboxed
environments.

- Fall back from the default `.codex/reviews/` output path to
  `${TMPDIR:-/tmp}/codex-claude-review-*.md` when the default location is not
  writable.
- Document `CLAUDE_REVIEW_FALLBACK_DIR` for choosing a different fallback
  directory.

Validation:

- `bash -n templates/codex-skills/claude-review/scripts/codex-claude-review`
- `bash templates/codex-skills/claude-review/scripts/codex-claude-review -h`
- Local fake-Claude smoke test for default output
- Local fake-Claude smoke test for read-only default output fallback
- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- `ansible-playbook -i inventory.ini site.yml --tags agent-review`
- Remote verification of `/usr/local/bin/codex-claude-review -h`
- `git diff --check`

## v0.3.3 - 2026-05-15

v0.3.3 adds the reverse dual-agent review workflow: Claude Code can ask Codex
CLI for a read-only Markdown review through Codex's native review command.

- Add a `codex-review` Claude skill template with a bundled
  `claude-codex-review` helper.
- Install the reverse helper into the VPS operator's `~/.claude/skills` and
  expose `claude-codex-review` on the system PATH.
- Move admin-user creation before user-owned skill installs so fresh VPS
  applies do not depend on the user already existing.
- Add a sharper README positioning sentence for the hardened agent-workbench
  use case.

Validation:

- `bash -n templates/claude-skills/codex-review/scripts/claude-codex-review`
- `bash templates/claude-skills/codex-review/scripts/claude-codex-review -h`
- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- `ansible-playbook -i inventory.ini site.yml --tags agent-review`
- Remote verification of `/home/steven/.claude/skills/codex-review/SKILL.md`
- Local and remote fake-Codex smoke tests for `claude-codex-review`
- `git diff --check`

## v0.3.2 - 2026-05-15

v0.3.2 tightens the release notes for the dual-agent review workflow.

- Explain why `codex-claude-review` matters: it makes review workflows explicit
  and auditable instead of relying on an agent that both changes and judges the
  same code.

## v0.3.1 - 2026-05-15

v0.3.1 fixes the Claude review workflow installation so Codex CLI on the VPS
can discover the `claude-review` skill, not just call the helper command.

- Install the `claude-review` Codex skill into the VPS operator's
  `~/.codex/skills` instead of only installing the `codex-claude-review`
  helper command.

Validation:

- `bash -n templates/codex-skills/claude-review/scripts/codex-claude-review`
- `bash templates/codex-skills/claude-review/scripts/codex-claude-review -h`
- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- Skill validation for `templates/codex-skills/claude-review`
- `ansible-playbook -i inventory.ini site.yml --tags agent-review`
- Remote verification of `/home/steven/.codex/skills/claude-review/SKILL.md`
- `git diff --check`

## v0.3.0 - 2026-05-15

v0.3.0 adds practical agent-workbench tooling on top of the v0.2.0 VPS model:
Python tool execution with uv/uvx, a controlled Codex-to-Claude review helper,
LangChain docs MCP guidance, and clearer operator access notes.

Highlights:

- Document the preferred Termius phone/tablet SSH key flow: create a
  device-specific Ed25519 key in Termius, add only the public key to
  `admin_authorized_keys`, then rerun Ansible.
- Add Termius key post-add verification guidance: label the key, record its
  fingerprint, verify presence without printing `authorized_keys`, and rename
  the Termius key after a successful login.
- Document that Termius should use OpenSSH over the Tailscale network, and that
  Tailscale SSH intercept may need to be disabled with
  `sudo tailscale set --ssh=false` if Termius hangs on authentication.
- Install the distro `bubblewrap` package when `install_codex_cli=true` so
  Codex CLI can find `bwrap` on Linux VPS hosts instead of warning and falling
  back to its bundled sandbox helper.
- Document adding the official LangChain docs MCP server to user-local Codex
  CLI config on Hermes workbench hosts.
- Add a pinned `uv`/`uvx` workbench baseline with `install_uv`, `refresh_uv`,
  `uv_version`, and `uv_install_dir` knobs; keep Poetry repo-specific.
- Add a `codex-claude-review` helper for the controlled dual-agent workflow:
  one agent develops, Claude Code writes a read-only Markdown review, and the
  reviewer never commits. This makes review workflows explicit and auditable
  instead of relying on an agent that both changes and judges the same code.

Validation:

- `bash -n templates/ansible/roles/base/templates/codex-claude-review.j2`
- `bash templates/ansible/roles/base/templates/codex-claude-review.j2 -h`
- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- `git diff --check`
- `git status --short`

## v0.2.0

v0.2.0 turns the template from a Hermes-only VPS deploy skill into a broader
private AI-agent workbench model. It adds explicit VPS zoning for live apps,
repos, agent workspaces, and service runtimes, plus optional Claude Code CLI,
Codex CLI, and private Firecrawl integration.

This release is the first version where the repository clearly behaves as an
opinionated AI-agent VPS operating model: a safe, rebuildable, agent-ready VPS
for personal or small-team workflows.

Highlights:

- Add an agent-workbench zoning model for app runtimes, staging, repos,
  disposable agent workspaces, OpenClaw, Hermes, Firecrawl, and backups.
- Add optional Claude Code CLI and Codex CLI install toggles to the Ansible base
  role.
- Add a source-controlled Hermes runtime skill for the private Firecrawl stack.
- Add a Hermes gateway URL-prefetch wrapper that hands public URL content to
  Hermes through local artifacts.
- Document Codex CLI access to private Firecrawl through a local MCP server.
- Document Termius access over Tailscale-only SSH.
- Bump the default pinned Hermes image to
  `nousresearch/hermes-agent:v2026.5.7`.
- Add architecture, safe-review, and AI-agent workbench framing to the public
  README.

Validation:

- `terraform -chdir=templates/infra fmt -check -diff`
- `terraform -chdir=templates/infra init -backend=false -input=false -lockfile=readonly`
- `terraform -chdir=templates/infra validate`
- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- `git diff --check`
- Secret-pattern scan reviewed; only documentation/helper references were
  present.

## v0.1.0

Initial sanitized deployment template for provisioning and operating Hermes
Agent on a Hetzner VPS with Terraform, Ansible, private-by-default access,
backups, health checks, and restore notes.
