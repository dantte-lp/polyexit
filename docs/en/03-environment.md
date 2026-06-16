# 03 — Environment

## Host OS

**Oracle Linux 10 (OL10) with UEK Release 8** (kernel `6.12.x-uekR8`).

- OL10 is binary-compatible with RHEL10 and shares its package universe (RPM via `dnf`). Reference: [Oracle Linux 10 release notes](https://docs.oracle.com/en/operating-systems/oracle-linux/10/relnotes10/).
- UEK R8 ships kernel `6.12-LTS`, which we need for:
  - First-class VRF (`l3mdev`) data path. Reference: [Linux VRF doc](https://docs.kernel.org/networking/vrf.html).
  - `nft` `route` hook on output (used by `pbr.nft`).
  - `tcp_l3mdev_accept` / `udp_l3mdev_accept` covering both v4 and v6.

Any modern EL10-compatible distribution (Rocky 10, Alma 10, RHEL 10) works the same; the playbook does not depend on Oracle-specific packages.

## Architecture

`x86_64` only — `firewalld` and `nftables` modules are built and tested on this arch. The Ansible inventory does not gate on `ansible_architecture`, but only `amd64` is tried in production.

## Kernel modules required

| Module        | Why                                                                     |
|---------------|-------------------------------------------------------------------------|
| `vrf`         | The `vrf-vpn` master device — comes from `kernel-uek-modules-extra`.    |
| `nf_tables`   | All `nft` rules use the `inet` family — built into base kernel.         |
| `nf_nat`      | `firewalld` and `vpn_dnat` DNAT/SNAT — built-in.                        |
| `xt_TCPMSS`   | MSS clamp — provided by `kernel-uek-modules-extra-netfilter`.           |

Persistence: `/etc/modules-load.d/vrf.conf` (deployed by the `vrf-vpn` role) writes `vrf\n` so `vrf` loads on every boot.

## Package list (top-level `base_packages`)

Defined in [`inventory/group_vars/all.yml`](../../inventory/group_vars/all.yml). Resolved with `dnf install`:

```yaml
base_packages:
  - nftables
  - firewalld
  - iproute
  - iproute-tc
  - socat
  - bind-utils
  - tcpdump
  - conntrack-tools
  - jq
  - git
  - vim
  - bash-completion
  - chrony
  - kernel-uek-modules-extra
```

Per-role packages added on demand:

| Role         | Extra packages                                          | Why                                                         |
|--------------|---------------------------------------------------------|-------------------------------------------------------------|
| `ocserv`     | `ocserv`                                                | OpenConnect SSL VPN server. [Project page](https://ocserv.gitlab.io/www/index.html). |
| `secure-dns` | `podman`, `podman-compose`, `python3-pip`               | Pi-hole + dnscrypt stack. Pi-hole image from `docker.io/pihole/pihole:latest`. |
| `georoute`   | none on target — Go binary cross-built on the controller | Avoids shipping a Go toolchain to every exit node.        |

## Why these specifically

| Tool                        | Why this and not the alternative                                                       |
|-----------------------------|----------------------------------------------------------------------------------------|
| `ocserv` (not `OpenVPN`)    | CSTP+DTLS, fast handshake, plays well with corporate networks; long-standing project. |
| `FRR` (not `BIRD`)          | More widely deployed in enterprise; `frr-reload.py` allows incremental hot reload.   |
| `nftables` (not `iptables`) | Atomic transactions; single ruleset across v4+v6 via `inet` family.                  |
| `firewalld` (not raw `nft`) | Zone abstraction; easy `--add-interface` for new tunnels.                            |
| `WireGuard` (not `IPsec`)   | Lower kernel attack surface; deterministic key exchange; single-file config.         |
| `podman` (not `docker`)     | Rootless option; no daemon; systemd-friendly; preinstalled on EL10.                  |
| `Pi-hole` (optional)        | Off-the-shelf filtering UI; replaceable with `unbound` for pure recursion.           |
| `Ansible` (not `Saltstack`) | Push-only; SSH-only; no extra agent on targets; YAML-only.                           |

## Disk usage budget

```text
/                       ~5 GB     base OL10 + packages
/var/lib/containers     ~1 GB     pihole image + persistent volumes (if secure-dns)
/etc                    ~20 MB
/var/log                grows — journald compaction at /etc/systemd/journald.conf default
```

A `30 GB` root partition is comfortable.

## Network requirements per host

| Direction | Port           | Purpose                                                |
|-----------|----------------|--------------------------------------------------------|
| inbound   | TCP/22         | SSH for Ansible                                        |
| inbound   | TCP/443        | ocserv CSTP control + data fallback                    |
| inbound   | UDP/443        | ocserv DTLS — optional (some DPI blocks this)          |
| inbound   | UDP/<wg-port>  | WireGuard between siblings (default 31518; per-host configurable) |
| outbound  | TCP/443        | Letsencrypt + RIPE Stat for georoute                   |
| outbound  | UDP/53         | DNS resolution (for ACME challenge)                    |

The Ansible roles do not open these ports themselves — they assume `firewalld` zones are pre-configured (see [`roles/common`](../../roles/common/tasks/main.yml)).

## Time

`chrony` is installed by the `common` role but its `chrony.conf` is **not** managed. Recommended config: pin pools close to your geography. RU example:

```ini
pool 0.ru.pool.ntp.org iburst
pool time.cloudflare.com iburst
```

NTP drift breaks DTLS key rotation and TLS cert validation — keep it tight.

## SELinux / AppArmor

Both **disabled** on the production hosts. The Ansible playbook does not touch SELinux state. If you enable `enforcing`:

- `ocserv` needs `setsebool ocserv_use_userspace_libraries on` and a label for `/etc/ocserv/connect-vrf.sh`.
- `podman` on EL10 generally works with the default policy.
- `nft` and `firewalld` are untouched by SELinux.

## What to read next

- [Inventory](04-inventory.md) — modeling a host.
- [Deployment](06-deployment.md) — first apply.
