# Security Policy

## Reporting a vulnerability

**Do not open a public issue for security bugs.** Use one of these private
channels:

1. **Preferred:** open a [private security advisory](https://github.com/dantte-lp/polyexit/security/advisories/new)
   on GitHub. Confidential thread + private patch + CVE when applicable.
2. Alternative: email `admin@lavrukhin.net` (encrypted mail preferred —
   request the PGP key first).

Include in your report:

1. A short description of the issue.
2. Steps to reproduce, ideally a minimal proof of concept.
3. The impact you believe it has.
4. The commit / branch you tested against.

Acknowledged within **3 business days**; remediation plan within
**10 business days** for high-severity issues.

## Scope

In scope:

- Issues in the Ansible playbooks, roles, templates, or `bin/vpn-user`
  wrapper that an attacker can exploit with access to the controller
  (e.g. tampered inventory leading to credential leak, role injection
  through host_vars, secret exposure in `ansible-playbook -v` output).
- Privilege-escalation paths through the deployed systemd units, nftables
  rules, or `connect-vrf.sh` hook.
- Cross-site or cross-VRF traffic leaks introduced by templated
  configurations.

Out of scope:

- Bugs in upstream components (`ocserv`, `FRR`, `nftables`, `firewalld`,
  WireGuard, the Linux kernel, Pi-hole). Report those upstream; we will
  mirror confirmed advisories.
- Misconfiguration on the operator's hosts (missing firewall rules,
  unsuitable kernel version, weak SSH keys).
- The example `vault.yml.example` containing placeholder strings —
  encrypt before commit.

## Secrets in the repository

This repository contains:

- `inventory/group_vars/vault.yml.example` — a template with `REPLACE_ME`
  placeholders only.
- Public IP addresses of the production fleet — these are already public
  via DNS A/AAAA records for `news.infra4.dev` and `docs.infra4.dev`.

This repository does **not** contain:

- WireGuard keys or PSKs (generated on each host, never committed).
- Cloudflare API tokens, ocserv `ocpasswd`, or BGP MD5 secrets.
- TLS private keys.

If you find anything that looks like a real secret in the git history,
report it via the private channels above so it can be revoked.

## Supported versions

The `main` branch is the only supported branch. Pin specific commits or
tags for reproducibility.
