# 08 — User management

VPN users are managed by `ocserv` via the `ocpasswd` utility. The fleet keeps every host's `/etc/ocserv/ocpasswd` in sync by **running `ocpasswd` on each host through Ansible**, not by rsyncing a single golden file. Each host generates its own salt for the same password — the resulting files differ byte-for-byte, but `ocserv` validates against any of them.

## Wrapper: `bin/vpn-user`

```bash
bin/vpn-user add <user>                  # prompts for password (read -s)
bin/vpn-user add <user> <password>       # one-shot, watch your shell history
bin/vpn-user del <user>
bin/vpn-user list
```

Source: [`bin/vpn-user`](../../bin/vpn-user)

The wrapper is a thin shell over `ansible-playbook playbooks/manage-user.yml -e vpn_action=...`. Use the playbook directly if you need extra Ansible flags:

```bash
ansible-playbook playbooks/manage-user.yml \
    -e vpn_action=add \
    -e vpn_user=alice \
    -e vpn_pass='generated-password-here' \
    --limit exits-country-local
```

## Examples

```bash
# Add a user, interactive password
$ bin/vpn-user add alice
Password for alice: ********
Confirm password:      ********

# Add with explicit password (e.g. when scripted)
$ bin/vpn-user add bob 'OneTimeP@ss12'

# Remove
$ bin/vpn-user del bob

# Audit — same user list on every host?
$ bin/vpn-user list
…
ok: [dev-03] => msg: 'dev-03 (11 users): alice, bmv, …'
ok: [dev-04] => msg: 'dev-04 (11 users): alice, bmv, …'
```

If the lists diverge, that is a bug — find which host is stale and re-run the add for the missing user.

## Password generation

```bash
# 20-character base64 alphabet, no `/+=`
openssl rand -base64 18 | tr -d '/+=' | head -c 20
```

Capture the output to your password manager **before** invoking `bin/vpn-user add` — once the playbook is past the `no_log: true` task the value is no longer in any log.

## Behavior on success

- Every reachable host gets the same user added.
- Handler `occtl reload` fires on every changed host — **graceful**; active sessions are not terminated, and their pushed config does not change. New session = new push.
- Placeholder hosts (e.g. `dev-05` with `ansible_host: "{{ public_v4 }}"` where `public_v4: TBD`) report `unreachable` — that is expected and not a failure to act on.

## Mass user import

```bash
while IFS=: read -r user pass; do
    bin/vpn-user add "$user" "$pass"
done < users.tsv
```

Format `users.tsv`: `user:password` per line. Watch out for shell history — `set +o history` before the loop.

## Authentication model

This playbook uses ocserv's `plain` backend (`/etc/ocserv/ocpasswd`). Alternatives that work with the same fleet but require swapping the `ocserv.conf` template:

| Backend             | When to use                                                                  |
|---------------------|------------------------------------------------------------------------------|
| `plain` (default)   | small fleets, no IdP                                                          |
| `pam`               | sync with system users; integrate with FreeIPA / SSSD                         |
| `radius`            | corporate RADIUS server (FreeRADIUS, NPS)                                     |
| `oidc`              | newer ocserv builds; SSO via Keycloak / Authentik                             |

Reference: [`ocserv.8` — Authentication section](https://ocserv.openconnect-vpn.net/ocserv.8.html#sect3).

## What to read next

- [georoute](09-georoute.md) — operator's view of the prefix feed.
- [Troubleshooting](10-troubleshooting.md) — when login or routing breaks.
