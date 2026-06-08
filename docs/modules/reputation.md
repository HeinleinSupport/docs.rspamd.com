---
title: Reputation module
---


# Reputation plugin

This plugin is designed to monitor the reputation of various objects and adjust scores accordingly.

For instance, if you have a DKIM domain that is known to be used for spam, this module enables you to decrease the negative score of the DKIM_ALLOW symbol, or even add some score.

Conversely, if a domain has a high reputation, the DKIM_ALLOW score will have a more negative score (like auto-whitelisting) and increase the score for DKIM_REJECT accordingly (since the message looks like a phishing attempt).

Additionally, this module encompasses the functionality of the following modules:

* [ip_score](/modules/ip_score) - by means of `ip` component
* url_reputation - by means of `url` component (removed in Rspamd 2.0)

## Configuration and principles of work

Like many other modules, this module requires a set of rules to be defined. Each rule comprises the following components:

* Selector configuration - specifies the data to be extracted from a message and defines data processing logic
* Backend configuration - determines where reputational tokens should be stored and queried. For instance, Redis can be used for both storing and extracting, while DNS can only be used as a read-only storage
* Common configuration - defines generic rule parameters, such as a symbol, that are not related to either the backend or the selector

Below are a few examples of such configurations:

~~~hcl
# local.d/reputation.conf
rules {
  ip_reputation = {
    selector "ip" {
    }
    backend "redis" {
      servers = "localhost";
    }

    # The ip selector default symbol is SENDER_REP (split: SENDER_REP_HAM / SENDER_REP_SPAM).
    # Override here if you prefer a different name.
    symbol = "SENDER_REP";
  }
  spf_reputation =  {
    selector "spf" {
    }
    backend "redis" {
      servers = "localhost";
    }

    symbol = "SPF_REPUTATION";
  }
  dkim_reputation =  {
    selector "dkim" {
    }
    backend "redis" {
      servers = "localhost";
    }

    symbol = "DKIM_REPUTATION"; # Also adjusts scores for DKIM_ALLOW, DKIM_REJECT
  }
  generic_reputation =  {
    selector "generic" {
      selector = "ip"; # see https://docs.rspamd.com/configuration/selectors
    }
    backend "redis" {
      servers = "localhost";
    }

    symbol = "GENERIC_REPUTATION";
  }
}
~~~

You also need to **define the scores** for symbols added by this module:

~~~hcl
# local.d/groups.conf
group "reputation" {
    symbols = {
        # ip selector uses SENDER_REP with split_symbols=true by default
        "SENDER_REP_HAM" {
            weight = -1.0;
        }
        "SENDER_REP_SPAM" {
            weight = 4.0;
        }

        "DKIM_REPUTATION" {
            weight = 1.0;
        }

        # spf selector uses split_symbols=true by default
        "SPF_REPUTATION_HAM" {
            weight = -1.0;
        }
        "SPF_REPUTATION_SPAM" {
            weight = 2.0;
        }

        "GENERIC_REPUTATION" {
            weight = 1.0;
        }
    }
}
~~~

The weight assigned to these symbols are merely examples and you should adjust them to fit your particular situation.

The image below illustrates the process of reputation token handling:

<center><img class="img-fluid" src="/img/reputation1.png" width="50%"></center>

### Backends configuration and principles of work

Selectors provide what are known as tokens for backends. For instance, in the case of IP reputation, these tokens could be `asn`, `ipnet`, and `country`. Each token is mapped to a particular key in the backend.

#### Redis backend: EWMA algorithm

The Redis backend does **not** maintain separate ham/spam/junk counters. Instead it stores a single floating-point score per bucket using an **exponential weighted moving average** (EWMA). On each update the new per-message score is blended with the stored value using a time-decay factor:

```
alpha = 1 - exp(-time_since_last_update / window)
new_stored = alpha * message_score + (1 - alpha) * previous_stored
```

Because `alpha` is proportional to the time elapsed since the last update, recent messages have more influence than old ones. The stored value therefore represents the sender's recent average score, not a running count of message classes.

Each bucket has two configuration attributes:

* `time` — the time window in seconds used to compute the decay factor;
* `mult` — a multiplier applied to the stored value when it is read back.

By default, one bucket with a time window of 30 days is defined for Redis:

~~~hcl
buckets = [
  {
    time = 60 * 60 * 24 * 30,
    name = '1m',
    mult = 1.0,
  }
];
~~~

Upon bucket lookup the stored EWMA score is multiplied by `mult` and passed to the selector's scoring formula (a `tanh`-based normalisation against the message's reject threshold).

<center><img class="img-fluid" src="/img/reputation2.png" width="50%"></center>

## Selector types

There are couple of pre-defined selector types, specifically:

* SPF reputation - `spf` selector
* DKIM reputation - `dkim` selector
* IP, asn, country and network reputation - `ip` selector (also available as `sender`, which is a registered alias for the same implementation)
* URLs reputation - `url` selector
* Generic reputation based on [selectors framework](/configuration/selectors) - `generic` selector

All selector types except for `generic` do not require explicit configuration. The `generic` selector, on the other hand, necessitates the setting of a selector attribute. For more advanced `selector` configurations, you may refer to the module's source code.

### Common selector options

All selectors support the following common options (defaults shown are the generic fallback; individual selectors may override them — see per-selector tables below):

| Option | Default | Description |
|--------|---------|-------------|
| `lower_bound` | 10 | Minimum number of messages before reputation scoring kicks in |
| `min_score` | nil | Minimum reputation score |
| `max_score` | nil | Maximum reputation score |
| `outbound` | true | Apply reputation to outbound messages |
| `inbound` | true | Apply reputation to inbound messages |
| `split_symbols` | false | Create separate `_HAM` and `_SPAM` symbols instead of a single symbol |
| `exclusion_map` | nil | Map of items to exclude from reputation tracking (all selectors honour this; for `ip`/`sender` a radix map is used, for all others a set map) |

### IP selector specific options

The `ip` selector (also accessible as `sender`) overrides the following common defaults:

| Option | Selector default | Description |
|--------|-----------------|-------------|
| `symbol` | `SENDER_REP` | Default symbol name (split: `SENDER_REP_HAM` / `SENDER_REP_SPAM`) |
| `outbound` | **false** | Reputation is applied to inbound traffic only by default |
| `split_symbols` | **true** | `_HAM`/`_SPAM` sub-symbols are emitted by default |

Additional IP-specific options:

| Option | Default | Description |
|--------|---------|-------------|
| `ipv4_mask` | 32 | Mask bits for IPv4 addresses |
| `ipv6_mask` | 64 | Mask bits for IPv6 addresses |
| `asn_prefix` | `a:` | Prefix for ASN hashes |
| `country_prefix` | `c:` | Prefix for country hashes |
| `ip_prefix` | `i:` | Prefix for IP hashes |
| `score_divisor` | 1 | Divisor applied to the final IP score |
| `asn_cc_whitelist` | nil | Map of ASNs or country codes to skip entirely |
| `scores` | see below | Per-component score multipliers |

The `scores` sub-table controls how much each component contributes to the overall IP reputation score:

| Component | Default multiplier |
|-----------|-------------------|
| `asn` | 0.4 |
| `country` | 0.01 |
| `ip` | 1.0 |

### SPF selector specific options

The `spf` selector overrides the following common default:

| Option | Selector default | Description |
|--------|-----------------|-------------|
| `split_symbols` | **true** | `_HAM`/`_SPAM` sub-symbols are emitted by default |

### DKIM selector specific options

| Option | Default | Description |
|--------|---------|-------------|
| `max_accept_adjustment` | 2.0 | Maximum score adjustment for accepted DKIM signatures |

### URL selector specific options

| Option | Default | Description |
|--------|---------|-------------|
| `max_urls` | 10 | Maximum number of URLs to check per message |
| `check_from` | true | **No effect** — this option is present in the default config table but is never read by the URL selector code |

### Generic selector specific options

| Option | Default | Description |
|--------|---------|-------------|
| `selector` | required | Selector expression (see [selectors documentation](/configuration/selectors)) |
| `delimiter` | `:` | Delimiter for multiple selector results |
| `whitelist` | nil | Map of whitelisted selector values |
| `exclusion_map` | nil | Map of selector values to exclude from scoring |

## Redis backend options

In addition to the standard Redis connection parameters (see [Redis configuration](/configuration/redis)), the Redis backend accepts:

| Option | Default | Description |
|--------|---------|-------------|
| `expiry` | 864000 (10 days) | TTL in seconds for reputation keys |
| `prefix` | `RR:` | Global Redis key prefix |
| `buckets` | see above | Array of EWMA bucket definitions |
| `hashed` | false | Hash the token key before storing it in Redis |
| `hash_alg` | `blake2` | Hash algorithm to use when `hashed = true` (`blake2`, `sha256`, etc.) |
| `hashlen` | nil (no truncation) | Truncate the hash to this many characters |
| `hash_encoding` | `base32` | Encoding for the hash output (`base32`, `hex`, or `base64`) |
