# hermes-vps Restore Runbook

Use this when the VPS is replaceable infrastructure and Hermes state is restored from backup.

## Expected Backup Contents

Backups created by `hermes-vps backup` include:

- `/etc/hermes`
- `/var/lib/hermes`
- `/etc/firecrawl` when Firecrawl has been installed

`/var/lib/hermes` contains Hermes secrets and state, including `.env`, `config.yaml`, sessions, memories, skills, logs, OAuth files, and pairing/authorization state. Keep backups private and encrypted when copied off the VPS.

`/etc/firecrawl` contains Firecrawl runtime configuration and generated secrets, including `firecrawl.env`. The default Hermes backup does not include Firecrawl Docker volumes (`firecrawl_postgres-data`, `firecrawl_rabbitmq-data`, `firecrawl_redis-data`) because live database volume snapshots can be inconsistent. Preserve those separately with a cold Docker-volume backup if Firecrawl job history matters.

## Fresh VPS Recovery

1. Recreate infrastructure with Terraform.
2. Update `templates/ansible/inventory.ini` with the bootstrap IP.
3. Run Ansible with `hermes_enable_service=false` and `firecrawl_enable_service=false`.
4. Copy the backup archive to the VPS.
5. Stop runtime containers and restore as root:

```sh
sudo docker compose -f /etc/hermes/docker-compose.yml down || true
sudo docker compose --env-file /etc/firecrawl/firecrawl.env -f /opt/firecrawl/docker-compose.yml down || true
sudo tar -xzf hermes-vps-backup.tar.gz -C /
sudo chown -R 10000:10000 /var/lib/hermes
sudo chmod 0700 /var/lib/hermes
sudo chmod 0600 /var/lib/hermes/.env 2>/dev/null || true
sudo chown -R root:root /etc/firecrawl 2>/dev/null || true
sudo chmod 0750 /etc/firecrawl 2>/dev/null || true
sudo chmod 0600 /etc/firecrawl/firecrawl.env 2>/dev/null || true
```

6. If restoring Firecrawl Docker volumes from a separate cold backup, restore them before starting Firecrawl.
7. Start Docker and rerun Ansible with `hermes_enable_service=true` and the intended Firecrawl settings.
8. Validate:

```sh
hermes-vps status
curl -fsS http://127.0.0.1:8642/health
curl -fsS http://127.0.0.1:3002
docker compose -f /etc/hermes/docker-compose.yml logs --tail=100 hermes
docker compose --env-file /etc/firecrawl/firecrawl.env -f /opt/firecrawl/docker-compose.yml ps
```

9. Re-check Telegram authorization with an allowed user or pairing flow.
10. Validate non-root SSH, then close bootstrap SSH to Tailscale-only in both UFW and Hetzner firewall.
