---
title: URL redirector module
---


# URL redirector module

This module provides a hook for [RBL](/modules/rbl) module to resolve redirects.
To enable this module one should add a `redirector_hosts_map` option to the module's configuration, i.e. by adding the following to `local.d/url_redirector.conf`:
~~~hcl
redirector_hosts_map = "${LOCAL_CONFDIR}/local.d/maps.d/redirectors.inc";
~~~

This file/URL should contain a list of domains that should be checked by URL redirector.

Dereferenced links are cached in Redis (see [here](/configuration/redis) for information on configuring redis), checked by SURBL module and added as tags for other modules.

# Configuration

The following settings could be set in `local.d/url_redirector.conf` to control behaviour of the URL redirector module.

~~~hcl
# Total per-message timeout budget in seconds (default 8)
timeout = 8;
# Per-hop HTTP request timeout in seconds (default 4).
# This caps each individual HEAD/GET request. timeout is the overall budget
# across all hops combined; http_timeout is the limit for a single hop.
# http_timeout may also be a table with connect_timeout, ssl_timeout,
# write_timeout and read_timeout for granular control.
http_timeout = 4;
# Redis operation timeout in seconds (default 2).
# The redis{} block timeout has higher priority when present.
redis_timeout = 2;
# How long to cache dereferenced links in Redis (default 1 day)
expire = 1d;
# How many nested redirects to follow (default 2)
nested_limit = 2;
# Prefix for keys in redis (default "rdr:")
key_prefix = "rdr:";
# Check SSL certificates (default false)
check_ssl = false;
# How many urls to check
max_urls = 5;
# Maximum body to process
max_size = 10k;
# Insert symbol if a redirect chain was resolved (no default; omit to suppress the symbol)
# redirector_symbol = "MY_REDIRECTOR_SYMBOL";
# Insert symbol when nested_limit is reached (default "URL_REDIRECTOR_NESTED")
redirector_symbol_nested = "URL_REDIRECTOR_NESTED";
# Insert symbol when a redirect leads to a non-HTTP(S) scheme (default "URL_REDIRECTOR_NON_HTTP")
redirector_symbol_non_http = "URL_REDIRECTOR_NON_HTTP";
# Follow merely redirectors
redirectors_only = true;
# Redis key for top urls
top_urls_key = 'rdr:top_urls';
# How many top urls to save
top_urls_count = 200;
# Check only those redirectors
redirector_hosts_map = "${LOCAL_CONFDIR}/local.d/maps.d/redirectors.inc";
# Regexp map of URL patterns that should be fetched with GET instead of HEAD (default unset)
# redirector_get_urls_map = "${LOCAL_CONFDIR}/local.d/maps.d/redirector_get_urls.inc";
# Control whether intermediate redirect hops are injected into the task
save_intermediate_redirs {
  # Inject hops that are themselves on redirector_hosts_map (default false)
  redirectors = false;
  # Inject hops that are NOT on redirector_hosts_map (default true).
  # Enabling this surfaces cloaker/rotator intermediates as task URLs.
  non_redirectors = true;
}
~~~

## Browser fingerprint profiles and user_agent

By default the module impersonates a real browser by picking one of five built-in fingerprint profiles at random (one profile per task, reused for every hop in every redirect chain). Each profile bundles a coherent set of HTTP request headers — including the exact header order, `User-Agent`, `Accept`, `Sec-Fetch-*` and, for Chromium-based browsers, `sec-ch-ua` client hints — matching what the corresponding real browser sends on a top-level navigation. The five built-in profiles are:

| Profile name | Browser |
|---|---|
| `chrome_win` | Google Chrome 148 on Windows 10 |
| `chrome_mac` | Google Chrome 148 on macOS |
| `edge_win` | Microsoft Edge 148 on Windows 10 |
| `firefox_win` | Firefox 150 on Windows 10 |
| `safari_mac` | Safari 26 on macOS |

To override this behaviour, set `user_agent` to a single string or a list of strings. When a list is provided, one entry is chosen at random per request. Setting `user_agent` disables the fingerprint profiles entirely and sends only a `User-Agent` header:

~~~hcl
# Single user agent string
user_agent = "Mozilla/5.0 (compatible; Rspamd/4.0; +https://rspamd.com)";

# Or a list — one is picked at random per request
# user_agent = [
#   "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...",
#   "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) ...",
# ];
~~~

To supply a fully custom set of fingerprint profiles (replacing the five built-in ones), set `fingerprint_profiles` to a list of profile objects, each with a `name` string and a `headers` list of `[name, value]` pairs.
