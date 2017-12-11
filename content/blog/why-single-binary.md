---
title: "Why a single binary is the right thing for NeoPG"
date: 2017-12-11
menu:
  main:
    parent: Blog
tags: [legacy]
---

NeoPG only provides a single binary for everything, while GnuPG is
split up into many binaries.  This article explains why.
<!--more-->

## NeoPG is a single binary with subcommands

In NeoPG, all functions are provided by a single binary, `neopg`.
This binary provides git-style subcommands (the current version does
not support many commands yet, but the help output illustrated the
concept):

```sh
$ ./src/neopg --help
NeoPG implements the OpenPGP standard.
Usage: ./src/neopg [OPTIONS] [SUBCOMMAND]

Options:
  --help                      display help and exit
  --version                   display version and exit

command to execute (GnuPG-compatible):
  gpg2                        invoke gpg2
  gpgsm                       invoke gpgsm
  agent                       invoke agent
  scd                         invoke scd
  dirmngr                     invoke dirmngr
  dirmngr-client              invoke dirmngr-client

tools (for experts):
  packet                      read and write OpenPGP packets
  random                      output random bytes
  hash                        calculate hash function
  compress                    compress and decompress data
  armor                       ASCII-encode and decode binary data
  cat                         the beginning of a new Unix system

Report bugs to https://github.com/das-labor/neopg
```

## GnuPG has many binaries

GnuPG splits up its features into many binaries:

* `gpg` provides an OpenPGP implementation
* `gpgsm` provides a X.509 implementation
* `gpg-agent` is a long-running daemon that provides secret key operations
* `dirmngr` is a long-runnning daemon that provides network access and certificate validation and caching
* `scdaemon` is a long-running daemon that provides smart-card access
* `pinentry*` are several pinentry programs for various user interfaces
* `gpgconf` is a command-line interface to the configuration files for all of the above
* `dirmngr-client` is a helper program to talk to the dirmngr daemon
* `gpg-connect-agent` is a helper program to talk to gpg-agent or any of the other daemons (including dirmngr)
* `gpgv` is a minimized version of gpg that only contains code for signature verification
* `gpg-zip` is a helper script to deal with encrypted and/or signed archive files
* `gpgtar` is a minimal replacement for gpg-zip that does not require a POSIX shell
* `gpgscm` is a scheme interpreter used to run the test suite
* `gpg-error-config`, `libassuan-config`, `libgcrypt-config`, `npth-config` are helper programs for locating the respective libraries, because the GnuPG project does not support pkg-config
* `gpg-error` is a small helper program for developers to interpret generalized GnuPG error codes
* `dumpsexp`, `gpg-error`, `gpgparsemail`, `kbxutil`, `watchgnupg`, `mpicalc` and `hmac256` are helper programs for developers

GnuPG also installs a couple of helper binaries in `libexec`:

* `gpg-check-pattern` can check a passphrase against a file with bad patterns
* `gpg-preset-passphrase` allows to set a passphrase in the cache of `gpg-agent`
* `gpg-protect-tool` is a tool to manage secret keys, and is sometimes vital if the public key has been lost
* `gpg-wks-client` is a tool to communicate with WKS server (for WKD support)

These are a lot of binaries, sometimes with inconsistent naming, and
inconsistent documentation and command line interface.  In fact,
communication with the long-running daemons requires in-depth
knowledge of the text-based Assuan protocol.  It is difficult to
discover these tools, and it can be difficult to use them.  Some are
intended for automatic use behind the scenes, but are often elevated
to rescue or debug tools, too.

## An old rule inconsistently applied

> Make each program do one thing well. To do a new job, build afresh rather than complicate old programs by adding new "features".
> -- [Doug McIlroy](https://en.wikipedia.org/wiki/Doug_McIlroy) in the [Bell System Technical Journal from 1978](http://emulator.pdp-11.org.ru/misc/1978.07_-_Bell_System_Technical_Journal.pdf)

This rule is often repeated in the community like a mantra.  But it is
less often applied consistently, or with any rigor.  Notably, GnuPG
has close to 400 command line options.  Some of the features in GnuPG
have been split off into `gpg-agent` and `dirmngr`, but still these
tools accumulate unrelated features, mostly based on implementation
convenience rather than on usability or reusability.  Also, the
maintenance and usability costs are often ignored by advocates of this
doctrine, which comes from a time were computer systems, even big ones
like UNIX, were much simpler.

## Advantages of a single binary

The features contained in a single binary are much __easier to
discover__ through the help output.  The common subcommand pattern
also encourages a __consistent user interface__.  It is __efficient__,
because all functions share the same code base, and there is much less
boilerplate in the source code.

An important advantage is also that all the code in the single binary
is always consistent with itself.  __Version mismatches are
impossible__ between the different components, because they are always
accessed and replaced as a whole.  Installing and updating becomes
much simpler.  There is a lot of duplicated boilerplate code in GnuPG
just to check version numbers of the components when building and
communicating.  All this boilerplate just goes away with a single
binary.

It also makes it __easier for distributions__, especially
privacy-aware distributions such as those running off an USB stick.
The next logical step is to embed file resources into the binary, too,
allowing everything that is required to use NeoPG to be contained in a
few files (the `neopg` binary and the `libneopg` library for other
applications based on NeoPG).  Linking `libneopg` statically into
`neopg` could possibly lead to a single file distribution.

A single binary based on a single library is also much __easier to
port__ to other, sometimes exotic architectures.  For example, mobile
phone operating systems are identifying processes with Apps, and may
place restrictions on the composition of such an App.

A single binary does not prevent a multi-process architecture.
Currently, NeoPG currently invokes itself with subcommands to start
`dirmngr` and `gpg-agent` servers on demand.  How many processes are
used during an operation is a question independent of the number of
binary files.

So, for now, NeoPG will follow a much more consolidated and integrated
design, and follow the benefits of it.
