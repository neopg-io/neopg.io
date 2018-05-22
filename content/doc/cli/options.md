---
title: Global Options
date: 2018-05-22
menu:
  main:
    parent: Docs
---

Global options.
<!--more-->

## Color

```text
  --color WHEN=auto           colorize the output (auto, always, or never)
```

`neopg` uses colors automatically by default, if the output is sent to
a terminal. If the output is redirected to a file or anything other
than an interactive terminal, color will not be used. This automatic
behaviour can be disabled by choosing `always` or `never`.

### Example

```text
$ neopg --color=never ...
```

## Logging

```text
  --log-level LEVEL=warning   set minimum log level (trace, debug, info, warning, error, critical, or off)
```

Log messages are sorted in severity levels, with increasing urgency
from `trace` to `critical`. By default, only warnings, errors, an
critical messages are shown. With `--log-level` a different minimum
severity level can be selected.

```text
  -v,--verbose                enable more logging (can be used multiple times)
```

With `-v` or `--verbose`, a different logging level can be selected
easily. One occurence of `-v` selects `info`, two occurences select
`debug`, and three or more occurences select `trace` as minimum
severity level.

