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
cd templates/ansible
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

## Public Webhooks Later

Do this as a separate change:

- Pick a public subdomain and configure DNS.
- Add a reverse proxy with TLS, such as Caddy or Traefik.
- Keep n8n bound to loopback behind the proxy if possible.
- Enable public `80` and `443` in Terraform and Ansible only after review.
- Configure n8n proxy URL settings:
  - `N8N_HOST`
  - `N8N_PROTOCOL=https`
  - `WEBHOOK_URL=https://<subdomain>/`
  - `N8N_PROXY_HOPS=1`
- Set `N8N_SECURE_COOKIE=true` once HTTPS is active.
- Review auth, MFA, backups, update policy, execution-data retention, and SSRF/file-access restrictions before exposing it.
