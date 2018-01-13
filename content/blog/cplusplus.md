---
title: "C++ as a migration path for legacy C code"
date: 2018-01-13
menu:
  main:
    parent: Blog
tags: [legacy]
---

NeoPG is written in C++, while GnuPG is written in C.  This article explains why.
<!--more-->

## C++ is the better C

C++ has a long history, but the original motivation was to provide an
alternative to C with better encapsulation and type safety.  For
better or worse, the language features are designed to let the
programmer control runtime costs.  Much can and has been said about
the advantages and disadvantages of C++.  So let me put this up front:

__If you are starting from scratch, and C compatibility is not a
concern, then my first recommendation would be to not use C++ but some
other language.__

For NeoPG, here are some reasons why C++ is not a terrible choice:

* GnuPG is written in C, and C++ is mostly compatible with legacy C code.
* C++ is a mature language, with excellent tooling and library support.
* The problems of C++ are well-known, and can be avoided.
* The legacy code base can benefit a lot from the C++ standard libraries and classes.
* C++ is an evolving language that keeps improving.

Here comes my not so great attempt at defending C++.

### C++ is compatible with C

Converting the legacy code base (490,000 lines of code) to C++ was a
straightforward, mechanical task that took only a couple of hours:

* All `*.c` files were renamed to `*.cpp`.
* All occurences of C++ keywords in named entities (variables, member
  and function names) were replaced.  Specifically, these were:
  `public`, `namespace`, `not`, `template`, `class` and `new`.
* Some enum definitions had to be moved outside a struct.
* Some type annotations were modified to fix invalid comparisons etc.
* The compiler flag `-fpermissive` was added.
* Several hundred "invalid conversion"-warnings were fixed semi-automatically using a 200-line Python script.
* Some other warnings were fixed manually.

The immediate benefit is stricter type checking from the compiler.  

### C++ is a mature, but evolving language

We have two strong free software implementations of the C++ language,
GCC and Clang.  Not many other languages can claim that.  There are
several great IDEs, code formatter, linter, static code analyzer, and
debugger available for C++ development.[^1] Many people are familiar
with it.

[^1]: I say this even though the C++ language is very difficult to parse, so code refactoring and IDE support is not easy to implement. It's not a great language in that respect, but it is old and important enough that people went to the trouble and implemented the necessary tools despite the difficulties, which is sufficient from a pragmatic point of view.

The Standard Template Library (STL) is a solid extension of the
language that, in many cases, removes the need for manual memory
management.

Boost is the inofficial second standard library, that provides many
further functions that are commonly needed, such as more string
algorithms.  There are also some other useful libraries in Boost, such
as Boost::locale.  Some Boost libraries are not so great or even
outdated, but Boost is modular, so you pick and choose.

There are several good crypto libraries written in C++, notably
CryptoPP and Botan.  NeoPG uses Botan, which is newer.  We will
discuss Botan in a future blog post, but its value for NeoPG can
hardly be overstated.

And C++ is evolving.  With C++11, C++14 and C++17, the community is
showing that it is aware of the problems and is making a serious
attempt at fixing them.  In comparison, C and POSIX are set in stone,
possibly forever.

### The problems are well-known

Yes, the order of static global initialization is not defined.

Yes, destructors are not exception safe.

Yes, you should not have external types with the same name in
different compilation units (I got bitten by that, because every
component in GnuPG has a `struct option`, and once I added STL types
to those, NeoPG crashed -- I had to rename them to `struct
dirmngr_options`, `struct gpg_agent_options` etc.).

Yes, [anything about templates], [anything about references and
const], etc., etc.

Many of these issues are well-known or can be easily researched.  They
have been documented many times, and can be avoided.  It's a
manageable concern.  In particular, it is a known risk, while a new
language comes with unknown risks.

### Legacy code can benefit a lot from C++

There are many areas in the legacy code base that obviously benefit
from C++.

* There is a lot of string manipulation, which benefits from the STL
  string type.
* Similarily, there are many other containers, which benefit from
  their corresponding STL types (vectors, maps, pairs, etc).
* In general, GnuPG does a lot of manual memory management.  For
  example, there are several dozens of implementations of
  automatically growing buffer.  Many of them ad-hoc, based on
  `realloc`, some generalized (`membuf`, `iobuf`, `strlist`,
  `es_fopenmem`).
* There is also a lot of memory management for higher level objects,
  with corresponding create and free functions.  In C++, these would
  simply be constructors and destructors.
* The pipe/filter abstraction to read and write OpenPGP packets is
  explicitely object-oriented.  It is much easier to capture this
  design in C++.

## A language for NeoPG

There are many programming languages, and I believe in picking the
right tool for the job.  In the case of NeoPG, the priorities were:

* Support for strong cryptography.
* Compatibility with C application developers.
* Convert legacy code quickly.
* Tool support for QA.

Everything else is, at this point, a secondary concern.  The [Sequoia
Project](https://sequoia-pgp.org/) uses Rust, and I envy that.  But
the first thing they had to do was to wrap an existing C crypto
library (they choose libnettle), because there is no high quality
crypto library for Rust yet.  That is their challenge.

My challenge will be to stay focussed on the parts of C++ that are
actually helpful, and not get bogged down by the rest.