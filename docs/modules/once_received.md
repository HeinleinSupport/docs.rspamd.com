---
title: Once received module
---

# Once received module

This module is intended to do simple checks for mail with one `Received` header. The underlying concept is that genuine emails tend to have multiple received headers, whereas spam originating from compromised user devices often exhibit certain negative characteristics, such as the use of `dynamic` or `broadband` IP addresses.

## Configuration

The module is **disabled by default** and does nothing unless `symbol` is explicitly set in configuration — all other options are only read when `symbol` is present. To enable it, define a `symbol` for generic emails with only one received header, optionally specify a `symbol_strict` for emails that exhibit negative patterns or have unresolved hostnames, and include **good** and **bad** patterns, which can utilise [lua patterns](http://lua-users.org/wiki/PatternsTutorial). Use `good_host` to exclude certain hosts from this module, and `bad_host` to identify specific negative patterns. Additionally, you can create a `whitelist` to define a list of networks for which the checks should be excluded.

### Configuration options

| Option | Default | Description |
|--------|---------|-------------|
| `symbol` | (none — module is inactive without this) | Symbol for messages with only one received header; **must be set to enable the module** |
| `symbol_strict` | nil | Symbol for messages matching bad patterns or unresolved hostnames |
| `symbol_mx` | `DIRECT_TO_MX` | Symbol for direct MUA to MX connections (detected via User-Agent/X-Mailer) |
| `good_host` | - | Lua pattern for hostnames to exclude from checks |
| `bad_host` | - | Lua pattern for hostnames that trigger strict symbol |
| `whitelist` | nil | Map of IP addresses/networks to exclude from checks |
| `check_local` | false | Apply checks to messages from local networks |
| `check_authed` | false | Apply checks to messages from authenticated users |

## Example

~~~hcl
once_received {
    symbol = "ONCE_RECEIVED";
    symbol_strict = "ONCE_RECEIVED_STRICT";
    good_host = "^mail";
    bad_host = ["static", "dynamic"];
    whitelist = "/tmp/ip.map";
}
~~~

As is typical, the IP map can include both IPv4 and IPv6 addresses, as well as networks in CIDR notation. You may also add optional comments to the map, indicated by a `#` symbol.
