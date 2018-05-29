---
title: "Not everything that looks encrypted, is encrypted"
date: 2018-05-28
menu:
  main:
    parent: Blog
tags: [legacy]
---

I found out that it is possible to create a message that looks
encrypted in GnuPG and many email clients, but where the plaintext is
actually not protected at all.
<!--more-->

*Thanks to Fabian Ising and Simon Friedberger for discussions!*

Before I start, let's take a look at the problem.  The following image
consists of screenshots of:

* Earlybird 52.7.0 with Enigmail 2.0.4 (20180516-1359, NixOS),
* Evolution 3.28.2 (Fedora),
* Mutt 1.9.5 (NixOS), and
* Outlook 2007/Gpg4win 3.1.1 (Windows 10).

It shows the rendering of a simple text email in the PGP/Inline format
(so no MIME or HTML is involved). It looks *exactly* as if the email
is encrypted to the recipient. But in fact everything highlighted red
in this image is a total lie - the result of a willful manipulation of
the message by the sender. Let's call this special message the *cake
message*, or short: the *cake*.

{{< load-photoswipe >}}
{{< gallery >}}
{{< figure link="/images/encryption-spoof/thunderbird.png" caption="Enigmail 2.0.4" >}}
{{< figure link="/images/encryption-spoof/evolution.png" caption="Evolution 3.28.2" >}}
{{< figure link="/images/encryption-spoof/mutt.png" caption="Mutt 1.9.5" >}}
{{< figure link="/images/encryption-spoof/gpgol.png" caption="Gpg4win 3.1.1" >}}
{{< /gallery >}}

I promise you that nothing in the cake is encrypted (you will see
later that except for Outlook this is literally true - the encrypted
content of the cake is exactly 0 bytes).  I also promise you that this
output is exactly the same as if the content were properly encrypted.

This bug is certainly not in the same category as a serious security
vulnerability, such as a plaintext leak or a signature spoof. But it
is confusing and hazardous, so it should be fixed. The handling of the
cake message also violates the OpenPGP standard. More importantly,
analyzing the bug helps to understand why OpenPGP is difficult to
implement, and why it is particularly difficult to implement OpenPGP
support using GnuPG.

At the end, I hope that you will understand more about the OpenPGP
standard, the mechanics inside GnuPG, and why the NeoPG project wants
to provide a modern and extensible programming interface for
applications based on OpenPGP.

## Investigating the cake

At first glance, the cake message looks perfectly innocent. But, assuming
that somehow your suspicion is raised, let's play Sherlock Holmes and
investigate a bit further what is going on.

As power users, we turn to the command line and see if we can get some
more information about the ciphertext of the cake message:

```
$ cat cake | gpg
gpg: WARNING: no command supplied.  Trying to guess what you mean ...
gpg: encrypted with 2048-bit RSA key, ID 66489556790B2E8E, created 2018-03-25
      "twitter://lambdafu"
This is fine!
```

Nope, the above output is perfectly normal for an encrypted file.
Adding `--verbose` doesn't change that either. But if we check out the
binary content of the cake message, we can see that it contains the
plaintext in unencrypted form:

```
$ cat cake | gpg --dearmor | strings
This is fine!
```

Modifying this string in the cake shows that the "decryption" output
changes as well, proving that it comes from the unprotected plaintext
part of the cake. Maybe we can find out more by listing the OpenPGP
packets in the cake using a debugging feature of GnuPG:

```
$ cat msg | gpg --list-packet
gpg: encrypted with 2048-bit RSA key, ID 66489556790B2E8E, created 2018-03-25
      "twitter://lambdafu"
# off=0 ctb=85 tag=1 hlen=3 plen=268
:pubkey enc packet: version 3, algo 1, keyid 66489556790B2E8E
	data: [2048 bits]
# off=271 ctb=d2 tag=18 hlen=2 plen=33 new-ctb
:encrypted data packet:
	length: 33
	mdc_method: 2
# off=306 ctb=cb tag=11 hlen=2 plen=20 new-ctb
:literal data packet:
	mode b (62), created 0, name="",
	raw data: 14 bytes
```

Each packet is introduced with a comment line (`#`) indicating the
offset in the file, the ctb and tag, as well as the header and packet
length. If you look very carefully here, you can figure out the
solution.

*Solution:* The encrypted data packet starts at offset 271 and spans
2+33 bytes, so it ends just before offset 306. The literal data packet
follows at offset 306. This means that the encrypted data packet is
not covering the literal data packet at all!

In comparison, this would be part of the output of a properly
encrypted file:

```
... (as before up to the encrypted data packet) ...
# off=271 ctb=d2 tag=18 hlen=2 plen=55 new-ctb
:encrypted data packet:
	length: 55
	mdc_method: 2
# off=284 ctb=cb tag=11 hlen=2 plen=20 new-ctb
:literal data packet:
	mode b (62), created 0, name="",
	raw data: 14 bytes
```

Here, the literal data packet starts at offset 284 and spans 2+20
bytes, so it ends just before 306. It is completely contained within
the encrypted data packet, which goes up to byte 328.

This solution raises new questions:

* What does the OpenPGP standard require of an encrypted message? Is
  the above message well-formed?
* How does GnuPG process the message?  What other methods are there to
  figure out the solution beside `--list-packets`?
* Why do the email clients render the message as if it were encrypted?
* Are there related issues in other parts of the system, known or unknown?
* How can we craft such a message using standard tools?

## The OpenPGP message format

OpenPGP, at its core, is a packet based format.  A packet has:

* a type (tag),
* a length (encoded in one of several formats),
* and some content (of the given length).

The content of a packet can be unstructured (such as plain text or
file data) or structured (with fields of fixed or variable
size). Sometimes, a packet can again contain a sequence of OpenPGP
packets. The encrypted data packet is such a packet, containing
usually a compressed data packet that itself contains a plaintext data
packet, but there are other possibilities.

Here is the composition of a simple, encrypted message without
compression (compare with the output of `--list-packets` above):

```
offset  content
  0     Public-Key Encrypted Session Key Packet
271     Encrypted Data Packet [
    284 Literal Data Packet ]
306     End of file
```

In contrast, the composition of the cake is slightly different, moving
the literal data packet from inside the encrypted packet to the
outside following it:

```
offset  content
  0     Public-Key Encrypted Session Key Packet
271     Encrypted Data Packet []
306     Literal Data Packet
328     End of file
```

### Message composition

In OpenPGP, exported keys, messages, and detached signatures are all specified as
sequences of packets of certain types, in a particular order. [Section
11](https://tools.ietf.org/html/rfc4880#section-11) of RFC4880
specifies the composition of a message. For our example, we
only need a small part of the complete specification:

```
   OpenPGP Message :- Encrypted Message | Literal Data Packet.

   Encrypted Message :- Public-Key Encrypted Session Key Packet, Encrypted Data Packet.

   In addition, decrypting an Encrypted Data Packet must yield a valid
   OpenPGP Message.
```

If read carefully, the OpenPGP standard actually allows an arbitrary
number of nested Encrypted Data packets, but this seems to be a sloppy
oversight, as there is no indication of any use case for this
possibility. The standard does not specify any uppper limit on the
depth of the recursion (GnuPG caps it arbitrarily at
`MAX_NESTING_DEPTH=32`), and for compressed data packets this has lead
to [problems in the
past](https://www.cvedetails.com/cve/CVE-2013-4402/)).

### The cake is not well-formed

In any case, the normal message above is well-formed, given the
following productions for the message:

```
OpenPGP Message
-> Encrypted Message
-> Public-Key Encrypted Session Key Packet, Encrypted Data Packet
```

And the following productions for the decrypted data packet:

```
OpenPGP Message
-> Literal Data Packet
```

However, the cake is not well-formed, and there are two reasons for that.

First, there is no production that creates both an encrypted data
packet and a literal data packet from a single OpenPGP Message at the
same level of nesting.

Second, the Encrypted Data Packet must form a valid OpenPGP Message
after decryption, but in fact it is the zero-length string, which is
not a valid OpenPGP Message at all.

### GnuPG should do more input validation

As we have seen, the cake message is not well-formed, and that would
be a good reason for GnuPG to reject it with an error and reject the
decryption result. Instead, it will happily process what we identified
as a sequence of two OpenPGP messages: one encrypted message, which is
responsible for creating the perception of a fully encrypted message,
and a plaintext message for the actual unprotected payload.

The truth is that GnuPG already tries to protect against this kind of
problem. Since version 1.4.7, GnuPG is supposed to stop processing
when encountering more than one message in the input, unless the
option `--allow-multiple-messages` is given. Unfortunately, the option
is a bit of a misnomer. The actual implementation does not check the
number of messages, but the number of plaintext packets in the input,
which, in case of the cake message, is exactly one. Apparently the
case of a completely empty encrypted data packet was not considered at
the time.

I added a [quick and dirty fix](https://github.com/das-labor/neopg/commit/8311895c0277423a8330e47753a6b8ae2ad4359e) for the legacy code to NeoPG to not
allow any plaintext packets after decryption, but when rewriting the
high-level parser, NeoPG will be very strict about message
composition, and verify that it corresponds to a proper grammar.

### Signatures are not affected

GnuPG does a better job validating the message composition of [signed
messages](https://dev.gnupg.org/source/gnupg/browse/master/g10/mainproc.c;d1431901f0143cdc7af8d1a23387e0c6b5bb613f$1790). Simple
variations on the cake message, but for signing instead for
encryption, are stopped by GnuPG, because it carefully checks the
number and order of signature and plaintext packets according to a
whitelist. It would be good to adopt that approach for all kind of
messages.

## GnuPG input processing

The core of GnuPG is a parser for OpenPGP messages, which runs in a
loop, and calls a handler for each packet type in turn. Depending on
the circumstances, parsing a packet triggers various side
effects. These side effects can be small, such as printing a
status line, or large, such as manipulating the input filter pipeline
to take care of format conversions (ASCII armor or
compression).  In many cases, GnuPG looks at packets mostly in isolation and
has only access to a limited amount of contextual information. After
processing, some results are summarized and validated.

This can be seen in the official programming interface for GnuPG, the
`--status-fd` interface. Here is the output for normal decryption (all
lines starting with `[GNUPG:]` are output to the status fd, while the
plaintext is output to stdout):

```
[GNUPG:] ENC_TO 66489556790B2E8E 1 0
[GNUPG:] KEY_CONSIDERED 013072FB93A232E7C9B1DB3F7EBDF89573BAFB58 0
[GNUPG:] KEY_CONSIDERED 013072FB93A232E7C9B1DB3F7EBDF89573BAFB58 0
[GNUPG:] DECRYPTION_KEY 9669A61C2F57DEC457976E7B66489556790B2E8E 013072FB93A232E7C9B1DB3F7EBDF89573BAFB58 u
[GNUPG:] KEY_CONSIDERED 013072FB93A232E7C9B1DB3F7EBDF89573BAFB58 0
[GNUPG:] BEGIN_DECRYPTION
[GNUPG:] DECRYPTION_INFO 2 1
[GNUPG:] PLAINTEXT 62 0 
[GNUPG:] PLAINTEXT_LENGTH 14
This is fine!
[GNUPG:] DECRYPTION_OKAY
[GNUPG:] GOODMDC
[GNUPG:] END_DECRYPTION
```

Lack of contextualization can be seen from the fact that the
`KEY_CONSIDERED` line is repeated three times. Apart from that, the
result seems fine, and makes perfect sense. The trouble starts when
GnuPG encounters unexpected packet sequences, such as the cake
message:

```
[GNUPG:] ENC_TO 66489556790B2E8E 1 0
[GNUPG:] KEY_CONSIDERED 013072FB93A232E7C9B1DB3F7EBDF89573BAFB58 0
[GNUPG:] KEY_CONSIDERED 013072FB93A232E7C9B1DB3F7EBDF89573BAFB58 0
[GNUPG:] DECRYPTION_KEY 9669A61C2F57DEC457976E7B66489556790B2E8E 013072FB93A232E7C9B1DB3F7EBDF89573BAFB58 u
[GNUPG:] KEY_CONSIDERED 013072FB93A232E7C9B1DB3F7EBDF89573BAFB58 0
[GNUPG:] BEGIN_DECRYPTION
[GNUPG:] DECRYPTION_INFO 2 1
[GNUPG:] NODATA 2
[GNUPG:] DECRYPTION_OKAY
[GNUPG:] GOODMDC
[GNUPG:] END_DECRYPTION
[GNUPG:] PLAINTEXT 62 0 
[GNUPG:] PLAINTEXT_LENGTH 14
This is fine!
```

There are only two differences compared to a normal message:

* There is an additional line `NODATA 2` which indicates that the
  encrypted data packet was entirely void of any content.
* The plaintext handling follows the end of the decryption process.

Both features are strong indicators that this is not a normal
decryption process. But these indicators are not explicit. For
example, the level of nesting of the processed packets is not apparent
from the output apart from the sequence order.

## Suppressing NODATA

GpgOL, the GnuPG Outlook plugin in Gpg4Win, notices the NODATA and
throws an error message. The same is true for the clipboard tool in
the certificate managers Kleopatra and GPA (again, in Gpg4win 3.1.1).

{{< gallery >}}
{{< figure link="/images/encryption-spoof/gpgol-nodata.png" caption="GpgOL" >}}
{{< figure link="/images/encryption-spoof/kleopatra-nodata.png" caption="Kleopatra" >}}
{{< figure link="/images/encryption-spoof/gpa-nodata.png" caption="GPA" >}}
{{< /gallery >}}

However, we can easily fix that by encrypting not a zero-length byte
stream, but an actual OpenPGP packet that is not a literal data packet
and that is otherwise ignored by GnuPG. A good choice is the range of
packet types reserved for private or experimental use (60-63). Every
parsed packet increments a counter, and `NODATA` is only output if this
counter is 0.

Here is the structure of the modified cake message:

```
offset content
0       Public-Key Encrypted Session Key Packet
271     Encrypted Data Packet [
    284 Private Packet 100 ]
311     Literal Data Packet
333     End of file
```

With this modification, Outlook, Kleopatra and GPA behave the same as
the other applications above: They give no indication that there was
anything wrong with the encrypted data packet in the first place, even
though no interesting data is encrypted.

{{< gallery >}}
{{< figure link="/images/encryption-spoof/gpgol.png" caption="GpgOL" >}}
{{< figure link="/images/encryption-spoof/kleopatra.png" caption="Kleopatra" >}}
{{< figure link="/images/encryption-spoof/gpa.png" caption="GPA" >}}
{{< /gallery >}}

## The GnuPG status interface is hard to use

It is an open secret in the GnuPG world that the status interface is
difficult to use, but it is the official interface used either
directly or through GPGME. Unfortunately, it is not very well
documented
([doc/DETAILS](https://dev.gnupg.org/source/gnupg/browse/master/doc/DETAILS)).

The `NODATA` case above shows that applications differ in their
interpretation of the status interface. GpgOL, Kleopatra and GPA
handle `NODATA` as an error, while Enigmail, Evolution and Mutt ignore
it.  Here is [the relevant
code](https://github.com/GNOME/evolution-data-server/blob/0b17843779ee1c80479c33de532d13ca3d9da425/src/camel/camel-gpg-context.c#L1139)
from Evolution, which specifically ignores the NODATA error, because
it relies on GnuPG to do the right thing somewhere else:

```c
  } else if (!strncmp ((gchar *) status, "NODATA", 6)) {
    /* this is an error */
    /* But we ignore it anyway, we should get other response codes to say why */
    gpg->nodata = TRUE;
```

## Improvements and mitigations

One early goal of the NeoPG project is to provide a proper,
stable and extensible API for OpenPGP applications, in form of a
software library.  This is a direct response to many years of
experience with the GnuPG status interface, and the position of the
GnuPG project that no such library will be developed,
period. [Efail](/blog/efail) and the encryption spoof issue described
above provide visible proof that such an API is urgently needed.

But there are some things that the GnuPG project can do to fix this
issue in the meantime:

* GnuPG can and should ensure that encrypted messages are well-formed, and return a proper error code if they are not.
* Applications can check if the `PLAINTEXT` status code follows the `END_DECRYPTION` status code, and throw away the plaintext in that case.

## Appendix: How to bake a cake

This section shows how to generate the above packages with only the
GnuPG command line tool. We assume that the username of the recipient
is "Nerd".

### The cake with `NODATA`

This cake works in many applications, including Enigmail, Evolution
and Mutt, as well as the GnuPG command line. It does not work in
GpgOL.

```sh
# Create an empty encrypted message.
gpg --no-literal --compress-level 0 -r Nerd --encrypt < /dev/null > 01-encrypted.pkt 2> /dev/null

# Create a literal data packet.
echo 'This is fine!' | gpg --store --compress-level 0 --faked-system-time 0 > 01-literal.pkt 2> /dev/null

# Store the plaintext after the encrypted packet.
cat 01-encrypted.pkt 01-literal.pkt > 01-message.gpg

cat 01-message.gpg | gpg --enarmor | sed -e "s/ARMORED FILE/MESSAGE/" | sed -e '/^Comment\:/d' > 01-pgp-inline.gpg
```

You can check the content of this message on the command line:

```sh
cat 01-pgp-inline.gpg | gpg --list-packets
cat 01-pgp-inline.gpg | gpg --status-fd=1 2> /dev/null > 01-status.log
```

### The cake without `NODATA`

This cake works in all applications that I tested.

```sh
# Create a private packet that is ignored by GnuPG
echo -n -e '\xfc\x03\x50\x47\x50' > 02-private.pkt

# Create an encrypted message.
gpg --no-literal --compress-level 0 -r Nerd --encrypt < 02-private.pkt > 02-encrypted.pkt 2> /dev/null

# Create a literal data packet.
echo 'This is fine!' | gpg --store --compress-level 0 --faked-system-time 0 > 02-literal.pkt 2> /dev/null

# Store the plaintext after the encrypted packet.
cat 02-encrypted.pkt 02-literal.pkt > 02-message.gpg

cat 02-message.gpg | gpg --enarmor | sed -e "s/ARMORED FILE/MESSAGE/" | sed -e '/^Comment\:/d' > 02-pgp-inline.gpg
```

Again, you can check the content on the command line:

```sh
cat 02-pgp-inline.gpg | gpg --list-packets
cat 02-pgp-inline.gpg | gpg --status-fd=1 2> /dev/null > 02-status.log
```

## Updates

I reported this issue here:

* [GnuPG](https://dev.gnupg.org/T4000)
