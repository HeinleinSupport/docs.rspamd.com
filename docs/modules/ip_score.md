---
title: IP Score module
---

# IP Score

:::warning Module removed in Rspamd 2.0
This module has been removed. The implementation file is a tombstone — configuring `ip_score` has no effect.

The module was retired due to serious design flaws: reputation tokens had no decay, which led to a positive-feedback loop and incorrect reputation calculations.
:::

## Migration

Use the [reputation module](/modules/reputation) instead. It provides equivalent IP, subnet, ASN, and country reputation tracking via the `SENDER_REP` selector family, with correct decay and a sound scoring model.

See the [reputation module documentation](/modules/reputation) for configuration details.
