---
title: GPT Plugin
---

# Rspamd GPT Plugin

The Rspamd GPT Plugin, introduced in Rspamd 3.9, integrates OpenAI's GPT API to enhance spam filtering capabilities using advanced natural language processing techniques. Here are the basic ideas behind this plugin:

* The selected displayed text part is extracted and submitted to the GPT API for spam probability assessment
* Additional message details such as Subject, displayed From, and URLs are also included in the assessment
* Then, we ask GPT to provide results in JSON format since human-readable GPT output cannot be parsed (in general)
* Some specific symbols (`BAYES_SPAM`, `FUZZY_DENIED`, `REPLY`, etc.) are excluded from the GPT scan
* Obvious spam and ham are also excluded from the GPT evaluation

The last two points reduce the GPT workload for something that is already known, where GPT cannot add any value in the evaluation. We also use GPT as one of the classifiers, meaning that we do not rely solely on GPT evaluation.



## Configuration Options

**By default, the GPT Plugin is disabled.** To enable the plugin, add the following command in your Rspamd configuration:

```hcl
gpt {
  enabled = true; # Ensure this line is present to enable the GPT Plugin
}
```

The full list of the plugin configuration options:

```hcl
gpt {
  # Enable the plugin
  enabled = true;

  # LLM provider type: openai (remote) or ollama (local)
  type = "openai";
  
  # Your OpenAI API key (not required for ollama)
  api_key = "xxx";
  
  # Model name (string or a list for ensemble requests)
  model = "gpt-5-mini";

  # Per-model parameters — works for both openai and ollama.
  # Each key is the model name; the value is merged into the request body.
  model_parameters = {
    "gpt-5-mini" = {
      max_completion_tokens = 1000,
    },
    "gpt-5-nano" = {
      max_completion_tokens = 1000,
    },
    "gpt-4o-mini" = {
      max_tokens = 1000,
      temperature = 0.0,
    }
  };

  # Overall HTTP request timeout (seconds)
  timeout = 10s;

  # Fine-grained HTTP stage timeouts (all optional; unset = use timeout)
  # connect_timeout = 3;
  # ssl_timeout = 3;
  # write_timeout = 5;
  # read_timeout = 10;

  # Optional server-side time budget passed as max_completion_time in the
  # request body. Not supported by the standard OpenAI API; only enable when
  # your endpoint/proxy explicitly supports it. The value sent is multiplied
  # by 0.95 to account for connection overhead.
  # request_timeout = 8;

  # Prompt for the model (use default if not set)
  prompt = "xxx";
  
  # Custom condition (Lua function)
  condition = "xxx";
  
  # Autolearn if GPT classified
  autolearn = true;

  # Custom Lua function to convert the model's reply. Leave unset to use the
  # built-in parsers for OpenAI / Ollama replies.
  reply_conversion = "xxx";

  # Controls how reply text is trimmed before sending to the LLM.
  # Accepted values: "replies" (default), "all", "none".
  reply_trim_mode = "replies";

  # Minimum number of words a text part must contain to be considered.
  # Parts shorter than this are skipped. 0 accepts any length.
  min_words = 10;

  # Header to add with the reason produced by GPT. Disabled (nil) by default.
  # Set to a header name string to enable, e.g.:
  # reason_header = "X-GPT-Reason";

  # Skip GPT scan if **any** of these symbols are present (and their absolute
  # weight is equal or above the specified value)
  symbols_to_except = {
    BAYES_SPAM = 0.9;
    WHITELIST_SPF = -1;
  };

  # Trigger GPT scan only when **all** of these symbols are present (and their
  # absolute weight is equal or above the specified value)
  # symbols_to_trigger = {
  #   URIBL_BLOCKED = 1.0;
  # };

  # Check messages that resulted in a `passthrough` action
  allow_passthrough = false;

  # Check messages that already look like ham (negative score)
  allow_ham = false;

  # Request / expect JSON reply from GPT
  json = false;

  # Minimum spam probability (from ensemble voting) to set GPT_SPAM
  consensus_spam_threshold = 0.75;

  # Maximum ham probability (from ensemble voting) to set GPT_HAM
  consensus_ham_threshold = 0.25;

  # Map of extra virtual symbols that could be set from GPT response categories
  # extra_symbols = {
  #   GPT_MARKETING = {
  #     score = 0.0;
  #     description = "GPT model detected marketing content";
  #     category = "marketing";
  #     group = "GPT";
  #   };
  # };

  # Prefix for Redis cache keys
  cache_prefix = "rsllm";

  # Add `response_format = {type = "json_object"}` to requests (OpenAI only)
  include_response_format = false;

  # API endpoint
  url = "https://api.openai.com/v1/chat/completions";

  # Custom context augmentation — a Lua expression that returns a function
  # with signature: function(task, content, callback). The callback must be
  # called with a string that is injected as additional context into the LLM
  # prompt. Async operations (Redis, HTTP) are supported.
  # context_augment = "return function(task, content, cb) cb('extra context') end";

  # Optional user/domain conversation context stored in Redis
  context = {
    enabled = false;
    # ... see Context block options below
  };

  # Optional web-search context (domains extracted from URLs in the email)
  search_context = {
    enabled = false;
    # ... see Search Context block options below
  };
}
```

### Core options

| Option | Type | Default | Description |
|---|---|---|---|
| `type` | string | `"openai"` | LLM provider. Accepted values: `openai`, `ollama` |
| `api_key` | string | — | API key. Required for `openai`; not needed for `ollama` |
| `model` | string or list | `"gpt-5-mini"` | Model name, or a list of names to query in parallel (ensemble mode) |
| `model_parameters` | table | see defaults | Per-model request body overrides (e.g. `max_completion_tokens`, `temperature`). Works for both `openai` and `ollama` |
| `url` | string | `"https://api.openai.com/v1/chat/completions"` | API endpoint |
| `timeout` | number | `10` | Overall HTTP request timeout in seconds |
| `connect_timeout` | number | `nil` | TCP connect timeout in seconds (falls back to `timeout`) |
| `ssl_timeout` | number | `nil` | TLS handshake timeout in seconds (falls back to `timeout`) |
| `write_timeout` | number | `nil` | Request write timeout in seconds (falls back to `timeout`) |
| `read_timeout` | number | `nil` | Response read timeout in seconds (falls back to `timeout`) |
| `request_timeout` | number | `nil` | Optional server-side time budget sent as `max_completion_time` in the request body. Multiplied by 0.95 before sending. Not supported by the standard OpenAI API |
| `prompt` | string | `nil` | System prompt. A sensible default is used when not set |
| `condition` | string | `nil` | Lua code returning a function that decides whether to run GPT on a message |
| `autolearn` | boolean | `false` | Feed GPT verdicts into Bayes learner |
| `reply_conversion` | string | `nil` | Lua code returning a custom reply parser. Leave unset to use built-in parsers |
| `reply_trim_mode` | string | `"replies"` | Controls body trimming before sending to the LLM: `replies`, `all`, or `none` |
| `min_words` | number | `10` | Minimum word count for a text part to be selected. `0` accepts any length |
| `reason_header` | string | `nil` | Header name to inject with the GPT explanation (e.g. `"X-GPT-Reason"`). Disabled by default |
| `symbols_to_except` | table | see defaults | Symbols that skip GPT processing; value is the minimum absolute weight (`-1` = any weight) |
| `symbols_to_trigger` | table | `nil` | Symbols that must all be present to trigger GPT processing |
| `allow_passthrough` | boolean | `false` | Still run GPT on messages with a `passthrough` action |
| `allow_ham` | boolean | `false` | Still run GPT on messages that already have a negative score |
| `json` | boolean | `false` | Ask the model for a JSON response and parse it as such |
| `consensus_spam_threshold` | number | `0.75` | Minimum spam probability required when resolving multi-model ensemble votes |
| `consensus_ham_threshold` | number | `0.25` | Maximum ham probability required when resolving multi-model ensemble votes |
| `extra_symbols` | table | see defaults | Extra virtual symbols the model can set via returned categories |
| `cache_prefix` | string | `"rsllm"` | Redis key prefix for response caching |
| `include_response_format` | boolean | `false` | Add `response_format = json_object` to the OpenAI request (only useful with `json = true`) |
| `context_augment` | string | `nil` | Lua expression returning a `function(task, content, callback)` for custom async context injection |

### `context` block — user/domain conversation context

This optional block enables per-user or per-domain conversation history stored in Redis. When enabled, a compact digest of recent messages is injected into the LLM prompt so the model can take prior classification history into account. Context is updated after every classification.

| Option | Type | Default | Description |
|---|---|---|---|
| `enabled` | boolean | `false` | Enable conversation context |
| `level` | string | `"user"` | Identity scope: `user`, `domain`, or `esld` |
| `key_prefix` | string | `"user"` | Redis key prefix component |
| `key_suffix` | string | `"mail_context"` | Redis key suffix component |
| `max_messages` | number | `40` | Maximum compact message summaries to keep in the sliding window |
| `min_messages` | number | `5` | Warm-up threshold — context is injected into the prompt only after this many messages have been collected |
| `message_ttl` | number | `1209600` | Messages older than this (seconds, default 14 days) are discarded when the window is recomputed |
| `ttl` | number | `2592000` | Redis key TTL in seconds (default 30 days) |
| `top_senders` | number | `5` | Number of top senders to track |
| `summary_max_chars` | number | `512` | Maximum characters of message body to store per summary |
| `flagged_phrases` | list | `["reset your password", "click here to verify"]` | Phrases to flag in stored summaries |
| `last_labels_count` | number | `10` | Number of recent classification labels to retain |
| `as_system` | boolean | `true` | Inject the context snippet as a system message (`false` = user message) |
| `enable_map` | table | `nil` | Simple map-based gating: `{selector = "...", map = "/path/to/map", type = "set"}` |
| `enable_expression` | table | `nil` | Maps-expression gating to enable context for specific conditions |
| `disable_expression` | table | `nil` | Maps-expression gating to disable context for specific conditions |

### `search_context` block — web search context

This optional block enables on-the-fly domain lookups for URLs found in the email. Search results are cached in Redis and injected into the LLM prompt as additional context.

| Option | Type | Default | Description |
|---|---|---|---|
| `enabled` | boolean | `false` | Enable web search context |
| `search_url` | string | `"https://leta.mullvad.net/search/__data.json"` | Search API endpoint |
| `search_engine` | string | `"brave"` | Search engine hint passed to the API (e.g. `brave`, `google`) |
| `max_domains` | number | `3` | Maximum number of unique domains to search |
| `max_results_per_query` | number | `3` | Maximum search results to use per domain |
| `timeout` | number | `5` | HTTP timeout in seconds for search requests |
| `cache_ttl` | number | `3600` | Search result cache TTL in seconds (1 hour) |
| `cache_key_prefix` | string | `"gpt_search"` | Redis cache key prefix for search results |
| `as_system` | boolean | `true` | Inject search context as a system message (`false` = user message) |
| `enable_expression` | table | `nil` | Maps-expression gating to enable search context for specific conditions |
| `disable_expression` | table | `nil` | Maps-expression gating to disable search context for specific conditions |

## Example Configuration

Here is a minimal example configuration:

```hcl
gpt {
  enabled = true;
  type = "openai";
  api_key = "your_api_key_here";
  model = "gpt-5-mini";
  model_parameters = {
    "gpt-5-mini" = {
      max_completion_tokens = 1000,
    }
  }
  timeout = 10s;
}
```

## Additional Notes

### JSON vs. Plain-Text Responses
The plugin works best when the model can answer in plain text because it gives the language model more freedom and generally yields better reasoning. Enabling `json = true` (and optionally `include_response_format`) constrains the model to produce a strict JSON object. This often degrades the probability estimate or makes the reply longer than required, so turn it on only if your prompts really need JSON.

### `reason_header`
The `reason_header` option is **disabled by default** (`nil`). Set it to a header name string (e.g. `reason_header = "X-GPT-Reason"`) to inject a mail header with the explanation produced by GPT. This is convenient for debugging or for downstream systems, but remember that the text is untrusted input and might reveal information to message recipients.

### Multiple-Model Ensemble
The `model` option can be a list:

```hcl
model = ["gpt-4o-mini", "gpt-5-nano"];
```

In this case Rspamd queries all listed models in parallel and applies a *consensus* algorithm:
* Each model returns a spam probability.
* If the majority classifies the message as spam with probability > `consensus_spam_threshold` (default 0.75), the highest spam probability is used.
* If the majority classifies it as ham with probability < `consensus_ham_threshold` (default 0.25), the lowest ham probability is used.
* Otherwise no GPT symbol is added (no consensus), and `GPT_UNCERTAIN` is set instead.

This improves robustness and lets you mix cheap fast models with a slower high-quality one.

### Caching Policies
To avoid repeated LLM calls, responses are cached in Redis (if configured). Key facts:
* Key: `<cache_prefix>_<env>_<digest>` where `env` depends on `prompt`, `model`, and `url`, and `digest` is the SHA256 of the examined text part (truncated).
* Default TTL is one hour (`cache_ttl` option accepts seconds).
* Workers coordinate using *pending* markers so that only one request per message is sent even in large clusters.
* Changing the prompt, model list, or endpoint automatically invalidates the cache because they are part of the key.
* You can tune `cache_prefix`, `cache_ttl`, and other cache options directly inside the `gpt {}` block.

## Conclusion

The Rspamd GPT Plugin integrates OpenAI's GPT models into Rspamd, enhancing its spam filtering capabilities with advanced text processing techniques. By configuring the options above, users can customize the plugin to meet specific requirements, thereby enhancing the efficiency and accuracy of spam filtering within Rspamd.
