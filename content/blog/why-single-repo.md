---
title: "Why a single repository is good for NeoPG"
date: 2017-12-07
menu:
  main:
    parent: Blog
tags: legacy
---

NeoPG combines all code in a single repository, while GnuPG uses many
repositories.  This article explains why.
<!--more-->

## GnuPG has many components

In NeoPG, all the source code is in a single repository.  This is in
stark contrast to GnuPG, which splits up the source code of the whole
into several different repositories:

* npth, a cooperative threading library (successor of GNU pth)
* libgpg-error, a shared library for error values, currently evolving to a platform support library AKA libgpg-rt
* libassuan, a remote procedure call library based on a simple text-protocol that supports sockets and pipes
* libksba, an ASN.1 parser and X.509 certificate support library
* libgcrypt, a library of elementary cryptographic functions, entropy sources and secure memory allocation
* gnupg, a group of binaries implementing OpenPGP and X509, among other things
* pinentry, a couple of binaries supporting passphrase entry in different desktop environments
* gpgme, a library to run gnupg and other binaries from an application
* ntbtls, a TLS library based on PolarSLL (under development)

This split puts a high burden on users, developers, and distributions.
Each software package must be maintained individually.  There is a
significant amount of code duplication across these packages.  And
because the components can be upgraded independently of each other,
spurious warnings or errors can occur that are frustrating users.

For example, a user who tries to install the packages without knowing
about shared library load paths and ldconfig can run into problems:
[Ubuntu
Forum](https://askubuntu.com/questions/846957/gpg-fatal-libgcrypt-is-too-old-need-1-7-0-have-1-6-5),
[GnuPG
Users](https://lists.gnupg.org/pipermail/gnupg-users/2016-November/057049.html),
[Arch Linux Forum](https://bbs.archlinux.org/viewtopic.php?id=215903).

Or a user who doesn't know what the right dependencies are can run
into problems:
[Reddit](https://www.reddit.com/r/linuxquestions/comments/51kng6/problems_installing_libassuan243_required_for/),
[GnuPG
Users](https://lists.gnupg.org/pipermail/gnupg-users/2014-December/051807.html),
[Ubuntu](https://askubuntu.com/questions/681041/trying-to-compile-gnupg-from-source),
[Ubuntu again](https://ubuntuforums.org/showthread.php?t=2297364).

Build instructions are shared in
[notes](https://gist.github.com/mattrude/3883a3801613b048d45b) and
passed around like trade secrets.  To make it easier for me, I wrote a
small script called `speedo.mk` that downloads, configures and builds
all required packages and installs them into a single directory.
Because this was such a repetetive process, I used template meta
programming in GNU Make.  Those were the days!  Today, speedo.mk is
part of the GnuPG distribution and recommended in the README.

## Almost all GnuPG components are used internally

Not all of this overhead and complication is necessary.  For most
of these libraries, there are no other users but GnuPG.[^1]  npth,
libgpg-error, libassuan and libksba are internal implementation
details of GnuPG that the user needs to know nothing about.

[^1]: Kleopatra, the certificate manager in KDE, uses libassuan to provide a UI server for key and file operations (GPA implements the same interface). This is used by gpgex, the Windows File Explorer extension.  All of these uses require GnuPG, and were developed for GnuPG-related projects.

And GPGME, which is the public interface to GnuPG for applications,
requires GnuPG to work.  Distributing it without GnuPG in a standalone
package serves no meaningful purpose from a technical point of view.

## Libgcrypt is the only component of GnuPG used by other projects

This leaves us with libgcrypt.  Libgcrypt was split from GnuPG 1 to
provide a reusable crypto-library for the GNU system.  And in fact, it
was used by GnuTLS for some time.  But in 2011 [GnuTLS switched to
libnettle](http://lists.gnu.org/archive/html/gnutls-devel/2011-02/msg00079.html),
for performance reasons, but also because libgcrypt had, from their
perspective, an unnecessary dependency on libgpg-error.

Today, libgcrypt is still is used by a couple of other packages, some
of them powerhorse applications of the free software community.  I
checked with `apt-cache rdepends libgcrypt20` on Ubuntu 16.04, and
found the following:

* [Wireshark](https://code.wireshark.org/review/p/wireshark.git) uses
  libgcrypt extensively to support cryptography in network protocols.
* [systemd](https://github.com/systemd/systemd) uses it in two major
  areas: First, it uses it to implement DNSSEC and OPENPGPKEY in its
  resolver.  Second, Lennart Poettering's brother Bertram implemented
  secure forward sealing in his doctoral thesis, so he uses libgcrypt
  primitives to implement his own pseudo number generator ([LWN
  article on Forward Secure
  Sealing](https://lwn.net/Articles/512895/)).  Systemd uses mostly
  the hashing and MPI primitives, and switches off secure memory handling.
* [libotr](https://bugs.otr.im/lib/libotr/commits/master) uses it
  internally and exposes `gcry_error_t` in its public interface.
  libotr development has stalled since 2014, with a security update in
  2016.
* [GNOME keyring](https://github.com/GNOME/gnome-keyring.git) uses
  libgcrypt extensively.
* [GNUnet](https://gnunet.org/) uses libgcrypt extensively.
* [lightdm](https://github.com/davvid/lightdm) uses libgcrypt for its
  secure memory allocation.
* [Weechat](https://weechat.org/), [VLC](http://www.vlc.de/), [Percona
  XtraBackup](https://github.com/percona/percona-xtrabackup),
  [Abiword](https://github.com/AbiWord/abiword.git),
  [Aircrack-NG](https://www.aircrack-ng.org/),
  [Amarok](https://amarok.kde.org/de) and about two dozen more packages.

So, it makes sense for libgcrypt to be managed as a standalone package.

## Pinentry is special

Asking for a passphrase is difficult to do, because there is no
standard interface for that.  This means that the implementation will
be different from operating to operating system, and also from use
case to use case.  Batch processing on the server, mobile phones,
desktop environments, all require different solutions.  This is a good
reason to keep passphrase dialog implementations (except for basic
terminal support) outside the main code base.  We will take a closer
look at pinentry in a future blog post.

## NeoPG consolidates all necessary components

The first thing I did for NeoPG was to combine all components of GnuPG
in a single repository (currently in a subdirectory `legacy`).
Immediately, this makes NeoPG easier to download, build, and modify.
But this was also done in anticipation of a major refactorisation of
the code base.

I want to replace libgcrypt with another crypto library, Botan.
Having a copy of libgcrypt in the tree helps with this restructuring,
as it allows changes at any level of the implementation.  It also
makes it easier to understand the impact of any change, and which
parts of the library are still used (directly or indirectly).  Botan
can also replace libksba and several parts of GnuPG itself (in
particular dirmngr and GnuPG's pipe/filter implementation `iobuf_t`),
so this is not just a matter of replacing the libgcrypt interface with
an alternative implementation.  I will explain the reasons for
changing the crypto library in a future blog post.

I also want to remove the need for npth, libgpg-error and libassuan
entirely.  This does not mean to replace them with alternatives that
provide the same interface.  It means restructuring the whole code
base to make these libraries unnecessary.  This is easier to do
"vertically" at all interface layers at the same time, one feature at
a time, rather than trying to remove one library entirely, and then
another.  For example, some parts of these libraries are concerned
about use of network sockets for communication.  If the use of sockets
is removed, some part in npth, some part in libgpg-error, and some
part in libassuan can be removed at the same time.  Again, this is
much easier done if all of the source code is in one place.

## Simple things should be simple

Building NeoPG should be as simple as:

```sh
$ cd build
$ cmake ..
$ make
```

Of course, there are some dependencies, notably a C++ compiler, Boost,
SQLite, gettext and Botan.  Of these, only Botan is an "exotic"
dependency that is not normally included in a distribution.  If
distributions are slow to pick up Botan 2, NeoPG will include a copy
of Botan in the source code to make it easy for people to build.[^2]

[^2]: I know that many GNU/Linux distributions require that such third-party libraries are only used from the distribution, so there will be a compile time option for that, too.
