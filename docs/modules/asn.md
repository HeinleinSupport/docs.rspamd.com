---
title: ASN module
---

# ASN module

The ASN module retrieves Autonomous System Number (ASN) information and related data for the sender's IP address. This includes the ASN, country code of the ASN owner, and the IP's announced subnet (network prefix).

The retrieved information is stored as mempool variables and made available to other plugins and modules for use in filtering rules.

## How it works

The module performs DNS TXT lookups against a DNSBL-style service. The IP octets are reversed before the lookup, so for an IP address like `1.2.3.4` the query is `4.3.2.1.asn.rspamd.com`. For example, querying `1.2.3.4` receives a response like:

```
1234 | 1.2.3.0/24 | DE | ripe |
```

This is parsed to extract:
- **ASN**: `1234` (the AS number)
- **IP Network**: `1.2.3.0/24` (the announced subnet)
- **Country**: `DE` (country code)

## Exported variables

The module exports the following mempool variables, available to Lua plugins after the prefilters stage:

| Variable | Description | Example |
|----------|-------------|---------|
| `asn` | Autonomous System Number | `15169` |
| `ipnet` | Announced IP network/subnet | `8.8.8.0/24` |
| `country` | Country code of ASN owner | `US` |

## Symbols

| Symbol | Score | Description |
|--------|-------|-------------|
| `ASN` | 0.0 | Informational symbol with ASN lookup results |
| `ASN_FAIL` | 0.0 | DNS lookup failed |

## Configuration

The ASN module is enabled by default. Settings can be added to `/etc/rspamd/local.d/asn.conf`.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `provider_type` | string | `rspamd` | Provider type (currently only `rspamd` is supported) |
| `provider_info` | object | (see below) | Provider-specific settings |
| `symbol` | string | `ASN` | Symbol to insert with lookup results |
| `check_local` | boolean | `false` | Perform lookups for local/private IP addresses |
| `check_authed` | boolean | `true` | Perform lookups for authenticated users |
| `symbol_fail` | string | `ASN_FAIL` | Symbol to insert when the DNS lookup fails |

### Provider info defaults

```hcl
provider_info {
  ip4 = "asn.rspamd.com";
  ip6 = "asn6.rspamd.com";
}
```

## Example configuration

~~~hcl
# local.d/asn.conf

# Provider type (only "rspamd" currently supported)
provider_type = "rspamd";

# DNS servers for lookups
provider_info {
  ip4 = "asn.rspamd.com";
  ip6 = "asn6.rspamd.com";
}

# Symbol name for results
symbol = "ASN";

# Skip lookups for local IPs (default)
check_local = false;
~~~

## Using ASN data in Lua

The ASN data can be accessed from Lua plugins after the prefilters stage:

~~~lua
local function my_callback(task)
  local asn = task:get_mempool():get_variable('asn')
  local country = task:get_mempool():get_variable('country')
  local ipnet = task:get_mempool():get_variable('ipnet')
  
  if asn then
    rspamd_logger.infox(task, 'ASN: %s, Country: %s, Network: %s', 
      asn, country or 'unknown', ipnet or 'unknown')
  end
end
~~~

## Using ASN data in multimap

The ASN data can be used with the [multimap](/modules/multimap) module:

~~~hcl
# local.d/multimap.conf

# Check ASN against a list
ASN_BLACKLIST {
  type = "asn";
  map = "/etc/rspamd/asn_blacklist.map";
  score = 5.0;
  description = "ASN is in blacklist";
}

# Check country code
COUNTRY_BLACKLIST {
  type = "country";
  map = "/etc/rspamd/country_blacklist.map";
  score = 2.0;
  description = "Sender country is blacklisted";
}
~~~
