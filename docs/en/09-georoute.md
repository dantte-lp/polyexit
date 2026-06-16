# 09 — georoute

A single Go binary that:

1. Fetches a country's prefix list from [RIPE Stat country-resource-list](https://stat.ripe.net/docs/data_api).
2. Aggregates adjacent / overlapping prefixes.
3. Atomically replaces the contents of `nft inet pbr <cc>_v4` and `<cc>_v6` via `nft -f -`.
4. Splices `network X/Y route-map MARK-<CC>-EXIT` lines into `/etc/frr/frr.conf` between marker comments.
5. Runs `frr-reload.py --reload` only when the rendered block changes.

Source: [github.com/dantte-lp/georoute](https://github.com/dantte-lp/georoute) (separate public repo).

## CLI flags

```text
-country string         ISO-3166 alpha-2 country code (RU, UZ, KZ, …) (default "RU")
-route-map string       FRR route-map name (default MARK-<CC>-EXIT)
-nft-set-v4 string      nftables v4 set name (default <cc>_v4)
-nft-set-v6 string      nftables v6 set name (default <cc>_v6)
-marker-prefix string   marker comment prefix (default <CC>-FEED)
-feed-url string        RIPE Stat URL (default country-resource-list for <cc>)
-lock-file string       exclusive flock path (default /run/georoute-<cc>.lock)
-frr-conf string        path to FRR config (default /etc/frr/frr.conf)
-reload                 run frr-reload on change (default true)
-nft                    atomically replace nft set inet pbr {<cc>_v4,<cc>_v6} (default true)
-dry-run                print summary without writing
-force                  force write even if unchanged
```

The defaults for the five `-route-map / -nft-set-* / -marker-prefix / -feed-url / -lock-file` flags are all derived from `-country`, so for any country a single argument is enough:

```bash
georoute --country=UZ --dry-run
georoute --country=KZ
```

## Configuration — `/etc/georoute/<cc>.env`

The Ansible `georoute` role writes one env file per country owned by the host. Example (`/etc/georoute/ru.env`):

```ini
GEOROUTE_COUNTRY=RU
GEOROUTE_FEED_URL=https://stat.ripe.net/data/country-resource-list/data.json?resource=RU&v4_format=prefix
GEOROUTE_ROUTE_MAP=MARK-RU-EXIT
GEOROUTE_NFT_SET_V4=ru_v4
GEOROUTE_NFT_SET_V6=ru_v6
GEOROUTE_MARKER_PREFIX=RU-FEED
GEOROUTE_FRR_CONF=/etc/frr/frr.conf
```

The `georoute@.service` template references it as `EnvironmentFile=`.

## Systemd units

```text
/etc/systemd/system/georoute@.service     # template service
/etc/systemd/system/georoute@.timer       # template timer
```

Enable per country:

```bash
systemctl enable --now georoute@ru.timer
systemctl list-timers georoute@*.timer
```

Cadence: `OnBootSec=5min`, `OnUnitActiveSec=12h`, `RandomizedDelaySec=30min`. RIPE Stat is rate-tolerant at this frequency.

## Safety properties

| Property                                           | Mechanism                                                                                  |
|----------------------------------------------------|--------------------------------------------------------------------------------------------|
| Two concurrent runs cannot race on `frr.conf`      | `flock` on `/run/georoute-<cc>.lock`                                                       |
| `frr.conf` write is atomic                         | `os.CreateTemp` + `rename` in the same dir (no shared `.new` suffix)                       |
| `nft` set replacement is atomic                    | single `nft -f -` invocation — kernel applies as one netlink transaction                   |
| Hostile JSON cannot OOM the host                   | `io.LimitReader(_, 32 MiB)` on response body                                                |
| Transient RIPE 503/429 does not skip a 12-h cycle  | 3 attempts with exponential backoff                                                        |
| `frr-reload.py` cannot exceed its time budget      | dedicated 3-minute child context (parent ctx is 5 min total)                                |
| Hostile env from a misbehaving caller cannot leak  | `cmd.Env = []string{"PATH=/usr/sbin:/usr/bin:/sbin:/bin"}`                                  |
| Idempotency                                        | SHA-256 hash of rendered block — `frr.conf` rewritten only on diff (unless `--force`)      |

References: [nftables wiki — atomic rule replacement](https://wiki.nftables.org/wiki-nftables/index.php/Atomic_rule_replacement), [Go `os.CreateTemp`](https://pkg.go.dev/os#CreateTemp), [`flock(2)`](https://man7.org/linux/man-pages/man2/flock.2.html).

## Operator commands

```bash
# Dry-run RU — print summary, no writes
georoute --country=RU --dry-run

# Force-write even if unchanged (post-recovery)
georoute --country=RU --force

# Verify a specific destination is covered by the set
nft get element inet pbr ru_v4 { 77.88.55.88 }

# Time of last run
journalctl -u georoute@ru.service -n 10 --no-pager

# Set size
nft list set inet pbr ru_v4 | grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+/[0-9]+' | wc -l
```

## What gets written to `frr.conf`

Inside `router bgp <ASN>`, between markers:

```text
 address-family ipv4 unicast
  network 100.64.4.0/24
  network 10.255.0.4/32
  ! BEGIN-RU-FEED-V4
  network 2.56.24.0/22 route-map MARK-RU-EXIT
  network 2.56.88.0/22 route-map MARK-RU-EXIT
  network 2.56.180.0/22 route-map MARK-RU-EXIT
  …  (~8 600 lines for RU)
  ! END-RU-FEED-V4
 exit-address-family
```

The markers **must** be added to `frr.conf` by hand once — `georoute` only writes between them, never inserts them. Initial bootstrap:

```text
  ! BEGIN-RU-FEED-V4
  ! END-RU-FEED-V4
```

Same for `-V6`.

## When the feed changes shape

RIPE Stat occasionally adds or removes prefixes from a country's delegated list. `georoute` re-runs every 12 h pick up these changes automatically. The exit IP for affected destinations changes at the next run boundary; existing TCP sessions on those destinations may break and need to reconnect.

## What to read next

- [Troubleshooting](10-troubleshooting.md) — what to do when geo-routing seems wrong.
- [References](11-references.md) — primary docs for `nft`, `FRR`, `RIPE Stat`.
