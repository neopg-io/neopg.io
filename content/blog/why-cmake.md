---
title: "Why CMake is better than Autoconf for NeoPG"
date: 2017-12-01
lastmod: 2017-12-06
menu:
  main:
    parent: Blog
---

NeoPG uses CMake instead of Autoconf.  This article explains why.
<!--more-->

The build system has the following responsibilities:

* Track the dependencies between the source and object files and
  regenerate only the object files for the modified sources.
* Integrate with other libraries and tools for translation, etc.
* Find the right compiler flags and definitions for the target system.

## Dependency Tracking

For all practical purposes, dependency tracking is a solved problem.
You list all source files used by a library or binary, and all
libraries used by a binary, and the rest is mostly automatic.
Autoconf and CMake do this equally well.

## Integrate With Thirdparty Libraries

Both Autoconf and CMake support pkg-config, and both have a rich
library of example scripts.  There are some cultural differences.  For
example, Autoconf integrates very well with other GNU packages, such
as gettext.  The CMake community is more diverse, and more pragmatic.
The gettext integration in CMake can be as simple as finding and
invoking xgettext.  From my experience so far, both seem to be about
equal.

### Aside: pkg-config
It's worth noting here that the libraries of the GnuPG project do not
integrate with pkg-config, but have their own ${LIBRARY}-config
scripts that works similarily.  We wrote those back in the days when
cross-compilation was not well supported by pkg-config.  From the
documentation, its cross-compilation support is still not great, and
relies on a wrapper script.  But there are many pkg-config users, so
it doesn't make sense anymore to not follow the mainstream here.

## Target Configuration

Finding the right compiler flags can be difficult.  There are many
potential portability problems in source code, some very obscure.
Changes in compilers and operating systems can uncover hidden
assumptions about the target platform in the source code (as GCC 4.4
did with type punning and the strict aliasing rule).  Sometimes there
are bugs in the system libraries that have to be worked around.  For
example, XCode 9.1 requires _DARWIN_C_SOURCE to be defined just to
include standard system headers.  The best strategy is to write
"normal" code, define as few flags as possible, and hope for the best.
Then deal with errors as they come in.  Some interfaces are more
broadly supported than others, and using C++ has helped a lot.

Every build system tries to solve the problem in two steps: First, an
attempt is made to put as much knowledge about potential target
systems into a configuration script as possible ahead of time.  Then,
the user downloading and compiling the software at the other end runs
the configuration script on the target platform.  The script tries to
determine a valid combination of flags for that specific platform.
However, there is a chicken-and-egg problem when it comes to running
the configuration script on the target platform.

### Autoconf

Autoconf is a set of scripts written in m4, which is an ancient macro
language from Kernighan & Richie, the authors of the original Unix and
the programming language C.  These m4 scripts are run by the
maintainer of the software package, so m4 is only required during
development.  The output of the package is a POSIX-compatible shell
script, which runs on any Unix-like system (such as GNU/Linux, *BSD
and MacOS).  The shell script is packed with arcane knowledge about
Unix-like operating systems, and that knowledge is applied in a
fine-grained manner.  For example, autoconf will test the byte-size of
standard types of the C programming language (char, int, long, etc.)
individually.  The result is a very flexible system, that often works
even with previously unknown Unix-like systems.

Autoconf is very slow.  Shell is not fast to begin with, and autoconf
will execute many utility programs and compile many small test files
to do its work.  As GnuPG is split up into several packages, and each
package is self-contained, autoconf will also do a lot of repetetive
work.  For example, it will test four times if "unsigned long" is 8
bytes.  You can say about autoconf what you want, but it is damn sure
about the size of "unsigned int", again, and again, and again.

Also, some systems do not have a POSIX-compatible shell, notably
Windows, and some systems are cross-compiled (Android, iOS).  For
these systems, the value of the Autoconf-generated scripts is
diminished.  It is much more important to integrate with the native
toolchains in these cases.

Autoconf was extended over the years with a set of related tools:
Automake for automatic Makefile generation, libtools for shared and
static library support, and so on.  These tools increasingly are just
responsible for installing and updating a lot of boilerplate into the
source code repository.  As the released source code archive must run
independently on any possible target system, all arcane knowledge
about these systems must be contained in every release of the software
using autoconf.  Any update to these scripts (for example to support a
new platform) causes a cascade of updates in all autoconf-using
packages that people want to compile for, and requires a new release
to be made available to the users.

In a well-maintained package, upgrading the included m4 files can be
automated (and maintainers are encouraged to do so), but users are
discouraged from doing it themselves: Partly because it requires these
"special tools" to be installed, partly because the process is very
fragile and can lead to subtle and not so subtle errors, which can not
be debugged by the maintainer.  GnuPG in particular has frozen and
patched (read: forked) the contained version of libtool and is
hesitant to change it (apparently to preserve support for the Windows
cross-building toolchain), abandoning support for niche-platforms and
thus essentially defeating the purpose of using a portable build
system in the first place:

> We don't want to update libtool, thus we [apply] a simple libtool patch supplied by IBM.
> -- (libgpg-error [0b192cff](https://dev.gnupg.org/rE0b192cff772bd416dc85b8140b9eb0d52e4175dd), 2013-10-22)

> Add hack to have different names for 64 bit Windows DLLs. [...] We
> need to stick to libtool 2.4.2 anyway, thus we take the easy way and
> hack libtool instead of adding "-release 6" to the Makefile.
> -- (libgpg-error [ca46b9a7](https://dev.gnupg.org/rEca46b9a7bccb2eab085fc45722ffca1210f48223), 2013-06-17)

For libassua, libgcrypt, libksba, and npth, the version of libtool in
libgpg-error is now the upstream source, and patches are manually
applied to all these libraries when deemed necessary.  See also:
[T1608](https://dev.gnupg.org/T1608), [Gentoo
383865](https://bugs.gentoo.org/383865).

### CMake

CMake solves the problem differently.  It does have a scripting
language (with its own somewhat idiosyncratic syntax), but the scripts
are interpreted by a binary, which must be installed on the target
system beforehand (this is the equivalent to the POSIX shell for
Autoconf).  Because installation of CMake is required anyway, CMake
can also include a bunch of builtin functions and a script library
that covers 80% of what most users need.

Because CMake is a compiled binary, it runs much faster than Automake.
Because it comes with batteries included, the amount of scripting
required is minimal.  It is widely adopted by the community, and it
integrates nicely on Windows, Android and iOS SDKs, in addition to
plain old Unix-like systems.

## The Numbers

**Boilerplate** The five core GnuPG packages (gnupg, libassuan,
libgcrypt, libgpg-error and libksba) ship collectively with well over
200 boilerplate scripts from the autoconf build system, while NeoPG
ships with a total of four (4), all of which are optional.  These four
scripts in NeoPG add features that neither Autoconf nor CMake provide
(Code Coverage reports, automatic changelog generation, source code
formatting, and release tagging with Git).

**Speed** CMake generates the Makefiles about 40% faster, although
this is not a fair comparison, because less features are tested in any
version of NeoPG.  More importantly, it regenerates the Makefiles
after a modification an order of magnitude faster than that, a caching
optimization that Autoconf doesn't do.  This makes development much
snappier.

**Programming** CMake is not the prettiest scripting language ever
designed, but m4 is particularly obscure and has some very problematic
quoting rules.  CMake comes with many more useful built-in functions.

**Cruft** Autoconf and related tools support some very obscure
platforms.  Libtool for example is generic to a fault.  Developers are
not familiar with many different shared and static library
implementations, so they don't recognize the abstractions in libtool
and can not use them correctly to support those platforms.  I have
seen many errornous usages of libtool (specifically the .la files)
that caused more problems than they solved.

**Platform Support** CMake supports Windows, Android SDK and iOS.
'nuff said.

**Integration** Both support pkg-config to find third-party libraries,
and both have a huge library of scripts to support common use cases.

**Maintainability** I don't have long-term experience with CMake, but
maintaining Autoconf-based build systems is a huge time sink.  It
requires frequent updates, and there have been a lot of breaking
changes over the years.  From the documentation, CMake has better
versioning support and breaking changes are introduced with
configurable policy flags.

## Summary

CMake is a very easy choice to make, because it accomplishes the main
goal, broad platform support, with a minimum of maintenance effort and
boilerplate.  It requires the user to download and install CMake, but
it is a very common build system and well supported, so this is not a
major issue.
