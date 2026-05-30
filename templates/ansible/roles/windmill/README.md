# Windmill Role

Optional private Windmill stack for VPS workflow automation experiments.

The role is disabled by default:

```yaml
install_windmill: false
windmill_enable_service: false
```

It installs:

- Postgres for Windmill state and job queue
- Windmill server
- One default worker
- One native worker
- Windmill Extra for editor assistance

Private binding is enforced. `windmill_bind_address` must be `127.0.0.1` or a Tailscale `100.64.0.0/10` address.

Before enabling the service, set or let the role generate database credentials in `/etc/windmill/.env`. Never commit that file.

Validation on the VPS:

```bash
sudo docker compose --env-file /etc/windmill/.env -f /etc/windmill/docker-compose.yml ps
curl -fsS http://127.0.0.1:8000
```
