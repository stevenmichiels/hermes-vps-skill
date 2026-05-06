# Security Policy

This repository is a sanitized deployment template. It does not provide a production-hardening guarantee, and operators are responsible for reviewing Terraform plans, Ansible changes, firewall exposure, runtime configuration, and secret handling before use.

## Supported Versions

Security fixes are expected on the default branch and the latest tagged release, once releases exist. Older commits and forks are not actively supported.

## Reporting a Vulnerability

Prefer a private contact method listed on the maintainer's GitHub profile for vulnerability reports. If no private contact is available, open a GitHub issue with only non-sensitive reproduction details and ask for a private channel.

Expected initial response time: best effort within 7 days.

Do not include secrets, tokens, private IPs, account IDs, hostnames, chat IDs, user IDs, OAuth payloads, Terraform state, Ansible inventory, or live infrastructure details in public reports.

## Operator Responsibilities

Before using this template, operators should review:

- Terraform plans and provider lockfile changes.
- Hetzner firewall exposure and UFW rules.
- SSH bootstrap CIDRs and Tailscale access controls.
- Remote install script tasks in Ansible roles.
- Docker image tags, especially optional Firecrawl images that default to upstream `latest`.
- Runtime env files and backup handling.
