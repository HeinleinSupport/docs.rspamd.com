---
title: DCC module
---

# DCC module

This module performs [DCC](https://www.dcc-servers.net/dcc/) (Distributed Checksum Clearinghouses) lookups to determine the *bulkiness* of a message based on how many recipients have seen similar content.

DCC uses fuzzy checksums to identify bulk mail. The bulkiness information is useful in composite rules. For example, if a message is from a freemail domain and is reported as bulk by DCC, it is likely spam and can be assigned a higher score.

**Important:** Before enabling this module, please review the [DCC License terms](https://www.dcc-servers.net/dcc/).

## Symbols

| Symbol | Score | Description |
|--------|-------|-------------|
| `DCC_REJECT` | 2.0 | DCC returned reject result |
| `DCC_BULK` | dynamic (base 1.0) | Message identified as bulk based on thresholds |
| `DCC_FAIL` | 0.0 | DCC check failed |

## Prerequisites

You must have the `dccifd` daemon installed and running:

1. Download and build the [DCC client](https://www.dcc-servers.net/dcc/source/dcc.tar.Z)
2. Configure `/var/dcc/dcc_conf`:
   ```
   DCCIFD_ENABLE=on
   DCCM_LOG_AT=NEVER
   DCCM_REJECT_AT=MANY
   ```
3. Start the daemon: `/var/dcc/libexec/rcDCC start`

By default, `dccifd` listens on Unix socket `/var/dcc/dccifd`.

## Configuration

Settings go in `/etc/rspamd/local.d/dcc.conf`.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `servers` | string | (required) | Socket path or TCP servers (e.g., `/var/dcc/dccifd` or `127.0.0.1:10045`) |
| `socket` | string | - | Alias for `servers` |
| `timeout` | number | `5.0` | Connection timeout in seconds |
| `retransmits` | number | `2` | Number of retry attempts |
| `default_port` | number | `10045` | Default TCP port |
| `body_max` | number | `999999` | Bulkiness threshold for body checksum |
| `fuz1_max` | number | `999999` | Bulkiness threshold for fuz1 checksum |
| `fuz2_max` | number | `999999` | Bulkiness threshold for fuz2 checksum |
| `default_score` | number | `1` | Base score multiplier used for `DCC_BULK` dynamic scoring |
| `symbol` | string | `DCC_REJECT` | Symbol for reject result |
| `symbol_bulk` | string | `DCC_BULK` | Symbol for bulk detection |
| `symbol_fail` | string | `DCC_FAIL` | Symbol for check failure |
| `message` | string | `${SCANNER}: bulk message found: "${VIRUS}"` | Message template for bulk detection results |
| `detection_category` | string | `hash` | Detection category reported to the scanner framework |
| `log_clean` | boolean | `false` | Log clean (non-bulk) results |
| `client` | string | `0.0.0.0` | Default client IP if not available |
| `cache_expire` | number | `7200` | Redis cache expiration (seconds) |
| `prefix` | string | `rs_dcc_` | Redis cache key prefix |

The module is activated by having a `dcc { }` configuration section; there is no separate `enabled` flag.

**Deprecated options:** `host` and `port` are accepted for backwards compatibility but emit a warning at startup. Use `socket` (Unix path or single host) or `servers` (TCP upstreams) instead.

## Example configuration

### Unix socket (local dccifd)

~~~hcl
# local.d/dcc.conf

servers = "/var/dcc/dccifd";

# Thresholds for bulk detection
body_max = 999999;
fuz1_max = 999999;
fuz2_max = 999999;
~~~

### TCP connection (remote or local)

~~~hcl
# local.d/dcc.conf

servers = "127.0.0.1:10045";
timeout = 5.0;
retransmits = 2;
~~~

### Custom thresholds

Lower thresholds trigger bulk detection more easily:

~~~hcl
# local.d/dcc.conf

servers = "/var/dcc/dccifd";

# Trigger bulk detection at lower counts
body_max = 100;
fuz1_max = 100;
fuz2_max = 100;

# Custom base score multiplier
default_score = 2.0;
~~~

## TCP configuration for dccifd

To configure `dccifd` to listen on TCP instead of Unix socket, edit `/var/dcc/dcc_conf`:

```
DCCIFD_ARGS="-SHELO -Smail_host -SSender -SList-ID -p *,10045,127.0.0.0/8"
```

This configures dccifd to:
- Listen on all interfaces, port 10045
- Accept connections from 127.0.0.0/8

## How scoring works

`DCC_REJECT` carries a fixed metric score of **2.0** and fires when DCC returns a hard reject (`R` result).

`DCC_BULK` fires for accepted messages where one or more checksum thresholds are met (`A`/`S` result). Its metric base score is **1.0**, but the actual per-message score is computed dynamically:

1. **Reputation (rep)**: DCC can report a reputation percentage (0–100%). When absent, rep defaults to 100%.
2. **Checksum counts**: body, fuz1, and fuz2 values are each compared against their respective `*_max` thresholds.
3. **Score formula**: for each threshold exceeded, the contribution is `default_score * (rep / 100) / 3`. The contributions are summed.

The symbol options on `DCC_BULK` record which thresholds were exceeded and the reputation value (e.g. `body=many fuz1=42 rep=75%`).

## Using in composites

Example composite rule combining DCC with other checks:

~~~hcl
# local.d/composites.conf

DCC_FREEMAIL_BULK {
  expression = "DCC_BULK & FREEMAIL_FROM";
  score = 5.0;
  description = "Bulk message from freemail";
}
~~~
