# Changelog

## Unreleased

- Document the preferred Termius phone/tablet SSH key flow: create a
  device-specific Ed25519 key in Termius, add only the public key to
  `admin_authorized_keys`, then rerun Ansible.
- Install the distro `bubblewrap` package when `install_codex_cli=true` so
  Codex CLI can find `bwrap` on Linux VPS hosts instead of warning and falling
  back to its bundled sandbox helper.

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
