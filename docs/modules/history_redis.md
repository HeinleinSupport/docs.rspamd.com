---
title: Redis history module
---


# Redis history module

The purpose of this module is to enable the storage of history in Redis lists with increased precision, thanks to its finer control over fields, optional compression, and out-of-the-box cluster support.

## Storage model

The Redis key used to store history is built from a Jinja template. The default template is `rs_history{{HOSTNAME}}{{COMPRESS}}`, where `{{HOSTNAME}}` is replaced with the local hostname and `{{COMPRESS}}` is replaced with `_zst` when compression is enabled (or an empty string when it is not). For example, on host `example.local` with compression enabled the key becomes `rs_history_example.local_zst`.

The template can be customised via the `key_prefix` setting.

## Compression

Rspamd uses [zstd](https://facebook.github.io/zstd/) compression, which is highly efficient for both compression and decompression, offering a typical compression rate of 50% for history elements. As long as you have sufficient computational power, it is recommended to use zstd in order to minimize Redis memory usage when storing elements.

## WebUI support

The Web Interface automatically recognises that history is stored in Redis and loads it appropriately.

## Configuration

The configuration of this module is pretty straightforward (use `local.d/history_redis.conf` to define your own values):

~~~hcl
servers = 127.0.0.1:6379; # Redis server to store history
key_prefix = "rs_history{{HOSTNAME}}{{COMPRESS}}"; # Key name template
expire = 432000; # Expire in seconds for inactive keys (no expiry by default)
nrows = 200; # Default rows limit
compress = true; # Use zstd compression when storing data in Redis
subject_privacy = false; # Subject privacy is off
subject_privacy_alg = "blake2"; # Hash algorithm used to obfuscate subjects
subject_privacy_prefix = "obf"; # Prefix prepended to obfuscated subjects
subject_privacy_length = 16; # Number of hash characters kept after truncation
~~~

**Note:** `expire` has no default — keys are kept indefinitely unless this option is set. The value `432000` (5 days) shown above is an example only.

**Note:** Reducing the `expire` value can result in a one-time situation where newer entries are deleted before older ones, creating a gap in the middle of the history.
