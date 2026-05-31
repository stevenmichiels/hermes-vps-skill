# Cloudflared role

Optional Cloudflare Tunnel support for Hermes VPS.

This role is disabled by default:

```yaml
install_cloudflared: false
cloudflared_enable_service: false
```

It supports the n8n public-webhook/private-editor pattern:

- n8n remains bound to `127.0.0.1:5678`.
- the VPS does not need public `80`, `443`, or `5678`.
- Cloudflare Tunnel exposes only selected public paths.

On Ubuntu/Debian hosts, prefer the official Cloudflare apt repository:

```yaml
cloudflared_install_method: apt_repo
```

The role installs the Cloudflare apt signing key and repository, then installs
the `cloudflared` package. The service still refuses to start until the tunnel
credentials JSON is already present on the VPS.

When `cloudflared_enable_service` is true, a rendered config or systemd unit
change restarts `cloudflared` so route updates take effect immediately.

## Bootstrap shape

Use a locally managed tunnel. No Cloudflare API token is needed for this path.
The controller only needs a browser login through `cloudflared`.

### 1. Confirm the hostname

Pick a hostname inside a domain that is already managed by Cloudflare
nameservers, for example:

```text
n8n.example.com
```

Do not create an `A` record pointing at the VPS. The tunnel route creates a
Cloudflare DNS `CNAME` pointing at `<tunnel-uuid>.cfargotunnel.com`.

### 2. Install controller-side cloudflared

On macOS with Homebrew:

```bash
brew install cloudflared
cloudflared --version
```

This is only for login, tunnel creation, and DNS routing from the controller.
The VPS-side daemon is installed by Ansible when
`cloudflared_install_method: apt_repo` is set.

### 3. Authenticate, create, and route the tunnel

```bash
cloudflared tunnel login
cloudflared tunnel create hermes-n8n
cloudflared tunnel route dns hermes-n8n n8n.<domain>
cloudflared tunnel list
```

The login command writes an account-scoped `cert.pem` on the controller. Treat
that file as sensitive because it can manage tunnels in the Cloudflare account.
Before running login, add local agent deny/exclude rules for:

```text
~/.cloudflared/cert.pem
~/.cloudflared/*.json
```

The create command writes a tunnel-specific credentials JSON such as:

```text
~/.cloudflared/<tunnel-uuid>.json
```

That JSON can run only its associated tunnel. Copy only that JSON to the VPS
path configured by `cloudflared_credentials_file`; do not copy `cert.pem` to
the VPS and do not paste either file into chat.

### 4. Set ignored local Ansible vars

In `templates/ansible/vars/local.yml`:

```yaml
install_cloudflared: true
cloudflared_enable_service: false
cloudflared_install_method: apt_repo
cloudflared_tunnel_uuid: "<tunnel-uuid>"
cloudflared_hostname: "n8n.<domain>"
cloudflared_hello_world_enabled: true
cloudflared_n8n_webhook_enabled: false
cloudflared_n8n_webhook_test_enabled: false
cloudflared_n8n_webhook_waiting_enabled: false
cloudflared_n8n_oauth_callback_enabled: false
```

Run Ansible once with `cloudflared_enable_service: false`. This installs the
daemon, creates the `cloudflared` group, renders config, and keeps the service
stopped.

### 5. Copy tunnel credentials to the VPS

After the first Ansible pass creates `/etc/cloudflared` and the `cloudflared`
group, copy the tunnel credentials JSON over the Tailscale/SSH path:

```bash
scp ~/.cloudflared/<tunnel-uuid>.json <admin_user>@<tailscale-host>:/tmp/<tunnel-uuid>.json
ssh <admin_user>@<tailscale-host> \
  'sudo install -o root -g cloudflared -m 0640 /tmp/<tunnel-uuid>.json /etc/cloudflared/<tunnel-uuid>.json && sudo rm /tmp/<tunnel-uuid>.json'
```

Use the real `admin_user` and Tailscale hostname/IP from ignored local
inventory.

### 6. Enable a hello-world tunnel first

Set:

```yaml
cloudflared_enable_service: true
cloudflared_hello_world_enabled: true
cloudflared_n8n_webhook_enabled: false
```

Rerun Ansible and validate `https://n8n.<domain>` reaches Cloudflare's
hello-world service. Then switch to the n8n webhook route.

### 7. Switch to n8n webhook steady state

Start with:

```yaml
cloudflared_hello_world_enabled: false
cloudflared_n8n_webhook_enabled: true
cloudflared_n8n_webhook_test_enabled: false
cloudflared_n8n_oauth_callback_enabled: false
cloudflared_n8n_webhook_waiting_enabled: false
```

## Validation

Dry-run ingress routing:

```bash
cloudflared tunnel ingress rule https://n8n.<domain>/webhook/abc
cloudflared tunnel ingress rule https://n8n.<domain>/webhook-test/abc
cloudflared tunnel ingress rule https://n8n.<domain>/webhook-waiting/abc
cloudflared tunnel ingress rule "https://n8n.<domain>/rest/oauth2-credential/callback?code=x&state=y"
```

Steady state should route only `/webhook/...` to n8n. Everything else should
hit `http_status:404` unless a temporary onboarding route is deliberately
enabled.

## Manual Cloudflare WAF Rate Limit

The tunnel role does not manage Cloudflare WAF rules. Configure this manually
in the Cloudflare dashboard so edge policy stays visible to the account owner
and is not hidden in the VPS Ansible run.

Recommended starting rule:

- Product area: WAF rate limiting rules.
- Scope: the zone that owns `cloudflared_hostname`.
- Expression: `starts_with(http.request.uri.path, "/webhook/")`
- Counting characteristic: IP.
- Threshold: 60 requests per 60 seconds.
- Action: Managed Challenge for the first rollout; switch to Block after
  observing legitimate webhook traffic.
- Mitigation timeout: 10 minutes.

Keep the rule scoped to `/webhook/`. Do not rate-limit `/webhook-test/`,
`/webhook-waiting/`, `/rest/*`, or the editor/API in this public hostname
pattern because those routes should normally stay closed at tunnel ingress.
