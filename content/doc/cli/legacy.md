---
title: Legacy Commands
date: 2018-05-15
menu:
  main:
    parent: Docs
---

Invoke the legacy support.
<!--more-->

The following subcommands implement a wide range of legacy utilities
included in the GnuPG software suite.  Support for these tools is
"best effort", and there may be significant differences for a variety
of reasons.

| Subcommand       | Legacy Tool      | Comment          |
|------------------|------------------|------------------|
| `gpg2`           | `gpg2`           |                  |
| `gpgsm`          | `gpgsm`          |                  |
| `agent`          | `gpg-agent`      | (no daemon mode) |
| `scd`            | `scd`            | (no daemon mode) |
| `dirmngr`        | `dirmngr`        | (no daemon mode) |
| `dirmngr-client` | `dirmngr-client` | (deprecated)     |

## Example Output

```
$ neopg gpg2
neopg: WARNING: no command supplied.  Trying to guess what you mean ...
neopg: Go ahead and type your message ...
```

## Shortcut

If you run `neopg` through a link or binary name that ends in `gpg` or
`gpg2`, it will invoke the `gpg2` subcommand automatically, making it
unnecessary to write a shell wrapper script for that purpose.

```
$ ln -s `which neopg` gpg2
$ ./gpg2
neopg: WARNING: no command supplied.  Trying to guess what you mean ...
neopg: Go ahead and type your message ...
```

This also works for names such as `neopg-gpg2`, `neo-gpg`, `ngpg`, etc.

All of the above is also true for the other legacy subcommands
respectively.  Here is a complete list:

| Executable Suffix | Subcommand       |
|-------------------|------------------|
| `gpg`             | `gpg2`           |
| `gpg2`            | `gpg2`           |
| `gpgsm`           | `gpgsm`          |
| `agent`           | `gpg-agent`      |
| `scd`             | `scd`            |
| `dirmngr`         | `dirmngr`        |
| `dirmngr-client`  | `dirmngr-client` |
