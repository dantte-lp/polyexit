# 10 — Troubleshooting

## Methodology

> Server-local tests **do not** validate tunnel paths. Always test from a real VPN client (or simulate one in a netns).

The rule of thumb:

```text
SYMPTOM: "feature X is broken for VPN client"
WRONG  : `ping`/`curl` from the host's shell — uses default-VRF routing
RIGHT  : test from the connected client (or `ip vrf exec vrf-vpn …` with the right src)
```

## Symptom matrix

| Symptom                                              | First check                                                              | Likely cause                                                                  |
|------------------------------------------------------|--------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| Client connects but no traffic egresses              | `nft list table inet firewalld | grep masquerade`                        | Wrong oif zone — `vpns* → wg*` is `trusted→trusted`, no masquerade fires      |
| RU prefix exits via FI                               | `nft get element inet pbr ru_v4 { <dst> }`                               | `ru_v4` set is empty — run `systemctl start georoute@ru.service`              |
| Reply to a sibling's client black-holes              | `ip route show <sibling pool>`                                           | Sibling pool route missing — `route-map BGP-MAIN-FIB` filter; ensure static    |
| v6 ping6 to a destination times out (NAT'd path)     | `ip -6 rule | grep "lookup 50"`                                          | `from <own-ipv6> lookup 50` matches reply too; add return route in table 50   |
| Pi-hole DNS times out (UDP/53)                       | `nft list table inet vpn_dnat | grep counter`                            | Counter == 0: DNAT does not match — check iif rule (`vpns*` and `vrf-vpn`)    |
| `vpns0` shows in `occtl` but client gets gateway-only RX | `journalctl -t ocserv-vrf -n 10`                                      | connect-script not running — `getsebool ocserv_use_userspace_libraries`        |
| WG handshake never completes                         | `wg show wg0 | grep handshake`                                            | Peer `endpoint` IP changed; PSK mismatch; UDP port blocked                    |
| BGP peer flaps                                       | `vtysh -c 'show bgp neighbor <ip>'`                                      | MD5 mismatch; ASN typo; `address-family` not activated on neighbor           |

## Live diagnostics

### 1. Where does a packet die?

```bash
# add a one-shot trace for one 5-tuple
nft 'add rule inet filter input ip daddr 1.1.1.1 meta nftrace set 1'
nft monitor trace                    # in another shell, while you reproduce

# or, broader — every drop reason
perf trace -e skb:kfree_skb
```

References: [nftables ruleset debug](https://wiki.nftables.org/wiki-nftables/index.php/Ruleset_debug/tracing); [SKB drop reasons enum](https://dxuuu.xyz/dropreason.html).

### 2. Cross-VRF routing decision

```bash
# what would the kernel do for a packet from client → 8.8.8.8?
ip route get 8.8.8.8 from 100.64.4.50 iif vrf-vpn

# v6 equivalent
ip -6 route get 2606:4700:4700::1111 from fdf3:bb42:9fc6:4::dead iif vrf-vpn
```

If the destination should reach via wg0 but the answer says `r64stk`, your table-10 default is wrong — re-apply the `vrf-vpn` role.

### 3. Is the prefix in the set?

```bash
nft get element inet pbr ru_v4 { 77.88.55.88 }
# expected:
# table inet pbr { set ru_v4 { type ipv4_addr; flags interval; elements = { 77.88.0.0/18 } } }
```

If error `No such file or directory` — the IP is not covered by any set element. Either the country feed legitimately does not include it, or `ru_v4` has been flushed (re-run `georoute --force`).

### 4. Did `georoute` run recently?

```bash
systemctl list-timers georoute@*.timer
journalctl -u georoute@ru.service -n 20 --no-pager
```

Look for `nft sets updated (ru_v4=8621 ru_v6=2186)` and `frr-reload completed` (or `frr.conf unchanged — skipping reload`).

### 5. Does the firewalld zone resolve as expected?

```bash
# what zone does my interface live in?
firewall-cmd --get-zone-of-interface=vpns0
firewall-cmd --get-zone-of-interface=wg0
firewall-cmd --get-zone-of-interface=podman4

# any rich-rules I forgot?
firewall-cmd --zone=trusted --list-rich-rules
firewall-cmd --zone=public  --list-rich-rules
```

Wildcard interfaces (`vpns+`, `vpns*`) do not show up via `--get-zone-of-interface` — they only appear in `--list-interfaces`.

### 6. WireGuard health

```bash
wg show wg0
# - handshake should be < 3 min ago
# - transfer counters growing
# - allowed-ips covers 0.0.0.0/0, ::/0
```

If handshakes never complete, packet sniff between the two endpoints:

```bash
tcpdump -nn -i ens1 udp port 31518
```

### 7. BGP

```bash
vtysh -c 'show bgp summary'
vtysh -c 'show bgp ipv4 unicast community 64512:201 | head'
vtysh -c 'show bgp neighbor fdf3:bb42:9fc6:ffff::3 advertised-routes'
ip -d route show table 10 proto bgp | head
```

`proto bgp` = `zebra (FRR)`. If the route is in BGP RIB but not in kernel, check `ip protocol bgp route-map BGP-MAIN-FIB` — it can deny installation in `main` (deliberate for country prefixes, accidental otherwise).

## Common bring-up mistakes

| Mistake                                                                 | Recovery                                                                |
|-------------------------------------------------------------------------|-------------------------------------------------------------------------|
| Forgot to add BEGIN/END markers to `frr.conf`                           | georoute will fail with `errBeginMissing` — add markers, re-run         |
| WG `Table = off` missing                                                | wg-quick installs default routes you do not want — set `Table = off`    |
| `vpns+` not in firewalld trusted                                        | replies blocked — `firewall-cmd --permanent --zone=trusted --add-interface=vpns+ && firewall-cmd --reload` |
| `connect-vrf.sh` not `chmod +x`                                         | ocserv silently skips it — clients work but VRF isolation is gone        |
| `ipv6-subnet-prefix=128` clients have no main-table v6 route             | enforced by connect-vrf.sh's `ip -6 route replace … table 10`           |
| Sibling pool not in main FIB                                            | static route from `vrf-vpn.service` `sibling_v4_pool` / `sibling_v6_pool`|

## Reverting a bad deploy

```bash
cd /opt/project/repositories/infra-config
git log --oneline -10            # find the last known-good commit
git revert <bad-commit>          # creates an inverse commit
ansible-playbook playbooks/site.yml --check --diff
ansible-playbook playbooks/site.yml
```

Manual state added outside the playbook (e.g. one-off `ip route add`) is **not** reverted by this — those need to be undone by hand or codified into the role.

## When to suspect the kernel itself

Cross-VRF behavior is fragile and has had upstream regressions:

- "vrf: Reset skb conntrack connection on VRF rcv" series — reverted because it broke DNAT for forwarded traffic.
- FRR issue [#15909](https://github.com/FRRouting/frr/issues/15909) — kernel 6.6/6.7 cross-VRF leak with TTL bug.

If your kernel is `6.6.x` or `6.7.x` and DNAT through VRF behaves erratically, upgrade to `6.8+` (or UEK R8 `6.12`).

## What to read next

- [References](11-references.md) — primary documentation for every component.
