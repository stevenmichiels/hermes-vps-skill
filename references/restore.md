# hermes-vps Restore Runbook

Use this when the VPS is replaceable infrastructure and Hermes state is restored from backup.

## Expected Backup Contents

Backups created by `hermes-vps backup` include:

- `/etc/hermes`
- `/var/lib/hermes`
- `/etc/firecrawl` when Firecrawl has been installed
- `/etc/n8n` and `/var/lib/n8n` when n8n has been installed

`/var/lib/hermes` contains Hermes secrets and state, including `.env`, `config.yaml`, sessions, memories, skills, logs, OAuth files, and pairing/authorization state. Keep backups private and encrypted when copied off the VPS.

`/etc/firecrawl` contains Firecrawl runtime configuration and generated secrets, including `firecrawl.env`. The default Hermes backup does not include Firecrawl Docker volumes (`firecrawl_postgres-data`, `firecrawl_rabbitmq-data`, `firecrawl_redis-data`) because live database volume snapshots can be inconsistent. Preserve those separately with a cold Docker-volume backup if Firecrawl job history matters.

`/etc/n8n` contains n8n runtime configuration and encryption material. The
general Hermes `tar.gz` backup is secret-bearing and should not be treated as an
encrypted artifact unless another layer encrypts it. Prefer the n8n online
SQLite backup for n8n restore validation because it is age-encrypted and
contains a consistent SQLite `.backup` output plus `/etc/n8n` and relevant
tunnel config.

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

## n8n Encrypted SQLite Restore Dry Run

Use this before storing real n8n credentials or depending on n8n as durable
state. The dry run must not overwrite live `/etc/n8n` or `/var/lib/n8n`.

Prerequisites:

- Local age identity for the n8n backup recipient.
- An encrypted n8n backup artifact such as
  `/var/backups/n8n/hermes-n8n-sqlite-online-*.tar.gz.age`.
- The n8n Docker image already present or intentionally pulled.

Dry-run flow:

1. Create a temporary restore workspace on the restore host:

```sh
restore_dir="$(mktemp -d /tmp/hermes-n8n-restore-XXXXXX)"
chmod 0700 "$restore_dir"
```

2. Decrypt and extract without printing file contents:

```sh
age -d -i ~/.config/age/hermes-n8n-backup-identity.txt \
  hermes-n8n-sqlite-online-YYYY-MM-DD-HHMMSS.tar.gz.age \
  | tar -xzf - -C "$restore_dir" --no-same-owner
```

3. Build an isolated runtime directory:

```sh
install -d -m 0700 -o 1000 -g 1000 "$restore_dir/runtime/data" "$restore_dir/runtime/files"
install -m 0600 -o 1000 -g 1000 \
  "$restore_dir/var/lib/n8n/data/database-backup-YYYY-MM-DD-HHMMSS.sqlite" \
  "$restore_dir/runtime/data/database.sqlite"
```

4. Create a restore-only env file from the restored `/etc/n8n/.env`, preserving
   secret values but overriding public/editor URLs and the port for isolation.
   Do not print the env file:

```sh
awk -F= '
  BEGIN {
    skip["N8N_PORT"]=1; skip["N8N_LISTEN_ADDRESS"]=1; skip["N8N_HOST"]=1;
    skip["N8N_PROTOCOL"]=1; skip["WEBHOOK_URL"]=1;
    skip["N8N_EDITOR_BASE_URL"]=1; skip["N8N_SECURE_COOKIE"]=1;
    skip["N8N_RUNNERS_ENABLED"]=1; skip["DB_TYPE"]=1;
  }
  !($1 in skip) { print }
' "$restore_dir/etc/n8n/.env" > "$restore_dir/restore.env"
cat >> "$restore_dir/restore.env" <<'EOF'
DB_TYPE=sqlite
N8N_PORT=5678
N8N_LISTEN_ADDRESS=0.0.0.0
N8N_HOST=127.0.0.1
N8N_PROTOCOL=http
WEBHOOK_URL=http://127.0.0.1:15678/
N8N_EDITOR_BASE_URL=http://127.0.0.1:15678/
N8N_SECURE_COOKIE=false
GENERIC_TIMEZONE=Europe/Brussels
TZ=Europe/Brussels
EOF
chmod 0600 "$restore_dir/restore.env"
```

5. Start an isolated container bound only to loopback:

```sh
docker run -d --name n8n-restore-dryrun \
  --env-file "$restore_dir/restore.env" \
  -p 127.0.0.1:15678:5678 \
  -v "$restore_dir/runtime/data:/home/node/.n8n" \
  -v "$restore_dir/runtime/files:/files" \
  docker.n8n.io/n8nio/n8n:2.21.7
```

6. Validate:

```sh
curl -fsS http://127.0.0.1:15678/healthz || curl -fsSI http://127.0.0.1:15678/
docker exec n8n-restore-dryrun n8n list:workflow
docker exec n8n-restore-dryrun n8n export:workflow --all --output=/tmp/workflows.json
docker exec n8n-restore-dryrun n8n export:credentials --all --decrypted --output=/tmp/credentials.json
```

If the restored database has no credentials, `export:credentials` may exit with
"No credentials found". That proves the restore starts with the restored
`N8N_ENCRYPTION_KEY`, but it does not prove credential decryption. Before using
real credentials, create a disposable test credential, run a new encrypted
backup, repeat this dry run, and verify a decrypted credential export succeeds
without printing the exported secret values.

7. Clean up decrypted material:

```sh
docker rm -f n8n-restore-dryrun
rm -rf "$restore_dir"
```

## n8n Live Restore

Use only after a successful dry run.

1. Stop n8n:

```sh
sudo docker compose -f /etc/n8n/docker-compose.yml down
```

2. Restore `/etc/n8n` from the encrypted backup or from another trusted,
   encrypted source. Confirm `/etc/n8n/.env` contains the original
   `N8N_ENCRYPTION_KEY` before starting n8n.
3. Restore the SQLite backup as `/var/lib/n8n/data/database.sqlite`.
4. Restore `/var/lib/n8n/files` from a trusted full backup if workflows depend
   on local files.
5. Fix ownership and permissions:

```sh
sudo chown -R 1000:1000 /var/lib/n8n/data /var/lib/n8n/files
sudo chmod 0700 /var/lib/n8n/data
sudo chmod 0770 /var/lib/n8n/files
sudo chown -R root:root /etc/n8n
sudo chmod 0700 /etc/n8n
sudo chmod 0600 /etc/n8n/.env
```

6. Rerun Ansible with the n8n role and validate:

```sh
ansible-playbook -i templates/ansible/inventory.ini templates/ansible/site.yml --tags n8n
sudo docker compose -f /etc/n8n/docker-compose.yml ps
curl -fsS http://127.0.0.1:5678/healthz || curl -fsSI http://127.0.0.1:5678/
```
