# n8n role

Optional private-by-default n8n deployment for Hermes VPS.

## Defaults

- Disabled by default in `templates/ansible/site.yml` with `install_n8n: false`.
- Service start is also disabled by default with `n8n_enable_service: false`.
- Host bind defaults to `127.0.0.1:5678`.
- n8n data persists under `/var/lib/n8n/data`.
- Local files for n8n workflows persist under `/var/lib/n8n/files` and mount as `/files`.
- `/etc/n8n/.env` is created on the VPS if missing and is never overwritten by Ansible.
- `N8N_ENCRYPTION_KEY` must be set explicitly in `/etc/n8n/.env` before the service can start.

## Enable Later

Set these in ignored `templates/ansible/vars/local.yml`:

```yaml
install_n8n: true
n8n_enable_service: false
```

Then create the files without starting n8n:

```bash
cd /Users/stevenmichiels/.codex/skills/hermes-vps/templates/ansible
ansible-playbook -i inventory.ini site.yml --limit vps --tags n8n
```

Set the encryption key on the VPS:

```bash
ssh <hermes-vps-ssh-target>
sudoedit /etc/n8n/.env
```

Replace `N8N_ENCRYPTION_KEY=__SET_ME__` with a strong key generated on the VPS:

```bash
openssl rand -hex 32
```

After that, set `n8n_enable_service: true` in ignored local vars and rerun the n8n tag.

## Access With SSH Tunnel

Keep `n8n_bind_address: "127.0.0.1"` and run:

```bash
ssh -N -L 5678:127.0.0.1:5678 <hermes-vps-ssh-target>
```

Open `http://127.0.0.1:5678` locally.

## Access Over Tailscale

Set `n8n_bind_address` to the VPS Tailscale IP in ignored local vars, for example:

```yaml
n8n_bind_address: "100.x.y.z"
```

The role allows `n8n_host_port` from `100.64.0.0/10` in UFW when bound to a Tailscale address. Then open:

```text
http://100.x.y.z:5678
```

This is still private to the tailnet and does not open public Hetzner firewall ports.

## Public Webhooks With Cloudflare Tunnel

Do this as a separate change:

- Keep n8n bound to `127.0.0.1:5678`.
- Use a locally managed Cloudflare Tunnel through the `cloudflared` role.
- Route only `^/webhook/` to `http://127.0.0.1:5678` in steady state.
- Keep `/webhook-test/`, `/webhook-waiting/`, `/rest/*`, and the editor/API private unless temporarily opened for setup.
- Keep public VPS `80`, `443`, and `5678` closed.
- Configure n8n public URL settings with:
  - `n8n_public_hostname`
  - `n8n_editor_base_url`
  - `n8n_proxy_hops`
  - `n8n_secure_cookie`
- Leave `N8N_SECURE_COOKIE=false` when the editor is reached through plain `http://127.0.0.1:5678` over SSH/Tailscale.
- Enable `n8n_sqlite_backup_enabled` only after setting an age recipient and a pinned SQLite sidecar image digest.
- Use webhook-level auth for any workflow with side effects.
