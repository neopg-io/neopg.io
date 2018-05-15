---
title: "EFAIL and OpenPGP"
date: 2018-05-15
menu:
  main:
    parent: Blog
tags: [legacy]
---

A group of researchers at the University of Applied Sciences Münster
under the lead of Sebastian Schinzel have uncovered a bunch of
problems in email encryption, specifically S/MIME and OpenPGP.  The
results should be a wake-up call for the OpenPGP community.
<!--more-->

## The EFAIL attacks on OpenPGP

The [EFAIL paper](https://efail.de/efail-attack-paper.pdf) describes
attacks on the CFB mode of operation. The attacker can create a new
ciphertext from bits and pieces of intercepted ciphertexts, send the
specifically created ciphertext to the victim, and exploit
backchannels in email clients to get access to the plaintexts of the
original ciphertexts.

The attacks are significant, because they cover a wide range of email
applications, and both dominant email security protocols, and they are
easy to reproduce once the details are known. Thus, the
[EFF](https://www.eff.org/de/deeplinks/2018/05/not-so-pretty-what-you-need-know-about-e-fail-and-pgp-flaw-0)
recommends to stop using email security altogether, until the
standards and applications can be updated. This advice, targeted at
non-technical users, was considered overblown by some in the technical
community, who have more confidence in their ability to configure and
use email security systems correctly.

## A collective failure

There has been some effort on social media to figure out who is
responsible for the vulnerability. OpenPGP supporters (and the GnuPG
project) called out the email clients for not honoring the error
messages that GnuPG emits. Security researches called out GnuPG for
emitting malicious plaintext even in the case of detectable integrity
violations. Although it is not a helpful enterprise to try to find
someone to blame, it is important to evaluate the circumstances that
led to this situation, and after the initial shock the participants on
social media started to dig into the technical details.

There are three sides to this: The OpenPGP standard, the GnuPG
implementation, and the email clients.

### OpenPGP standardized MDC and then stopped

The integrity check (Modification Detection Code, MDC) was added to
RFC4880 as a stop-gap measure to prevent exactly the kind of attacks
that the EFAIL paper pursues (and which have been documented in the
context of OpenPGP by [Schneier and Katz in
2001](https://www.schneier.com/academic/archives/2000/08/a_chosen_ciphertext.html)). It
was also recognized that compression made the attack more
complicated. But support for CFB+MDC+Compression was never
enthusiastic, and it was clear that the decision had to be revisited
eventually. Numerous potential issues were pointed out over the years,
and better solutions were developed (authenticated encryption
schemes). However, it took the OpenPGP community many years to
standardize the MDC in 2007, and since then the effort to develop the
standard further has been sluggish at best. When the IETF working
group was reinstated in 2015, part of the charter was to add an
authenticated encryption scheme to OpenPGP. Unfortunately, the working
group was [closed in November
2017](https://mailarchive.ietf.org/arch/msg/openpgp/d6_ymZTQ6TtizxqVjjPk0AG1dLg)
because "there is not sufficient interest to successfully complete the
work of the working group".

This failure of the OpenPGP community to keep up to date with
developments in security and cryptography research created a technical
debt that includes, among others, legacy support for weak ciphers and
modes of operation, as well as fragile defaults that are difficult to
use securely.

### GnuPG does the right thing, barely

The GnuPG code base was developed incrementally over 20 years, with a
simple core value: "change as little as possible". This conservative
dogma can help to reduce the number of new bugs added to the code
base. In theory, it allows the code base to settle down in a local
optimum and stabilize over the years. The problem is that the local
optimum can be far away from the global optimum, and there is little
room in this development model to make drastic changes to push the
code over a "wall of effort" that separates it from something
better. It also makes it hard to meet increasing or changing demands
from our software ecosystem in a timely manner.

One case where this becomes apparent is the official programming
interface of GnuPG, the "--status-fd", "--with-colons" and
"--command-fd" interface detailed in
[doc/DETAILS](https://dev.gnupg.org/source/gnupg/browse/master/doc/DETAILS). This
interface was bolted onto the existing code base, simply by inserting
appropriate print statements throughout the operations. This required
almost no changes to GnuPG. This interface is also very easy to modify
and extend informally (just add another print statement). It can
easily deal with the streaming mode of operation, where information is
output as a side-effect of message processing. Thus it can also deal
with the complexities of the OpenPGP message format, that allows for
complex nesting of signatures and other packets.  However, the
interface is not easy to use correctly, and is not completely
documented. GPGME deals with some of the complexity, but in many cases
it simply forwards the information, and GPGME users have to consult
GnuPG to find out what the meaning of that information is.

So, although it is true that in the case of an MDC error GnuPG will
emit the ``DECRYPTION_FAILED`` status code, it is not at all clear
from the documentation that this is a serious problem and that the
plaintext that was emitted is dangerous to render. This is the full
documentation from ``doc/DETAILS`` for this:

```
DECRYPTION_FAILED
    The symmetric decryption failed - one reason could be a wrong
    passphrase for a symmetrical encrypted message.
```

Implementors of applications would have to consult the OpenPGP
standard, cryptographic experts, or other resources to find out about
potential vulnerabilities in case of MDC violations.

### The reckless helpfulness of email clients

Email clients want to be helpful. They want to show as much
information to their users as possible. Historically, support for
OpenPGP has been somewhat fragile, and there have been plenty of cases
where decryption failures occured *without* malice. For example, at
some point there was some confusion about how line endings and white
space should be encoded. Inline PGP was sometimes mangled by mail
processing software. At some point, the systems were not reliable
enough to simply suppress plaintext on decryption failure.

These times have now passed, and systems have become more consistent
and reliable. But some email clients did not take advantage of these
improvements to make the system more strict in case of decryption
failures (there was no incentive to do so, because implementors did
not know about the dangers of chosen plaintext attacks).

Email clients have also become more helpful with the content they can
render. Normal people actually enjoy colors, fonts, images and HTML
content in general. So email clients have long evolved from text and
richtext and now include a browser engine.

Take these two things together, and you get a major attack vector on
email security, which was exploited by EFAIL.

## We can do better

My initial reaction to the vulnerabilities was: What can NeoPG do better?

NeoPG already [removed support for non-authenticated
encryption](https://github.com/das-labor/neopg/commit/8e18825c7f6b4d2175a44612f6e4cf7aba4dcbb7)
in October 2017. That was done as a proactive measure, without
knowledge of the EFAIL attack. My gut feeling was that this was an
obsolete, arcane feature in the OpenPGP standard that should not be
supported anymore.

I considered suppressing the output of any plaintext in case of
decryption failure. However, due to the streaming mode of operation
this is easier said than done. Buffering the plaintext output seems
dangerous, because it can be many times larger than the ciphertext due
to compression. Buffering the ciphertext and processing it twice seems
doable at least for small ciphertexts, but retrofitting the legacy
code for that purpose is a lot of effort. In NeoPG, we will implement
this protection as part of the rewrite, when it will be easier to
integrate it in the API (direct memory access rather than passing data
through a file descriptor).

The best way forward is to move to an authenticated encryption scheme,
but that still allows for truncation in case of streaming
operation. There is a proposal for such a scheme in RFC4880bis, but it
has not been scrutinized by the community yet.

## The disclosure

Some people on social media were critical of the disclosure process by
the EFAIL team. I can only relate my own experience here, which was
outstanding.  Even though NeoPG is only a tiny fish in the barrel of
OpenPGP implementations, they contacted me on January 2nd, 2018,
openly shared details about their findings with me and invited me over
for discussion. Their only request to me was that I honor the
disclosure process.

My initial reaction was that the MDC feature should be sufficient
protection, but that I was happy to meet them anyway.  On January 8th,
I met several members of their team, and they explained to me in great
detail how implementation issues in email clients were instrumental in
executing the attacks, deepening my understanding of the issue and
making me aware of how the pieces of the puzzle fit together.

The team answered all of my questions without reservation, shared
their time with me to explore implementation and standardization
issues, and kept me informed about any changes and progress in the
attacks on a continual basis. They are passionate and deeply concerned
about email security, and it is a pleasure to work with them. I have
the utmost respect for the challenges they face dealing with the large
number of vulnerabilities they uncovered, and am very grateful for
their effort.
