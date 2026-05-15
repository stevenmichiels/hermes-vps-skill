# Changelog

## Unreleased

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
  reviewer never commits.

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
