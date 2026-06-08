---
title: SpamAssassin rules module
---

# SpamAssassin rules module

This module is intended to enable the adoption and reading of SpamAssassin rules within Rspamd.

## Overview

SpamAssassin offers a comprehensive set of rules that prove useful in low-volume environments. 
The purpose of this plugin is to integrate these rules seamlessly into Rspamd. The configuration 
process for this plugin is straightforward. All you need to do is compile your SpamAssassin rules 
into a single file and feed it to the SpamAssassin module:

~~~hcl
spamassassin {
	# Any key that is not a recognised option is treated as a path or glob
	# to a ruleset file.  The names "ruleset" and "base_ruleset" below are
	# arbitrary labels — they carry no special meaning.
	ruleset = "/path/to/file";
	base_ruleset = "/var/db/spamassassin/3.004002/updates_spamassassin_org/*.cf";
	# Limit search size to 100 kilobytes for all regular expressions
	# (default: 0, meaning unlimited)
	match_limit = 100k;
	# Those regexp atoms will not be passed through hyperscan:
	pcre_only = ["RULE1", "__RULE2"];
}
~~~

Rspamd has the capability to read several files containing SpamAssassin rules, and it
also supports glob patterns. All rules are parsed into a uniform structure, which means
that if an individual rule occurs multiple times, it may be overwritten.

The module disables itself entirely if no ruleset files are successfully loaded. At least
one valid ruleset must be supplied for the module to be active.

## Limitations and principles of work

Rspamd tries to optimize SA rules quite aggressively. Some of that optimizations
are described in the following [presentation](https://highsecure.ru/ast-rspamd.pdf).
To achieve this objective, Rspamd treats all rules as `expression atoms`. While meta 
rules are considered as **real** Rspamd rules that possess their symbol and score, 
other rules are typically concealed. Nevertheless, it is possible to specify a minimum 
score required for a rule to be treated as a normal rule.

    alpha = 0.5

The default value of `alpha` is `0.5`. By configuring it in this manner, all rules with
scores greater than `0.5` (in absolute value) will be regarded as full-fledged rules
and evaluated accordingly. Lower the value to promote more SA rules into visible Rspamd
symbols. (Note that `alpha` has no effect on rules that have no `score` line in the
ruleset file.)

The priority at which SA scores are registered in the Rspamd metric can be controlled with:

    scores_priority = 2

The default value is `2`. Because Rspamd assigns a higher priority to scores registered
later (or with a higher priority value), this default allows SA scores to override any
Rspamd built-in scores for the same symbol. Lower the value if you want Rspamd's own
scores to take precedence.

At present, Rspamd boasts the following functions:

* body, rawbody, full, meta, header, uri and other rules
* some header functions, such as `exists`
* some eval functions
* some plugins:
    + 'Mail::SpamAssassin::Plugin::FreeMail',
    + 'Mail::SpamAssassin::Plugin::HeaderEval',
    + 'Mail::SpamAssassin::Plugin::ReplaceTags',
    + 'Mail::SpamAssassin::Plugin::RelayEval',
    + 'Mail::SpamAssassin::Plugin::MIMEEval',
    + 'Mail::SpamAssassin::Plugin::BodyEval',
    + 'Mail::SpamAssassin::Plugin::MIMEHeader',
    + 'Mail::SpamAssassin::Plugin::WLBLEval',
    + 'Mail::SpamAssassin::Plugin::HTMLEval'

As of now, Rspamd does **not** offer support for network plugins and some other plugins.

Despite these limitations, the majority of SpamAssassin rules can be utilized in Rspamd, 
rendering the transition process far more seamless for users who opt to switch from SpamAssassin to Rspamd.

The overall performance of Rspamd is somewhat diminished when processing SpamAssassin rules, 
as these rules incorporate numerous inefficient regular expressions that scour through voluminous 
text bodies. Nevertheless, the optimizations implemented by Rspamd can markedly reduce the workload 
required to process SpamAssassin rules. Furthermore, if the PCRE library is constructed with 
JIT support, Rspamd can gain a substantial boost in performance. Upon launching, Rspamd indicates 
whether JIT compilation is available and issues a warning if it is not. It is worth noting that certain 
regular expressions might benefit from "hyperscan" support, which is accessible on x86_64 platforms 
as of Rspamd version 1.1.

The SpamAssassin plugin is implemented in Lua and encompasses numerous functional components. 
As a result, to enhance its speed, one may wish to compile Rspamd with [luajit](https://luajit.org),
a high-speed engine that approaches the speed of standard C. Since Rspamd version 0.9, LuaJIT has been 
enabled by default.
