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

## Bootstrap shape

Use a locally managed tunnel:

```bash
cloudflared tunnel login
cloudflared tunnel create hermes-n8n
cloudflared tunnel route dns hermes-n8n n8n.<domain>
```

Copy the generated credentials JSON to the VPS path configured by
`cloudflared_credentials_file`, then enable this role.

Start with:

```yaml
cloudflared_hello_world_enabled: true
cloudflared_n8n_webhook_enabled: false
```

Then switch to steady state:

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
