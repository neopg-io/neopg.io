---
title: "No daemons in NeoPG"
date: 2017-12-13
menu:
  main:
    parent: Blog
tags: [legacy]
---

NeoPG will not have long-running daemons.  This article explains why.
<!--more-->

## Daemons in GnuPG

GnuPG 1 is a simple binary that runs standalone.  GnuPG 2 has a
different architecture that includes several long-running daemon
processes that support it:

* `gpg-agent` supports all secret-key operations for OpenPGP and
  S/MIME. It is a critical part of GnuPG 2.
* `dirmngr` does all network accesses and can cache some of the
  results, such as certificate revocation lists (CRLs).  Dirmngr is
  necessary for OpenPGP key retrieval and X.509 certificate validation.
* `scdaemon` supports smart cards and other hardware tokens.

### GnuPG Agent

The `gpg-agent` daemon was introduced to provide process isolation for
the bytes in secret keys.  This can reduce the risk of information
leaks in bugs like [http://heartbleed.com/](Heartbleed).[^1] Although
heartbleed is a single incidence so far, it had serious impact, and it
would have been prevented by process isolation in the spirit of
`gpg-agent`, so the approach has some value.

[^1]: GnuPG, as many other cryptographic software (including NeoPG), tries to scrub sensitive data from memory after use, and calls `mlock()` to prevent it from being written to swap space.  Heartbleed showed that this is insufficient in some cases.

However, `gpg-agent` is not limited to process isolation for secret
key operations, it is also a long-running daemon providing other
services, such as a secret key cache to avoid entering the passphrase
multiple times within a short time.

The `gpg-agent` process runs with the same privileges as the other
user processes and can be debugged by any attacker with code execution
permissions for that user.  Such an attacker can mount a range of
other attacks as well, including keyloggers.  However, this shows that
process isolation is a narrow defense-in-depth technique.

Over time, `gpg-agent` has accumulated the following functions
(`agent/command.c`):

* `GETEVENTCOUNTER`
* `ISTRUSTED`
* `HAVEKEY`
* `KEYINFO`
* `SIGKEY`
* `SETKEY`
* `SETKEYDESC`
* `SETHASH`
* `PKSIGN`
* `PKDECRYPT`
* `GENKEY`
* `READKEY`
* `GET_PASSPHRASE`
* `PRESET_PASSPHRASE`
* `CLEAR_PASSPHRASE`
* `GET_CONFIRMATION`
* `LISTTRUSTED`
* `MARKTRUSTED`
* `LEARN`
* `PASSWD`
* `INPUT`
* `OUTPUT`
* `SCD`
* `KEYWRAP_KEY`
* `IMPORT_KEY`
* `EXPORT_KEY`
* `DELETE_KEY`
* `GETVAL`
* `PUTVAL`
* `UPDATESTARTUPTTY`
* `KILLAGENT`
* `RELOADAGENT`
* `GETINFO`
* `KEYTOCARD`

The feature list shows that `gpg-agent` includes trust management
(`LISTTRUSTED`, `MARKTRUSTED`), has intimate knowledge of OpenPGP
import/export formats (`IMPORT_KEY`, `EXPORT_KEY`), interposes smart
card operations (`SCD`, `KEYTOCARD`), and is responsible for any user
interaction over pinentry.  It can also be used as a drop-in
replacement for `ssh-agent`.

That `gpg-agent` is an intermediate for `pinentry` is often
inconvenient.  For each conneciton, `gpg-agent` has to be informed
about the terminal and GUI capabilities of the calling application
(via environment variables such as TTY, XDISPLAY, LC_LANG etc).  The
information is passed on to `pinentry`, which then tries to open a
dialog accessible for the user.  This does not work in all
configurations (see for example [GnuPG
#2818](https://dev.gnupg.org/T2818)).  Some applications had to write
wrapper scripts around `pinentry` to be able to implement their own
passphrase management.  Since GnuPG 2.1.12, GnuPG supports a special
loopback mode that punts all interactions back to the calling
application, effectively circumventing the whole architecture.

Process isolation for secret key material can be done in a
lightweight, short-lived process.  It does not need to be implemented
in a long-running daemon that accumulates unrelated tasks.  Using a
leaner design allows to implement stronger process isolation than just
POSIX fork/exec, depending on the operating system
(e.g. [seccomp](https://en.wikipedia.org/wiki/Seccomp) on Linux).
This would give higher confidence that the secret key material can't
leak to other processes.

As `gpg-agent` is a mandatory part of GnuPG, it is also a common
source of frustration for distributions and users.  In Debian, it has
been patched to allow [better power
management](https://dev.gnupg.org/T1805).

### Directory Manager

The directory manager (`dirmngr`) was originally developed to isolate
some S/MIME related network functions.  Making it a stand-alone
package had some other advantages, too (it could be developed without
requiring copyright assignments from contributors)

Before GnuPG 2.1, OpenPGP did not use `dirmngr`, but stand-alone
keyserver helper processes to isolate network access.  With GnuPG 2.1,
`dirmngr` moved into GnuPG proper, and the keyserver helper processes
were moved into `dirmngr`.

Putting all network access into an isolated process again helps to
limit the risk of Heartbleed-like exploits.  Although GnuPG is not
supposed to have secret key material in its process (that has been
moved into `gpg-agent`), there may be other sensitive data like
plaintext in it.

However, GnuPG is rarely used alone.  Commonly it is used by a larger
application such as Thunderbird with Enigmail, and these applications
usually do have network access, too, and they keep copies of the
sensitive material.  Also, most of the data retrieved from the network
by `dirmngr` is passed on to `gnupg` unfiltered and processed there,
so the isolation is imperfect.

Again, `dirmngr` has attracted more functions over time, implementing
high-level functions rather than low-level building blocks:

* `DNS_CERT`
* `WKD_GET`
* `LDAPSERVER`
* `ISVALID`
* `CHECKCRL`
* `CHECKOCSP`
* `LOOKUP`
* `LOADCRL`
* `LISTCRLS`
* `CACHECERT`
* `VALIDATE`
* `KEYSERVER`
* `KS_SEARCH`
* `KS_GET`
* `KS_FETCH`
* `KS_PUT`
* `GETINFO`
* `LOADSWDB`
* `KILLDIRMNGR`
* `RELOADDIRMNGR`

The main functions are X.509 certificate retrieval and validation
(including CRLs and OCSP), and OpenPGP key retrieval over HKP, HTTP,
DNS, WKD and LDAP.  Also, `dirmngr` can consult a "software database"
(`LOADSWDB`) to check if the installed version of GnuPG is current.
It is also planned to use `dirmngr` to refresh keys in the background
regularly.

Because of specific design decisions, `dirmngr` includes its own
implementation of a HTTP client library, its own DNS resolver, and
implements access to the Tor network.

### Smart Card Daemon

GnuPG supports OpenPGP smart cards and other hardware tokens such as
Gnuk, using the included `scdaemon`.  This daemon will try to get
exclusive access to the token using libusb and a built-in CCID driver,
and fall back to `pcscd`.  If GnuPG is the only smart card user on a
system, and the hardware supports the CCID interface, the CCID driver
with `scdaemon` provides a good user experience.  However, users with
hardware security tokens usually want to use it with other
applications, too, and then the problems start.  Users often
experience problems when `scdaemon` and `pcscd` conflict with each
other.

`scdaemon` includes the following operations:

* `SERIALNO`
* `LEARN`
* `READCERT`
* `READKEY`
* `SETDATA`
* `PKSIGN`
* `PKAUTH`
* `PKDECRYPT`
* `INPUT`
* `OUTPUT`
* `GETATTR`
* `SETATTR`
* `WRITECERT`
* `WRITEKEY`
* `GENKEY`
* `RANDOM`
* `PASSWD`
* `CHECKPIN`
* `LOCK`
* `UNLOCK`
* `GETINFO`
* `RESTART`
* `DISCONNECT`
* `APDU`
* `KILLSCD`

Unsurprisingly, there is some overlap with `gpg-agent`, which
implements similar functionality in software, and in fact acts as an
intermediate to `scdaemon` for common operations (also, `gpg-agent`
can start `scdaemon` on demand).

In this case, I believe process isolation was chosen to provide more
robustness against failure.  It is a common user experience that
`scdaemon` has to be terminated to release the hardware device, or
because it gets stuck.

### npth

Because these daemons are long-running and can be used concurrently
from parallel `gnupg` processes, they need to be multithreaded.  This
is done using `npth`, a cooperative multi-threading library similar to
GNU Pth.  This adds considerably complexity, and has been the source
of several race conditions.

### libasssuan

All these daemons are accessible through a line-based inter-process
communication protocol provided by `libassuan`.  The interface can be
used over a socket or pipe transport.  To support this on Windows, and
in the cooperative multi-threading environment of `npth`, there is
quite a bit of platform support in `libassuan`, `libgpg-error` and
`npth`.

## NeoPG has no daemons

In NeoPG, we have not yet removed or replaced the process isolation
and the inter-process communication protocol.  However, we have
greatly simplified it by starting the processes not as long-running
daemons, but as short-lived helper processes, that exit once they are
done.

This allows us to run the processes in pipe mode instead of socket
mode[^2], and to remove support for concurrency.  In fact, NeoPG does not
not use `npth` anymore, and its libassuan implementation is greatly
slimmed down.

[^2]: On Windows, pipe mode uses a socket pair, because they provide an interface closer to POSIX file descriptors.

In the future, we will move more functions of these processes back
into the main binary (or rather, into the `libneopg` library).
Finally, we will investigate how to make process isolation stronger
and more lightweight for the remaining sensitive functions, to provide
some protection against attacks like Heartbleed.

For smart card access, we plan to support only pcscd, which already is
a daemon.  Although we don't know yet about all the issues involved in
smart card support, there is no reason why NeoPG should behave or
require something different from any other application.

The main disadvantage is that we do not make use of the caches in this
configuration, and we have not replaced the caches with alternative
mechanisms yet.  We believe that moving caches closer to the
application will, in most cases, be an adequate alternative.  In some
cases, there are other alternatives.  For example, the major desktop
environments provide passphrase caching on their own.  And long-lived
caches such as downloaded CRLs, have to reside on permanent storage
anyway, and only require filesystem-level locking.
