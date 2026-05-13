# hermes-vps Learnings

This file is for sanitized operational lessons only.

It must not contain real tokens, user IDs, channel IDs, hostnames, OAuth
payloads, IP addresses, private account labels, or local paths.

## Initial Notes

- Hermes Docker state is a single mounted data directory: host
  `/var/lib/hermes` to container `/opt/data`.
- Hermes Docker docs use `nousresearch/hermes-agent:latest`, but this skill
  pins stable release tags for VPS repeatability.
- Gateway API and health endpoint use port `8642`; keep it bound to
  `127.0.0.1` unless a public exposure decision is explicit.
- Telegram requires `TELEGRAM_BOT_TOKEN` and authorization via
  `TELEGRAM_ALLOWED_USERS`, `GATEWAY_ALLOWED_USERS`, or DM pairing.

## Launch Notes

- Load `vars/local.yml` from the controller, not the target VPS. Otherwise
  Ansible may silently fall back to defaults during bootstrap.
- Keep `hermes_enable_service=false` for the first hardening pass; the gateway
  should start only after `/var/lib/hermes/.env` contains the required provider
  and messaging values.
- Close bootstrap SSH in two layers after Tailscale is approved and tested:
  update the Hetzner firewall with Terraform, then remove the old bootstrap
  `/32` from UFW with Ansible.
- If a rebuilt VPS reuses an old public IP, SSH may correctly block on a stale
  `known_hosts` entry. Confirm the rebuild, then remove only the stale host
  entry.
- A fresh hardened VPS can be operationally correct while `hermes-vps status`
  is still red for missing Telegram config, missing allowed users, and a
  stopped gateway.
- The Compose service must load `hermes_env_file` with `env_file`; host-side
  status checks can see `/var/lib/hermes/.env` even when the running container
  cannot.
- The `/health` endpoint on port `8642` requires Hermes API server env values
  (`API_SERVER_ENABLED=true` and an `API_SERVER_KEY`) in addition to the port
  mapping.
- In Docker, set `API_SERVER_HOST=0.0.0.0` inside the container while keeping
  the Compose published host bind on `127.0.0.1`; otherwise the health endpoint
  works only from inside the container.
- Firecrawl is not part of Hermes Agent itself. If needed, run it as a separate
  private Compose stack on `127.0.0.1:3002` and attach only the API service to
  the Hermes Docker network with alias `firecrawl`.
- Firecrawl's upstream `nuq-postgres` image expects `POSTGRES_DB=postgres`;
  using another database can leave NUQ tables missing during initialization.
