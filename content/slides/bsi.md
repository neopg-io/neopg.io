---
title: "OpenPGP and NeoPG"
date: 2018-03-28
menu:
  main:
    parent: Slides
revealoptions:
  transition: fade
description: "BSI, Bonn 28. März 2018"
pdf: /slides/bsi.pdf

---

# NeoPG

## An Alternative to GnuPG

<small>
_Marcus Brinkmann · BSI, Bonn (28. März 2018)_
</small>

---

## Overview

* OpenPGP
* Comparison of some implementations
* NeoPG

---

# OpenPGP

---

## Keys/1000 per year

<canvas class="stretch" data-chart="line">
SKS Uploads/Thousand, 315.344, 376.343, 414.952, 244.339, 176.660, 147.771, 149.171, 152.243, 121.457, 127.064, 130.802, 132.074, 122.065, 127.413, 162.920, 306.447, 360.199, 324.706, 328.173, 279.097
<!--
{
"data" : {
	"labels" : ["1998", "1999", "2000", "2001", "2002", "2003", "2004", "2005", "2006", "2007", "2008", "2009", "2010", "2011", "2012", "2013", "2014", "2015", "2016", "2017"],
	"datasets" : [{ "borderColor": "orange" } ]
	}
}
-->
</canvas>

---

## Standardisation

* RFC2440 (1998)
* RFC3156 (2001): OpenPGP/MIME
* RFC4880 (2007) ⟵ Current version
* RFC5581 (2009): Camellia block cipher
* RFC6637 (2012): ECDH+ECDSA (NIST)

---

## Crypto-Museum (Examples)

* SHA-1 (hardwired)
* <!-- .element: class="fragment" --> 3DES, CAST5 (64-bit block cipher)
<pre class="fragment" style="font-size: small;">Werner Koch: AES256, AES192, AES, CAST5, 3DES
Triple Nerd: 3DES, CAST5, AES, AES192, AES256
GnuPG takes: <span class="fragment">3DES</span></pre>
* <!-- .element: class="fragment" --> CFB mode with integrity check<br/>
* <!-- .element: class="fragment" --> No trust model

---

## Shrinking Community

* "It's time for PGP to die." (Matthew Green, 2014)

* <!-- .element: class="fragment" --> "I think of GPG as a glorious experiment that has run its course." (Moxie Marlinspike, 2015)

* <!-- .element: class="fragment" -->"I'm giving up on PGP" (Filippo Valsorda, 2016)

* <!-- .element: class="fragment" -->"Sorry, but I cannot decrypt this
message. I don't have a version of PGP that runs on any of my devices."
(Phil Zimmermann, 2015)

---

<section>
<img src="/images/intro-to-modern-crypto-cover.jpg">
<!--Introduction to Modern Cryptography, 2nd Ed. (Jonathan Katz, 2014)-->
</section>

---

<section>
<img class="stretch" src="/images/serious-cryptography-cover.jpg">
<!--Serious Cryptography (Jean-Philippe Aumasson, 2017)-->
</section>

---

## OpenPGP Working Group (IETF)

June 2015: Beginning of RFC4880bis.  <!-- Chair: Daniel Kahn Gillmor,
Christopher Liljenstolpe.  Redakteur: Werner Koch. -->

* Brainpool, SECG, Curve25519 for ECDSA+ECDH
* Integration of Ed25519
* SHA2-256 fingerprint
* AEAD mode (EAX)

<div class="fragment">Unfortunately missing:
<ul><li>KDF Argon2 (to replace S2K)</li></ul>
</div>

---

## OpenPGP Working Group (IETF)

<blockquote style="margin: 0; width: 100%;">
"The Chairs and the AD have concluded that there is not sufficient interest to successfully complete the work of the [OpenPGP] working group."<br/>
  – IESG Secretary, 11 Nov 2017
</blockquote>
</div>

---

## RFC4880bis

* Implemented in GnuPG (g10 Code GmbH)
* Implemented in RNP (Ribose Inc.)
* Curve25519+ECDH instead of X25519
* Informal development now

---

## OpenPGP-Profile

Documenting:

* Scope of implementation of RFC4880(bis)
* Additional restrictions (input, output)
* Intentional differences

At least: One profile per implementation

Better: Shared profiles

---

## Interoperability

Currently: Star-topology (test with standard and GnuPG)

* Shared test vectors
* Interoperability tests
* Better visibility of implementations
* Strenghten the ecosystem outside of GnuPG

---

# Implementations

---

## Overview

<table class="noborder" style="font-size: small;">
<tr><th>Project</th><th>Responsible</th><th>Language, Crypolib.</th><th>API</th><th>Platforms</th><th>License</th></tr>
<tr><td>GnuPG</td><td>g10 Code, GnuPG e.V.</td><td>C, libgcrypt</td><td>CLI, GPGME</td><td>Linux, Windows, macOS</td><td>GPLv3</td></tr>
<tr><td>Sequoia-PGP</td><td>PEP Foundation</td><td>Rust, libnettle</td><td>Rust, C</td><td>Linux, ?</td><td>tbd (GPLv3, LGPLv3)</td></tr>
<tr><td>OpenKeychain</td><td>Community</td><td>Java, BouncyCastle</td><td>Android Service</td><td>Android</td><td>GPLv3 (API: Apache)</td></tr>
<tr><td>RNP</td><td>Ribose, Inc.</td><td>C, Botan</td><td>C</td><td>Linux, macOS</td><td>BSD 2-clause</td></tr>
<tr><td>NeoPG</td><td>Community</td><td>C++, Botan</td><td>C++, C</td><td>Linux, macOS (Windows)</td><td>BSD 2-clause / GPLv3 (legacy)</td></tr>
</table>

---

## API-Requirements (examples)

* Signature-verification _without_ key management: `verify(data, signature, public_key)`
* <!-- .element: class="fragment" --> Dito, for a specific subkey
* <!-- .element: class="fragment" --> Public-Key-Informationen _without_ import
* <!-- .element: class="fragment" --> Packet-stream-analysis for research
* <!-- .element: class="fragment" --> Stability _and_ progress

---

## Is there a GnuPG library?

2010
<blockquote style="font-size: small;">
This has been frequently requested. However, the current
viewpoint of the GnuPG maintainers is that this would lead to several
security issues and will therefore not be implemented in the
foreseeable future. However, for some areas of application gpgme could
do the trick.
</blockquote>

<div class="fragment">2013
<blockquote style="fontsize: small;">
No, nor will there be.
</blockquote>
</div>

---

## Tapping potential

* Applications know requirements better
* Allow unforeseen potential
* Exchange in both directions
* Better software quality in depth

---

# NeoPG

---

## NeoPG as an Alternative

* Fork of GnuPG
* Cleanup code, easier development
* Stable, extensible API
* Find new applications
* Easier to use

---

## A Community

* Developed on GitHub
* Discussion over social media
* Requirements through community dialogue
* Change-documentation in issues, pull-requests
* Background information in blogs
* Code Review (4-Eyes)

---

## Software-Engineering

High quality despite large changes.

* Coding Standards (`clang-format`, SonarQube)
* Continuous Integration Tests
* Test-Coverage
* Static Analysis
* Fuzzing

---

## Architecture

* One repository
* One library
* One CLI-tool
* lightweight process isolation (Heartbleed)

---

## Delegate Responsibility

* C++, STL, some Boost
* libcurl for network access
* SQLite for persistent database
* Botan as crypto-library

---

## Command Line

* Git-style subcommands (`neopg encrypt ...`)
* Compatibility layer (`neopg gpg2 ...`)
* Colored terminal output
* Better diagnostics

---

## Status

* Infrastructure established
* Minus 250.000 LoC (~50%)
* Minus 120 CLI options (~30%)
* First API parts (OpenPGP parser)

---

## Example: Parser-Design

* Ad-Hoc Parser are common cause of errors
* <!-- .element: class="fragment" -->Formal language for all untrusted input
* <!-- .element: class="fragment" -->Consistent: even for trivial input (1 Byte!)
* <!-- .element: class="fragment" -->C++ allows typesafe formal grammar ...
* <!-- .element: class="fragment" -->... parser generator without loss of performance
* <!-- .element: class="fragment" -->Some mistakes can be avoided systematically

<!-- .element: class="fragment" -->https://neopg.io/blog/no-ad-hoc-parser/

---

## <i class="ui icon globe orange"></i>neopg.io
## <i class="ui icon twitter orange"></i>@neopg_
