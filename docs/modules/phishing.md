---
title: Phishing module
---

# Phishing module

The purpose of this module is to identify potentially phished URLs.

## Principles of phishing detection

Rspamd tries to detect phished URLs within HTML text parts. Firstly, it extracts the URL from either the `href` or `src` attribute and then searches for the text contained within the link tag. If a URL is enclosed within a specific tag, Rspamd compares it to the main URL to determine if they belong to the same top-level domain. URLs that belong to the same top-level domain are considered non-phished. Here are some examples of URLs that are not considered phished:

    <a href="http://sub.example.com/path">http://example.com/other</a>
    <a href="https://user:password@sub.example.com/path">http://example.com/</a>

And the following URLs are considered as phished:

    <a href="http://evil.co.uk">http://example.co.uk</a>
    <a href="http://t.co/xxx">http://example.com</a>
    <a href="http://redir.to/example.com">http://example.com</a>

## Configuration of phishing module

Here is an example of full module configuration.

~~~hcl
phishing {
	symbol = "PHISHED_URL"; # Default symbol

	# Check only domains from this list
	domains = "file:///path/to/map";

	# Make exceptions
	exceptions = {
		REDIRECTOR_FALSE = [
			"$LOCAL_CONFDIR/local.d/redirectors.inc",
		];
		PHISHED_WHITELISTED = [
			"$LOCAL_CONFDIR/local.d/phishing_whitelist.inc",
		];
	}
	phishing_exceptions = {
		OWN_DOMAINS = [
			"$LOCAL_CONFDIR/local.d/own_domains.inc",
		];
	}
}
~~~

If an actual (as opposed to phished) domain is detected in a map referred to by the `exceptions` setting, then the associated symbol is returned, and the URL is not checked further. This feature allows exclusions to be made for known redirectors, particularly ESPs.

In addition, if a phished domain is detected in a map referred to by `phishing_exceptions`, the associated symbol is returned, and the URL is not checked further. This feature allows for fine-grained control to prevent false positives and to identify highly dangerous phishing emails, such as those targeting banks or payment systems.

Finally, the default symbol is yielded- if `domains` is specified then
only if the phished domain is found in the related map.

The maps used in this module can consist of either effective second-level domain parts (eSLD) or whole domain parts of the URLs (FQDN).

### strict_domains

For domains that require stricter phishing detection (e.g., financial institutions), you can configure `strict_domains` which will always trigger the phishing symbol regardless of other checks:

~~~hcl
phishing {
    strict_domains = {
        STRICT_PHISHING = [
            "$LOCAL_CONFDIR/local.d/strict_phishing_domains.inc",
        ];
    }
}
~~~

## Options

| Option | Type | Default | Description |
|---|---|---|---|
| `symbol` | string | `PHISHED_URL` | Symbol inserted when a phished URL is detected |
| `domains` | map | — | When set, only emit `symbol` when the phished domain appears in this map |
| `exceptions` | map table | — | Per-symbol maps for anchor/redirector exceptions; matching URL skips further checks |
| `phishing_exceptions` | map table | — | Per-symbol maps for known-phishing exceptions; matching phished domain skips further checks |
| `strict_domains` | map table | — | Per-symbol maps; matching domains always fire at full weight |
| `openphish_enabled` | boolean | `false` | Enable the OpenPhish URL feed |
| `openphish_map` | URL/path | `https://raw.githubusercontent.com/openphish/public_feed/refs/heads/main/feed.txt` | URL or path for the OpenPhish plain-text feed |
| `openphish_url` | URL/path | — | Alias for `openphish_map` |
| `openphish_premium` | boolean | `false` | Set to `true` when using an OpenPhish premium JSON feed |
| `phishtank_enabled` | boolean | `false` | Enable Phishtank DNS lookups |
| `phishtank_suffix` | string | `phishtank.rspamd.com` | DNS zone used for Phishtank lookups |
| `generic_service_enabled` | boolean | `false` | Enable a custom generic phishing feed |
| `generic_service_map` | URL/path | — | URL or path for the generic feed data |
| `generic_service_url` | URL/path | — | Alias for `generic_service_map` |
| `generic_service_name` | string | `generic service` | Human-readable name for the generic service (used in log messages) |
| `generic_service_symbol` | string | `PHISHED_GENERIC_SERVICE` | Symbol inserted when a URL matches the generic feed |
| `phishing_feed_exclusion_enabled` | boolean | `false` | Enable per-host exclusions from all phishing feeds |
| `phishing_feed_exclusion_map` | URL/path | — | URL or path to the exclusion map (plain hostname list, no scheme/path) |
| `phishing_feed_exclusion_symbol` | string | `PHISHED_EXCLUDED` | Symbol inserted when a URL's host is found in the exclusion map |

## Symbols

| Symbol | Description |
|---|---|
| `PHISHED_URL` | Default symbol — a phished URL was detected |
| `PHISHED_OPENPHISH` | URL matched the OpenPhish feed |
| `PHISHED_PHISHTANK` | URL matched the Phishtank DNS feed |
| `PHISHED_GENERIC_SERVICE` | URL matched the generic service feed (default; overridden by `generic_service_symbol`) |
| `PHISHED_EXCLUDED` | URL host found in the phishing feed exclusion map (default; overridden by `phishing_feed_exclusion_symbol`) |

## Openphish support

Starting from version 1.3, Rspamd provides support for [openphish](https://openphish.com)  This public feed can be loaded as a map in Rspamd using HTTPS and utilized to check URLs in messages against the openphish list. In case of any match, Rspamd adds the symbol `PHISHED_OPENPHISH`.

Moreover, if you are using a research or commercial data feed, Rspamd can also leverage its data to provide more details about the URLs detected, such as their sector (e.g., "Finance"), brand name (e.g., "Bank of Zimbabwe"), and other relevant information.

To configure the openphish module, there are a couple of available options:

~~~hcl
phishing {
	# Enable openphish support (default disabled)
	openphish_enabled = true;
	# URL of feed, default is public url:
	openphish_map = "https://raw.githubusercontent.com/openphish/public_feed/refs/heads/main/feed.txt";
	# For premium feed, change that to your personal URL, e.g.
	# openphish_map = "https://openphish.com/samples/premium_feed.json";

	# Change this to true if premium feed is enabled
	openphish_premium = false;
}
~~~

## Phishtank support

Rspamd has included support for [phishtank](https://phishtank.com) since version 1.3. The module default is `phishtank_enabled = false`; the stock Rspamd configuration may enable it separately via `phishtank.rspamd.com` DNS. To enable or disable Phishtank lookups in your local configuration:

~~~hcl
# local.d/phishing.conf
phishtank_enabled = true;  # set to false to disable
~~~

You can also use your own phishtank DNS zone:

~~~hcl
# local.d/phishing.conf
local phishtank_suffix = 'phishtank.rspamd.com'; # Replace with your own zone
~~~

## Generic feed support

To enable support for custom phishing maps from a local file or online URL catalog, you need to enable the generic service support by creating and enabling a service definition. You'll also need to have a local file or URL containing the map data. For instance, you can use a local map from the [CaUMa](https://cauma.pop-ba.rnp.br/about.html) URLs catalog.


~~~hcl
# local.d/phishing.conf
generic_service_enabled = true;
generic_service_name = 'CaUMa';
generic_service_symbol = "PHISHED_CAUMA";
generic_service_map = "file:///path/to/map";;
~~~

The following definition is also necessary to define a weight value to the symbol.

~~~hcl
# local.d/phishing_group.conf
symbols {
    "PHISHED_CAUMA" {
        weight = 5.0;
        description = "Phished URL";
        one_shot = true;
    }
}
~~~

## Exclusions from phishing feeds

To exclude hosts from phishing feed checks (Openphish, Phishtank, or Generic) you need to enable phishing feed exclusion and configure map data to a local file or online hosts catalog. The exclusion map should only contain a list of host names without a scheme and path. It is available in version 3.7 and greater.

~~~hcl
# local.d/phishing.conf
phishing_feed_exclusion_enabled = true;
phishing_feed_exclusion_map = "file:///path/to/map";
~~~
