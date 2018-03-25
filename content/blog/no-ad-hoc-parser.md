---
title: "No ad hoc parser in NeoPG"
date: 2018-03-24
menu:
  main:
    parent: Blog
tags: [legacy]
---

NeoPG uses formal grammars even for parsing trivial data structures,
down to individual bytes.  This article explains why.
<!--more-->

Parsing byte streams into structured data is a staple of computer
programming, and essential to many tasks.  Common examples are all
kind of files from user configuration to persistent data storage,
network communication, but also command line arguments or interactive
data input.

Some of these tasks are performed by dedicated parsers hidden beneath
high level abstractions.  `libcurl` happily takes care of HTTP data,
[`cli11`](https://github.com/CLIUtils/CLI11) is a dedicated command
line parser that also implements some of the semantics of argument and
option handling, and `Botan` does all the heavy lifting when handling
X.509 certificates.

But sometimes, no libraries exist, and the programmer has to implement
the parsing themselves.  For example, command line parsers usually
only understand numeric or string arguments, and other restrictions
have to be implemented manually.  A large software suite can have
dozens or hundreds of such coincidental parsers.  And of course, for
software that implements the OpenPGP standard, implementing a parser
for OpenPGP data formats is a core responsibility.

## Ad hoc parsing in legacy code

GnuPG, too, contains many parsers.  We can not give a complete list
here, but here are some notable examples:

* Of course, GnuPG contains an OpenPGP parser.  A lot of it is
  contained in the file `g10/parse-packet.c`, but `common/iobuf.c`
  also plays an important role.  Overall, the details of parsing
  OpenPGP is strewn all over the GnuPG source code.
* Because this main OpenPGP parser in GnuPG is not designed to be
  reusable, GnuPG contains _another_ specialized OpenPGP parser just
  for keys in local keyrings in `kbx/keybox-openpgp.c`.
* Of course, GnuPG also has parsers for its configuration files, its
  key databases, and other support files.
* GnuPG did use `libcurl` in the past, but replaced it with a
  hand-written, specialised HTTP parser in `dirmngr/http.c` at some
  time.  It also includes its own DNS client library, three URL
  parsers, and a custom argument line parser.
* GnuPG-related processes communicate over a line-based text protocol
  implemented in `libassuan`.  The library supports parsing of the 
  transports in this text protocol, but deserialization of the
  exchanged information requires many little parsers all throughout the
  code base.

All these parsers are hand-written C code, implemented with whatever
standard functions seem locally appropriate (`strtok`, `strlen`,
`strchr`, and so on).  Offsets and lengths values are often calculated
manually.  There is only one formal grammar in the GnuPG code base,
and that is for the ASN.1 parser in `libksba` (which is why it is not
included in the above list).  Here is a typical example from the
`GET_PASSPHRASE` command in the Assuan protocol of `gpg-agent`, which
takes four white-space separated arguments:

```
  cacheid = line;
  p = strchr (cacheid, ' ');
  if (p)
    {
      *p++ = 0;
      while (*p == ' ')
        p++;
      errtext = p;
      p = strchr (errtext, ' ');
      if (p)
        {
          *p++ = 0;
          while (*p == ' ')
            p++;
          prompt = p;
          p = strchr (prompt, ' ');
          if (p)
            {
              *p++ = 0;
              while (*p == ' ')
                p++;
              desc = p;
              p = strchr (desc, ' ');
              if (p)
                *p = 0; /* Ignore trailing garbage. */
            }
        }
    }
```

## Why ad-hoc parsing is bad

There are some obvious reasons why ad-hoc parsing is bad.  First, it
is very error-prone.  [Out-of-bounds
violations](https://dev.gnupg.org/rGd8bce478be3ae9e401841a77d189ef3c81ccb757), for example
when [miscounting length of
arrays](https://dev.gnupg.org/rMde4a1ea684e1591975feb801e7651309e1ee2c49), is a common
source for errors.  Also, the standard library interface functions for
string processing are [not](https://dev.gnupg.org/rG3fbeba64a8bfb2b673230c124a3d616b6568fd2f)
[easy](https://dev.gnupg.org/rG8b5a2474f21dd4f1aa2a283e2f57d75e42742af5)
[to](https://dev.gnupg.org/rG5e3679ae395e7a7e44f218f07bbe487429f1b279)
[use](https://dev.gnupg.org/rGbf43b39c05cfc68ea17483c78f14bfca6faf08eb).

But there are more fundamental problems.  Often it is hard to
understand what a parser actually does, in particular on unexpected
input.  In fact, this problem is [intractable in
general](https://en.wikipedia.org/wiki/Halting_problem).  Even if a
formal grammar is provided along with the ad-hoc parser (which is
usually not the case), it is just as hard to show that the ad-hoc
parser actually implements that grammar.  With two ad-hoc parsers for
the same language, we can not easily understand if they actually
agree.  There are three URL parsers in GnuPG.  Do they treat a domain
name like [`brave.com%60x.code-fu.org`](https://github.com/nodejs/node/issues/19468) in
the same way?  There are two OpenPGP parsers in GnuPG.  Will they
treat public keys downloaded from a keyserver in the same way?

The fine people from [LANGSEC](http://langsec.org/) explain it this
way (Kudos to [Kai Michaelis](https://panopticon.re/me/) for pointing
this out to me):

> "When input handling is done in ad hoc way, the de facto recognizer,
> i.e. the input recognition and validation code ends up scattered
> throughout the program, does not match the programmers' assumptions
> about safety and validity of data, and thus provides ample
> opportunities for exploitation. Moreover, for complex input
> languages the problem of full recognition of valid or expected
> inputs may be UNDECIDABLE, in which case no amount of input-checking
> code or testing will suffice to secure the program. Many popular
> protocols and formats fell into this trap, the empirical fact with
> which security practitioners are all too familiar."

They make this recommendation:

> "[...] the only path to trustworthy software that takes untrusted
> inputs is treating all valid or expected inputs as a formal language,
> and the respective input-handling routines as a recognizer for that
> language."

This is precisely what NeoPG aims to do.

## Parsing in NeoPG

In NeoPG, we try very hard to avoid ad-hoc parsing.  Instead, we
either use (hopefully widely used) libraries that implement
abstractions, or we use formal grammars to process untrusted input.

Here are some examples for responsibility delegation:

* NeoPG uses `CLI11` for command line parsing.  The parser in `CLI11`
  certainly qualifies as an ad-hoc parser, but it is hidden behind an
  abstract library interface.
* We also use `libcurl`, which probably contains several ad-hoc
  parsers, again hidden behind an abstract library interface.
* We also use Botan for certificates and TLS.  Again, the parsers in
  Botan are hidden behind an abstract library interface.

Clearly, delegating parsing to third parties does not prevent common
bugs in ad-hoc parsers from occuring.  However, this is not different
from other bugs in third party libraries, and there are compensating
benefits to sharing the same software with many other projects.  It's
a simple risk and benefit analysis.

However, NeoPG has also its own parsers:

* NeoPG does have its own URL parser, because [`libcurl` does not
  expose an interface to the internal
  implementation](https://github.com/curl/curl/issues/2412).  The risk
  of two different URL parsers must be compensated with [extra
  testing](https://github.com/das-labor/neopg/issues/61).
* Of course, NeoPG has a custom OpenPGP parser ([work in
  progress](https://github.com/das-labor/neopg/pull/60)).

These parsers are not ad-hoc parsers, but generated based on formal
grammars using the excellent template library
[PEGTL](https://github.com/taocpp/PEGTL).  For example, the grammar
for an
[URL](https://github.com/taocpp/PEGTL/blob/master/include/tao/pegtl/contrib/uri.hpp)
in PEGTL is lifted straight from
[RFC3986](https://www.ietf.org/rfc/rfc3986.txt), while there is no
corresponding expression of the _intent_ in
[GnuPG](https://dev.gnupg.org/source/gnupg/browse/master/dirmngr/http.c;fa0ed1c7e2eee7c559026696e6b21acc882a97aa$1282).
For comparison, here is the small part that deals with a username
before the host.

From RFC3986:


```BNF
      userinfo    = *( unreserved / pct-encoded / sub-delims / ":" )
      authority   = [ userinfo "@" ] host [ ":" port ]
```

PEGTL grammar in NeoPG (slightly simplified without backtrack handling):

```C++
struct userinfo : star< sor< unreserved, pct_encoded, sub_delims, one< ':' > > > {};
struct authority : seq< opt< userinfo, one< '@' > >, host, opt< one< ':' >, port > > {};

template <>
struct action<pegtl::uri::userinfo> : bind<&URI::userinfo> {};

template <>
struct action<pegtl::uri::host> : bind<&URI::host> {};

template <>
struct action<pegtl::uri::port> : bind<&URI::port> {};

void URI::parse {
  pegtl::parse<uri::authority, uri::action>(input, *this);
}

```

GnuPG source code:

```C
/* Check for username/password encoding */
if ((p3 = strchr (p, '@'))) {
    uri->auth = p;
    *p3++ = '\0';
    p = p3;
}

for (pp=p; *pp; pp++) *pp = tolower (*(unsigned char*)pp);

/* Handle an IPv6 literal */
if(*p == '[' && (p3 = strchr(p, ']'))) {
    *p3++ = '\0';
    /* worst case, uri->host should have length 0, points to \0 */
    uri->host = p + 1;
    uri->v6lit = 1;
    p = p3;
} else
  uri->host = p;

if ((p3 = strchr (p, ':'))) {
  *p3++ = '\0';
  uri->port = atoi (p3);
  uri->explicit_port = 1;
}
```

In GnuPG, semantic actions (`tolower`, `atoi`) are mixed with the
parser, and the underlying formal grammar is barely recognizable.  It
is hard to extract the intent of the source code, and it is also hard
to verify its correctness.

In contrast, the PEGTL approach separates the recognition of the input
from the semantic actions.  This makes it easier to reason about the
correctness of the code, and it also hides the mechanics of pointer
arithmetics and string manipulation.

There is a setup cost in understanding the parser generator, the
language in which the grammar is specified, and how to attach the
semantic actions to grammar rules.  Also, in the case of PEGTL,
backtracking has to be handled manually (a design choice that
maximizes the potential performance).  Given that a large scale
software project can easily contain dozens of parsers and hundreds of
grammar rules, this setup cost quickly pays off.

## Really no ad-hoc parsing?

What if you just have to parse something extremely simple?  For
example, a single byte?  Even in this case, NeoPG uses a formal
grammar.  Here is how the version byte from an OpenPGP public key
packet is parsed:

```
struct version : must<any> {};

template <typename Rule>
struct action : nothing<Rule> {};

template <>
struct action<version> {
  template <typename Input>
  static void apply(const Input& in, PublicKeyData::Version& version) {
    version = static_cast<PublicKeyData::Version>(in.peek_byte());
  }
};

...

PublicKeyData::Version version;
pegtl::parse<version, action>(input, version);
```

This is 12 lines of code to extract a single byte from the input
stream and store it in an `enum` variable.  This seems excessively
complicated compared to the corresponding code in GnuPG
(`g10/parse-packet.c::parse_key`):

```C
int parse_key (IOBUF inp, int pkttype, unsigned long pktlen, ...)
{
  int version = iobuf_get_noeof (inp);
  pktlen--;
  ...
}
```

Can you spot the bug?  Take your time, I will wait.

What is missing in GnuPG is a check if `pktlen` is at least 1.  In
fact, if `pktlen` is 0, then the version byte will be read from the
data _following_ the current packet in the input stream, which usually
is the header of the next packet.  This is a boundary violation that
keeping track of `pktlen` is supposed to prevent.  In turn, `pktlen`
will underflow, and, because it is an unsigned long value, it will be
set to `2^64-1`.  A following sanity check will cause GnuPG to bail
out and read the rest of the input stream (trying to skip `2^64-1` bytes
of data, which it believes to be the length of the remaining data in
the current packet).

The consequence is that GnuPG will behave considerably different from
a conforming implementation, which will recognize the empty public key
packet as empty and that will move the input pointer to the beginning
of the next byte following the empty packet in the input stream.

This type of bugs are often not exploitable in isolation, but they can
be very useful building blocks in constructing larger, more complex
attacks.  You can keep track of this bug upstream in
[T3862](https://dev.gnupg.org/T3862).

The fix is simple, of course:

```C
  if (!pktlen)
    return GPG_ERR_INV_PACKET;
  int version = iobuf_get_noeof (inp);
  pktlen--;
```

But this needs to be done everywhere input is read.  Which is
literally dozens of places for this parser alone, and which often
requires counting the required input bytes manually:

```text
$ grep pktlen parse-packet.c |grep if
...(snippet)...
parse-packet.c:  if (pktlen != 3)
parse-packet.c:  if (pktlen < 4)
parse-packet.c:  if (pktlen > 200)
parse-packet.c:  if (pktlen < 2)
parse-packet.c:  if (minlen > pktlen)
parse-packet.c:  if (pktlen < 12)
parse-packet.c:  if (pktlen < 16)
parse-packet.c:      if (pktlen == 0)
parse-packet.c:  if (pktlen == 0)
parse-packet.c:      if (pktlen < 12)
parse-packet.c:  if (pktlen < 2)
parse-packet.c:      if (pktlen < 2)
parse-packet.c:      if (pktlen < n)
parse-packet.c:      if (pktlen < 2)
parse-packet.c:      if (pktlen < n)
parse-packet.c:  if (pktlen < 2)
parse-packet.c:      if (pktlen > (5 * MAX_EXTERN_MPI_BITS / 8))
parse-packet.c:  if (pktlen < 13)
parse-packet.c:  if (!pktlen)
parse-packet.c:  if (pktlen < 11)
parse-packet.c:  else if (pktlen > MAX_KEY_PACKET_LENGTH)
parse-packet.c:      if (pktlen < 1)
...(and so on)...
```

This is really awkward, and can lead to hard to find errors.  Good
software engineering principles require us to refactor the code to
eliminate these redundancies.  And one way to do so rigorously is by
consistently applying a formal grammar even in simple cases.

## PEGTL rulez!

PEGTL is a good example how the C++ template system can be used in a
constructive and positive way.  The grammar is written in native C++,
instead of a domain specific-language.  PEGTL then generates a parser
which is completely inlined, and can be optimized by the compiler.

This means that, essentially, the above 12 lines of code to formally
parse a version byte and store it into a variable, should be compiled
to essentially the correct 4-line version in the GnuPG code.  There
are some minor differences, because the `must<>` rule generates an
exception instead of returning an error value, but this is by choice.
The overhead is purely syntactical.  The important benefit is that the
source code expresses the intent of the program to the reader, while
the toolchain takes care of compiling it to efficient code.

## Try it!

I hope I could convince you that it might be a good idea to follow the
LANGSEC recommendation not only for complex input languages, but
consistently throughout a program even for seemingly trivial cases.  I
understand that this is not an option for everybody, in all
circumstances.  Especially dynamic programming languages struggle with
performance problems, because they have significant call overhead and
can not inline as aggressively as C++ template libraries.

If you have worked on software projects that followed this approach,
I would love to hear about your experiences, good or bad!
