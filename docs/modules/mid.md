---
title: MID module
---


# MID module

The purpose of the MID module is to suppress the `INVALID_MSGID` (malformed Message-ID header) and `MISSING_MID` (missing Message-ID) rules for messages which are DKIM-signed by some particular domains. The module is disabled by default and requires a `source` map to be configured before it becomes active.

## Configuration

The module requires a `source` parameter pointing to a map file. Without it the module logs a warning and disables itself at startup.

### Configuration options

| Option | Default | Description |
|--------|---------|-------------|
| `source` | required | Map of domains and optional Message-ID patterns |
| `symbol_known_mid` | `KNOWN_MID` | Symbol for known Message-ID pattern match |
| `symbol_known_no_mid` | `KNOWN_NO_MID` | Symbol for known missing Message-ID |
| `symbol_invalid_msgid` | `INVALID_MSGID` | Symbol to suppress for invalid Message-ID |
| `symbol_missing_mid` | `MISSING_MID` | Symbol to suppress for missing Message-ID |
| `symbol_dkim_allow` | `R_DKIM_ALLOW` | DKIM allow symbol to check |
| `csymbol_invalid_msgid_allowed` | `INVALID_MSGID_ALLOWED` | Composite symbol for allowed invalid Message-ID |
| `csymbol_missing_mid_allowed` | `MISSING_MID_ALLOWED` | Composite symbol for allowed missing Message-ID |

### Example configuration

~~~hcl
mid = {
  source = "${CONFDIR}/mid.inc";
}
~~~

The `source` setting points to a map to check DKIM signatures (& optionally message-ids) against, formatted as follows:

~~~
example.com /^[a-f0-9]{8}(?:-[a-f0-9]{4}){3}-[a-f0-9]{12}-0$/
example.net
~~~

With this configuration, when a message is DKIM-signed by `example.net` (no Message-ID pattern required), `KNOWN_NO_MID` is inserted. When signed by `example.com` and the Message-ID matches the regex, `KNOWN_MID` is inserted instead.

The module then registers two composite symbols that trigger alongside the respective base symbols:

- `INVALID_MSGID_ALLOWED` fires when both `KNOWN_MID` and `INVALID_MSGID` are present.
- `MISSING_MID_ALLOWED` fires when both `KNOWN_NO_MID` and `MISSING_MID` are present.

By assigning a sufficiently negative score to these composite symbols in your `scores` configuration, the positive score of `INVALID_MSGID` or `MISSING_MID` is counteracted. The base symbols themselves are not removed — the net score is reduced to a neutral level via the composite.
