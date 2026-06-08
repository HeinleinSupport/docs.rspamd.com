---
title: Mime types modules
---


# Rspamd mime types module

This module is intended to do some mime types sanity checks. That includes the following:

1. Checks whether mime type is from the `good` list (e.g. `multipart/alternative` or `text/html`)
2. Checks if a mime type is from the `bad` list (e.g. `multipart/form-data`)
3. Checks if an attachment filename extension is different from the intended mime type
4. Checks for archives content (rar and zip are supported) and find certain bad files inside
5. Checks for some other bad patterns commonly used by spammers, e.g. extensions hiding (e.g. `.pdf.exe`)

## Configuration

`mime_types` module reads mime types map specified in `file` option. This map contains binding

```
type/subtype score
```

When score is more than `0` then it is considered as `bad` if it is less than `0` it is considered as `good` (with the corresponding multiplier).
When mime type is not listed then `MIME_UNKNOWN` symbol is inserted.

**Important:** the module disables itself if the `file` (map) option is not set in the configuration.

If `regexp` is set to `true` (default: `false`), the `file` map is loaded as a regexp map instead of a plain key-value map.

`extension_map` option allows to specify map from a known extension to a specific mime type:

~~~hcl
extension_map = {
  html = "text/html";
  htm  = "text/html";
  txt  = "text/plain";
  pdf  = "application/pdf";
}
~~~

When an attachment extension matches the left part but the content type does not match the right part then symbol `MIME_BAD_ATTACHMENT` is inserted.

For extensions not explicitly listed in `extension_map`, Rspamd uses a built-in full MIME-type database. Mismatches against those automatically-matched entries use the `other_extensions_mult` multiplier (default: `0.4`) instead of the full `1.0` applied to explicit `extension_map` entries.

## Archives support

Since 1.3, this module supports archives processing (rar and zip formats) and can check files inside archives. There are additional options added for more precise archives checks, for example, a special symbol for nested archives.

### Archive exceptions

Some file formats are internally ZIP-based (e.g. Office Open XML, OpenDocument) and should not be treated as archives. The `archive_exceptions` table lists extensions that are whitelisted from archive content scanning:

~~~hcl
archive_exceptions {
  docx = true;
  odp  = true;
  ods  = true;
  odt  = true;
  pptx = true;
  vsdx = true;
  xlsx = true;
}
~~~

### Default configuration

Here is a representative excerpt of the default configuration with comments. The full default tables are defined in the source (`mime_types.lua`) and can be inspected with `rspamadm confighelp mime_types`.

~~~hcl
# When set to true, the 'file' option is loaded as a regexp map
regexp = false;

# Multiplier applied to MIME-type mismatches from the built-in extension
# database (not from explicit extension_map entries)
other_extensions_mult = 0.4;

extension_map {
  html = "text/html";
  htm  = "text/html";
  shtm = "text/html";
  shtml = "text/html";
  txt  = "text/plain";
  pdf  = "application/pdf";
}

# Extensions that are treated as 'bad' (score multiplier shown).
# This is a representative subset; the full list includes 100+ entries
# covering executables, scripts, and dangerous Windows shell extensions.
bad_extensions {
  exe  = 1;
  jar  = 2;
  iso  = 4;
  com  = 4;   # note: score is 4 in current source, not 2
  bat  = 4;   # note: score is 4 in current source, not 2
  ace  = 4;
  arj  = 2;
  cab  = 3;
  lnk  = 4;
  scr  = 4;
  js   = 4;
  vbs  = 4;
  hta  = 4;
  chm  = 4;
  wsf  = 4;
  # ...many more — see source for the full list
}

# Extensions that are particularly penalized when found inside archives.
# Entries with low multipliers (0.1) are only checked for archives with
# a single file; they are still scored when appearing alone inside a zip.
bad_archive_extensions {
  chm  = 4;
  docx = 0.1;
  exe  = 0.1;
  hta  = 4;
  iso  = 4;
  jar  = 3;
  js   = 0.5;
  lnk  = 4;
  pdf  = 0.1;
  pptx = 0.1;
  vbs  = 4;
  wsf  = 4;
  xlsx = 0.1;
}

# Used to detect an archive inside another archive (score multiplier)
archive_extensions {
  7z   = 1;
  ace  = 1;
  alz  = 1;
  arj  = 1;
  bz2  = 1;
  cab  = 1;
  egg  = 1;
  lz   = 1;
  rar  = 1;
  txz  = 1;
  xz   = 1;
  zip  = 1;
  zpaq = 1;
}

# Extensions whitelisted from archive content scanning
# (these formats are ZIP-based but not actual archives)
archive_exceptions {
  docx = true;
  odp  = true;
  ods  = true;
  odt  = true;
  pptx = true;
  vsdx = true;
  xlsx = true;
}
~~~

## Symbols

| Symbol | Option key | Description |
|---|---|---|
| `MIME_UNKNOWN` | `symbol_unknown` | Content type is not in the mime types map |
| `MIME_BAD` | `symbol_bad` | Content type is in the bad list |
| `MIME_GOOD` | `symbol_good` | Content type is in the good list |
| `MIME_BAD_ATTACHMENT` | `symbol_attachment` | Extension/content-type mismatch |
| `MIME_ENCRYPTED_ARCHIVE` | `symbol_encrypted_archive` | Encrypted or unreadable archive |
| `MIME_OBFUSCATED_ARCHIVE` | `symbol_obfuscated_archive` | Archive with obfuscated content |
| `MIME_EXE_IN_GEN_SPLIT_RAR` | `symbol_exe_in_gen_split_rar` | Executable found inside a generic split RAR (e.g. `.001`/`.002` parts) |
| `MIME_ARCHIVE_IN_ARCHIVE` | `symbol_archive_in_archive` | Archive nested inside another archive |
| `MIME_DOUBLE_BAD_EXTENSION` | `symbol_double_extension` | Double bad extension (e.g. `.pdf.exe`) |
| `MIME_BAD_EXTENSION` | `symbol_bad_extension` | Bad attachment extension |
| `MIME_BAD_UNICODE` | `symbol_bad_unicode` | Filename contains obscured/obfuscated Unicode characters |

All symbol names can be overridden via the corresponding option key in `local.d/mime_types.conf`.

## User settings usage

From version 1.9.1, it is possible to tune this module via [Users settings](/configuration/settings). To use that, one can apply the following settings:

~~~hcl
test {
  from = "user@example.com";

  apply {
    plugins {
      mime_types = {
        bad_extensions = {
          exe = 100500,
        },
        bad_archive_extensions = {
          js = 100500,
        },
      }
    }
  }
}
~~~

## Filename whitelist

It's possible to add a regex whitelist map of filenames you want to bypass the mime_type scanning:

~~~hcl
# local.d/mime_types.conf

  filename_whitelist = "$LOCAL_CONFDIR/maps.d/mime_types.wl";
~~~

The map file should look like this:

~~~
/^hello_world\.exe$/
~~~
