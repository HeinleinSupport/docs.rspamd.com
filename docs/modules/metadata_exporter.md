---
title: Metadata exporter
---


# Metadata exporter

The Metadata exporter operates on a set of rules that identify interesting messages, and subsequently sends information based on these rules to an external service. The exporter supports Redis Pub/Sub, HTTP POST, and SMTP as built-in backends, while also allowing users to define custom backends as desired.

Potential applications of the Metadata exporter include quarantining, logging, alerting, and feedback loops.

### Theory of operation

For each rule defined in configuration:

 - A `selector` function identifies messages that we want to export metadata from (default selector selects all messages).
 - A `formatter` function extracts formatted metadata from the message (default formatter returns full message content).
 - A `pusher` function (defined by the `backend` setting) pushes the formatted metadata somewhere

A number of such functions are defined in the plugin which can be used in addition to user-defined functions.

### Configuration

~~~hcl
metadata_exporter {

  # Each rule defines some export process

  rules {

    # The following rule posts JSON-formatted metadata at the defined URL
    # when it sees a rejected mail from an authenticated user
    MY_HTTP_ALERT_1 {
      backend = "http";
      url = "http://127.0.0.1:8080/foo";
      # More about selectors and formatters later
      selector = "is_reject_authed";
      formatter = "json";
    }

    # This rule posts all messages to a Redis Pub/Sub channel
    MY_REDIS_PUBSUB_1 {
      backend = "redis_pubsub";
      channel = "foo";
      # Default formatter and selector is used
    }

    # This rule sends an e-Mail alert over SMTP containing message metadata
    # when it sees a rejected mail from an authenticated user
    MY_EMAIL_1 {
      backend = "send_mail";
      smtp = "127.0.0.1";
      mail_to = "user@example.com";
      selector = "is_reject_authed";
      formatter = "email_alert";
    }

  }

}
~~~

### Stock pushers (backends)

 - `http`: sends content over HTTP POST
 - `json_raw_tcp`: sends JSON content over a raw TCP connection
 - `redis_pubsub`: sends content over Redis Pub/Sub
 - `redis_stream` (4.0+): sends content to a Redis Stream
 - `send_mail`: sends content over SMTP

### Stock selectors

 - `default`: selects all mail
 - `is_spam`: matches messages with `reject`, `add header`, or `rewrite subject` action
 - `is_spam_authed`: matches messages with `reject`, `add header`, or `rewrite subject` action from authenticated users
 - `is_reject`: matches messages with `reject` action
 - `is_reject_authed`: matches messages with `reject` action from authenticated users
 - `is_not_soft_reject`: matches all messages except those with `soft reject` action

### Stock formatters

 - `default`: returns full message content
 - `email_alert`: generates an e-Mail report about the message
 - `json`: returns JSON-formatted metadata about a message
 - `multipart` (3.14.2+): Sends metadata as JSON part + raw message as `message/rfc822` part using standard `multipart/form-data`
 - `msgpack` (3.14.2+): Binary MessagePack format with embedded message (efficient for binary data)
 - `json_with_message` (3.14.2+): JSON with base64-encoded message
 - `structured` (4.0+): Rich structured export with Rspamd UUID correlation, extracted text, attachments, images, URLs in MessagePack format

### Settings: general

The following settings can be defined on any rule:

 - `selector`: defines selector for the rule
 - `formatter`: defines formatter for the rule
 - `backend`: defines backend (pusher) for the rule
 - `defer`: if true, `soft reject` action is forced on failed processing
 - `timeout`: defines module timeout (default: '5s')

### Selectors and custom variables in rule options

*Available since version 4.1*

A small set of per-message routing options can reference a [selector](/configuration/selectors) or a user-defined custom variable instead of (or in addition to) a literal value. Two placeholder forms are recognized inside the value of a supported option:

 - `$name` — expands to the value returned by a custom variable named `name`
 - `${expr}` — expands to the value returned by a custom variable named `expr`, or, if no such variable is defined, evaluates `expr` as a selector expression

Custom variables are defined in the `custom_variables` group and referenced by name, similarly to `custom_select`/`custom_format`/`custom_push`:

~~~hcl
metadata_exporter {
  custom_variables {
    alert_mailbox = "return function(task) return 'security@example.com' end";
  }

  rules {
    MY_EMAIL_1 {
      backend = "send_mail";
      smtp = "127.0.0.1";
      mail_to = "$alert_mailbox";
      selector = "is_reject_authed";
      formatter = "email_alert";
    }
  }
}
~~~

Each custom variable is a Lua function that receives the `task` object and returns a string (or a list of strings).

#### Options that support expansion

Only the following options accept `$name`/`${expr}` placeholders:

 - `mail_from`, `mail_to`, `helo` (`send_mail` backend)
 - `channel` (`redis_pubsub` backend)
 - `stream_key` (`redis_stream` backend)
 - the `email_template` body (`email_alert` formatter)

`mail_to` additionally accepts a list mixing literal addresses and placeholders:

~~~hcl
mail_to = ["${rcpts:addr}", "backup@example.com"];
~~~

Expanded email addresses are parsed and validated; any result that is not a valid address is dropped and logged rather than being sent.

All other options — including `url`, `host`, `port`, `smtp`, `smtp_port`, `user`, `password`, `mime_type`, `meta_header_prefix`, every `*_timeout` setting, `max_len`, and all boolean flags — are always taken literally and are never expanded, even if their configured value happens to contain a `$` character. This is intentional: message-derived data may influence *who* receives an alert, but never *where* a rule connects or *how* it authenticates.

Expanded values are also sanitized: any CR/LF sequence is collapsed to a single space, so a hostile selector result cannot inject extra SMTP commands, mail headers, or Redis protocol framing.

Within `email_template`, the same placeholders can additionally reference the general metadata keys described below (e.g. `$mail_from`, `$user`, `$score`). Metadata keys take priority over custom variables and selectors of the same name.

### Settings: `http` backend

 - `url` (required): defines the URL to post content to
 - `meta_header_prefix`: prefix for meta headers (default: `'X-Rspamd-'`)
 - `meta_headers` (bool): if set to `true`, general metadata is added to HTTP request headers (default: `false`). **Deprecated in 3.14.2**: Use `formatter = "multipart"` or `formatter = "msgpack"` instead.
 - `mime_type`: defines the MIME type of the content sent in the HTTP POST
 - `user` & `password`: if both parameters are set, Basic authentication will be used
 - `gzip` (bool): specifies whether the payload needs to be sent with gzip compression (default: `false`)
 - `keepalive` (bool): specifies whether the connection should use keepalive (default: `false`)
 - `no_ssl_verify` (bool): disable SSL certificate verification (default: `false`)
 - `connect_timeout`: timeout for establishing the TCP connection
 - `ssl_timeout`: timeout for SSL handshake
 - `write_timeout`: timeout for writing the request body
 - `read_timeout`: timeout for reading the response

### Settings: `redis_pubsub` backend

 - `channel` (required): defines Pub/Sub channel to post content to (supports `$name`/`${expr}` placeholders, 4.1+)

See [here](/configuration/redis) for information on configuring Redis servers.

### Settings: `redis_stream` backend

*Available since version 4.0*

 - `stream_key` (required): defines Redis Stream key to append content to (supports `$name`/`${expr}` placeholders, 4.1+)
 - `max_len`: optional maximum length for the stream (uses `MAXLEN ~` for approximate trimming)
 - `per_recipient`: if `true`, creates per-recipient streams by appending `:recipient@address` to `stream_key`

The backend uses Redis `XADD` command. This is useful for building event-driven pipelines with consumer groups.

### Settings: `json_raw_tcp` backend

 - `host` (required): hostname or IP address of the TCP server
 - `port` (required): TCP port to connect to

The backend sends the formatted content as-is over a raw TCP connection without an application-layer framing protocol. No response is read from the server. Combine with the `json` formatter to push newline-delimited JSON to a log aggregator or SIEM.

### Settings: `send_mail` backend

If the `send_mail` backend is used with the default formatter, the original spam message content will be analyzed by Rspamd and is highly likely matched as spam.

When `send_mail` backend is used in conjunction with `email_alert` formatter, the URLs found in the symbols options will be analysed by Rspamd and the report will be matched as spam possibly.

<mark>To prevent <b>looping</b>, it is essential to ensure that email messages from the Metadata exporter are <b>not scanned</b> by Rspamd.</mark> This can be achieved by setting up a specific Postfix Transport to bypass Rspamd, or by allowing the recipient of the `email_alert` to receive spam.

 - `smtp` (required): hostname of SMTP server
 - `mail_to` (required): recipient of e-mail alert. May be a literal address, a `$name`/`${expr}` placeholder (4.1+), or a list combining any of these forms
 - `mail_from`: Sender address (default empty; supports `$name`/`${expr}` placeholders, 4.1+)
 - `email_alert_user` (1.7.0+, default false): Send a copy of the alert to the authenticated SMTP username
 - `email_alert_sender` (1.7.0+, default false): Send a copy of the alert to the SMTP sender (NB: please ensure that it can be trusted)
 - `email_alert_sender_variable` (4.1+): Send a copy of the alert to the address returned by the named custom variable
 - `email_alert_recipients` (1.7.0+, default false): Send a copy of the alert to SMTP recipients (NB: please ensure they can be trusted; don't use this?)
 - `email_template`: template used for alert (default shown below)
 - `helo`: HELO to send (default 'rspamd'; supports `$name`/`${expr}` placeholders, 4.1+)
 - `smtp_port`: SMTP port if not 25

The default value for `email_template` is as follows:

~~~
From: "Rspamd" <$mail_from>
To: $mail_to
Subject: Spam alert
Date: $date
MIME-Version: 1.0
Message-ID: <$our_message_id>
Content-type: text/plain; charset=utf-8
Content-Transfer-Encoding: 8bit

Authenticated username: $user
IP: $ip
Queue ID: $qid
SMTP FROM: $from
SMTP RCPT: $rcpt
MIME From: $header_from
MIME To: $header_to
MIME Date: $header_date
Subject: $header_subject
Message-ID: $message_id
Action: $action
Score: $score
Symbols: $symbols
~~~

Variables can be substituted according to general metadata keys described in the next section.

### General metadata

Metadata as returned by the `json` formatter can be referenced by key in `email_template`. The following keys are defined:

- `action`: metric action for message
- `from`: SMTP FROM
- `header_date`: Contents of Date header(s)
- `header_from`: Contents of From header(s)
- `header_subject`: Contents of Subject header(s)
- `header_to`: Contents of To header(s)
- `ip`: IP of message sender
- `mail_from` (`email_template` only): sender of alert
- `mail_to` (`email_template` only): recipient of alert
- `message_id`: Message-ID of original message
- `our_message_id` (`email_template` only): message-ID generated for alert
- `qid`: Queue-ID of message provided by MTA
- `rcpt`: SMTP RCPT
- `score`: Metric score of the message
- `symbols`: Symbols in metric, in their natural (unordered) evaluation order
- `symbols_sorted` (`email_template` only): Symbols in metric, sorted alphabetically by name
- `symbols_score` (`email_template` only): Symbols in metric, sorted by score (highest first)
- `user`: authenticated username of message sender
- `subject`: Subject of the message as parsed by Rspamd
- `rspamd_server`: hostname of the Rspamd instance that processed the message
- `fuzzy`: comma-separated list of fuzzy hashes matched for the message, if any
- `scan_time`: message scan time in milliseconds
- `size`: message size in bytes
- `date` (`email_template` only): current date/time, formatted for use in a `Date:` header

### Predefined template variables

In addition to the general metadata keys above, the following predefined variables are always available for `$name`/`${name}` substitution in `email_template` (they live in the same namespace as [custom variables](#selectors-and-custom-variables-in-rule-options), so a `custom_variables` entry with the same name overrides the built-in one):

 - `content`: the raw content of the original message. This is what lets you attach the original message to an alert, e.g. as a `message/rfc822` MIME part (see the example below)
 - `uid`: the first 6 characters of the internal Rspamd task UID
 - `local_date`: current local date/time, formatted with `%c` (e.g. `Sat Aug 22 10:00:00 2026`)
 - `our_boundary`: a random MIME boundary string, generated once per alert, for building a multipart `email_template`

### Custom functions

It is possible to define custom selectors/pushers/backends. Functions are defined in the `custom_select`/`custom_format`/`custom_push` groups and referenced by name in the `selector`/`formatter`/`backend` settings:

~~~hcl
metadata_exporter {

  # Define custom selector(s)
  custom_select {
    mine = <<EOD
return function(task)
  -- Select all messages
  return true
end
EOD;
  }

  # Define custom formatter(s)
  custom_format {
    mine = <<EOD
return function(task)
  -- Push message ID
  return task:get_message_id()
end
EOD;
  }

  # Define custom backend(s)
  custom_push {
    mine = <<EOD
return function (task, data, rule)
  -- Log payload
  local rspamd_logger = require "rspamd_logger"
  rspamd_logger.infox(task, 'METATEST %s', data)
end
EOD;
  }

  rules {

    CUSTOM_EXPORT {
      selector = "mine";
      formatter = "mine";
      backend = "mine";
    }

  }

}
~~~

### Examples

#### Email alert with the original message attached as `message/rfc822`

This example builds a multipart e-mail alert with the `send_mail` backend and `email_alert` formatter: a plain-text summary of the scan result, plus the original message attached as a `message/rfc822` part using the predefined `$content` and `$our_boundary` variables.

~~~hcl
metadata_exporter {

  rules {

    SPAM_WITH_ORIGINAL {
      backend = "send_mail";
      smtp = "127.0.0.1";
      mail_from = "postmaster@example.com";
      mail_to = "security@example.com";
      selector = "is_reject_authed";
      formatter = "email_alert";
      email_template = 'From: "Rspamd" <$mail_from>
To: $mail_to
Subject: [SPAM $score] $header_subject
Date: $date
MIME-Version: 1.0
Message-ID: <$our_message_id>
Content-Type: multipart/mixed;
 boundary="$our_boundary"; charset=utf-8

This is a multi-part message in MIME format.
--$our_boundary
Content-Type: text/plain; charset=utf-8
Content-Transfer-Encoding: 8bit

Authenticated username: $user
IP: $ip
Queue ID: $qid
SMTP FROM: $from
SMTP RCPT: $rcpt
Action: $action
Score: $score
Symbols: $symbols_sorted

--$our_boundary
Content-Type: message/rfc822;
 name="$qid.eml"
Content-Transfer-Encoding: 8bit
Content-Disposition: attachment;
 filename="$qid.eml"

$content

--$our_boundary
';
    }

  }

}
~~~

Note that `content` is the raw, unmodified message, so `$content` must be placed in the template as-is, without any additional encoding. Also remember the looping warning above: the mailbox receiving this alert must not be re-scanned by Rspamd, or the attached original message could trigger the rule again.

#### Python Receiver for `multipart` formatter

This example demonstrates how to build a simple service using `aiohttp` that accepts metadata and messages sent by the `metadata_exporter` with `formatter = "multipart"`.

It conceptually shows how to quarantine messages to Kafka or Cassandra.

Prerequisites:
```bash
pip install aiohttp aiokafka cassandra-driver
```

receiver.py:
```python
import asyncio
import json
import logging
from aiohttp import web
# Optional: import for Kafka/Cassandra
# from aiokafka import AIOKafkaProducer
# from cassandra.cluster import Cluster

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("rspamd-receiver")

async def handle_push(request):
    """
    Handle POST request from Rspamd metadata_exporter (multipart).
    Expected parts:
    - 'metadata': JSON object with Rspamd metadata
    - 'message': Raw message content (optional)
    """
    reader = await request.multipart()
    metadata = {}
    message_content = b""

    # Iterate through multipart fields
    async for field in reader:
        if field.name == 'metadata':
            # Parse metadata JSON
            try:
                raw_json = await field.read()
                metadata = json.loads(raw_json)
                logger.info(f"Received metadata for Message-ID: {metadata.get('message_id')}")
            except Exception as e:
                logger.error(f"Failed to parse metadata: {e}")
                return web.Response(status=400, text="Invalid Metadata")
        
        elif field.name == 'message':
            # Read raw message content
            message_content = await field.read()
            logger.info(f"Received message content ({len(message_content)} bytes)")
    
    if not metadata:
        return web.Response(status=400, text="Missing Metadata")

    # --- Quarantine Logic Example ---
    
    # 1. Kafka Example (Async)
    # await produce_to_kafka(metadata, message_content)
    
    # 2. Cassandra Example (Async)
    # await save_to_cassandra(metadata, message_content)
    
    logger.info("Message processed successfully")
    return web.Response(text="OK")

# Mock functions for illustration
async def produce_to_kafka(metadata, content):
    # producer = AIOKafkaProducer(bootstrap_servers='localhost:9092')
    # await producer.start()
    # try:
    #     await producer.send_and_wait("rspamd-quarantine", value=content, key=metadata.get('message_id').encode())
    # finally:
    #     await producer.stop()
    pass

async def save_to_cassandra(metadata, content):
    # loop = asyncio.get_event_loop()
    # cluster = Cluster(['127.0.0.1'])
    # session = cluster.connect('mail_quarantine')
    # stmt = "INSERT INTO messages (id, metadata, content) VALUES (%s, %s, %s)"
    # await loop.run_in_executor(None, session.execute, stmt, (metadata['message_id'], json.dumps(metadata), content))
    pass

app = web.Application()
app.add_routes([web.post('/push', handle_push)])

if __name__ == '__main__':
    web.run_app(app, port=8080)
```

Configure Rspamd to use this receiver:

```hcl
metadata_exporter {
  rules {
    QUARANTINE {
      backend = "http";
      url = "http://127.0.0.1:8080/push";
      selector = "is_reject"; # Export rejected messages
      formatter = "multipart"; # Send metadata + raw message
    }
  }
}
```

### Structured formatter (4.0+)

The `structured` formatter provides rich, analysis-ready metadata in MessagePack format with Rspamd UUID correlation:

~~~hcl
metadata_exporter {
  rules {
    STRUCTURED_EXPORT {
      backend = "redis_stream";
      formatter = "structured";
      stream_key = "rspamd:events";
      max_len = 10000;
      # Optional: compress text/attachments with zstd
      zstd_compress = true;
    }
  }
}
~~~

#### Structured formatter output fields

| Field | Type | Description |
|-------|------|-------------|
| `uuid` | String | Rspamd internal message UUID (from `task:get_uuid()`) |
| `ip` | String | Sender IP address |
| `from` | String | SMTP envelope sender |
| `rcpt` | String | SMTP envelope recipient |
| `user` | String | Authenticated username |
| `score` | Number | Spam score |
| `action` | String | Rspamd action |
| `symbols` | Object | Symbol results |
| `text` | String/Binary | Extracted plain text (optionally zstd-compressed) |
| `text_truncated` | Boolean | True if text was truncated (max 32KB) |
| `text_compressed` | Boolean | True if text is zstd-compressed |
| `attachments` | Array | Attachment metadata with content |
| `images` | Array | Embedded image metadata with content |
| `urls` | Array | Extracted URLs with host/TLD |
| `is_reply` | Boolean | True if message has In-Reply-To header |

#### Structured formatter features

- **UUID correlation**: Rspamd's internal message UUID enables cross-system correlation via the injected `X-Rspamd-UUID` header
- **X-Rspamd-UUID header**: Automatically injected into message for IMAP/external correlation
- **Smart text extraction**: Cleaned, reply-trimmed text up to 32KB
- **Attachment analysis**: Includes detected MIME type (not just announced), size, digest, and optional content
- **URL extraction**: Up to 100 URLs with host and TLD information
- **Zstd compression**: Optional compression for text, attachments, and images to reduce storage

#### Zstd compression option

When `zstd_compress = true` is set:
- Text, attachment content, and image content are compressed with zstd
- Compressed fields include `content_compressed = true` or `text_compressed = true`
- Consumer must decompress using zstd library
