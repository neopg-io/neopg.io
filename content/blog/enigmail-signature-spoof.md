---
title: "SigSpoof 2: More ways to spoof signatures in GnuPG (CVE-2018-12019)"
date: 2018-06-13
author: Marcus Brinkmann, Kai Michaelis
menu:
  main:
    parent: Blog
tags: [legacy]
---

This is another attack to spoof digital signatures specific to Enigmail.
<!--more-->

Previously, we showed how to spoof ["encrypted"
messages](/blog/encryption-spoof) that were not actually encrypted,
and how to spoof ["trusted" signatures](/blog/gpg-signature-spoof) on
messages that were not actually signed.

<img class="ui centered medium rounded image" src="/images/enigmail-signature-spoof/sigspoof-2.png">

<div class="ui right floated basic segment" style="padding: 0">
<div class="ui card">
  <div class="content">
      <span class="right floated avatar image">
        <img src="/images/avatar/seu.png">
      </span>
    <a class="header">Kai Michaelis</a>
    <div class="meta">
      <span class="date">Hacker, Bochum</span>
    </div>
    <div class="description">
        Fellow hacker from <a href="https://wiki.das-labor.org/w/LABOR_Wiki">Das Labor</a> in Bochum.
 Check out his cross platform disassembler for reverse engineering,
 <a href="https://panopticon.re">Panopticon</a>!
    </div>
  </div>
  <div class="extra content">
    <a href="https://twitter.com/_cibo_">
      <i class="twitter icon"></i>
      _cibo_
    </a>
  </div>
</div>
</div>

This time, we show two more methods to spoof the signer and trust
value of signatures with GnuPG that is specific to Enigmail.  Again,
we exploit the awkwardness of the text-based GnuPG status interface.

*This work was done in collaboration with Kai Michaelis (see info box)
at the Bochumer hacker space [Das
Labor](https://www.das-labor.org/). Thanks also to [Simon
Wörner](https://github.com/SWW13) for help with the CVEs and
discussions!*

Many thanks also to Patrick Brunschwig, who maintains Enigmail in his
spare time, and who did an amazing job responding and attending to
these issues in a quick and professional manner!

## tl;dr

We found multiple vulnerabilities in Enigmail:

> [CVE-2018-12019:](https://www.cvedetails.com/cve/CVE-2018-12019) The
> signature verification routine in Enigmail 2.0.6.1 interprets user ids
> as status/control messages and does not correctly keep track of the
> status of multiple signatures, which **allows remote attackers to spoof
> arbitrary email signatures via public keys containing crafted primary
> user ids.**

You can protect yourself:

* Upgrade to [Enigmail 2.0.7](https://sourceforge.net/p/enigmail/forum/announce/thread/b948279f/)

### Identifiers

This vulnerability is tracked under the following identifiers:

* [CVE-2018-12019](https://www.cvedetails.com/cve/CVE-2018-12019)

Distribution updates:

* [Suse](https://www.suse.com/de-de/security/cve/CVE-2018-12020/)

## Demonstrating the signature spoof

This screenshot is from Enigmail 2.0.6, and apparently shows a message
with a valid signature by Patrick Brunschwig. In reality, this message
was signed and sent by us using a specially crafted key.

<a href="/images/enigmail-signature-spoof/enigmail.png"><img class="ui centered bordered image" src="/images/enigmail-signature-spoof/enigmail.png"></a>

## Method I: Signature Status Confusion

This method allows an attacker to craft **a message with a signature
by an untrusted key that Enigmail shows as "trusted."** The message
box will be green and the envelope will carry a red wax seal instead
of a grey seal with a question mark.

### Root cause

Enigmail can only keep track of a single signature. Enigmail has
little contextual information on the status lines, and will overwrite
the signature state with the last seen signature. In particular, these
status messages are relevant to the discussion:

* `GOODSIG [KEY_ID] [USER_ID]` indicates a good signature and provides a short key id and the primary user id.
* `EXPKEYSIG` and `REVKEYSIG` are similar to `GOODSIG` but for signatures by expired or revoked keys.
* `VALIDSIG [FPR] [CREATION_DATE] [TIMESTAMP] [EXPIRE_TIMESTAMP]
  [VERSION] [RESERVED] [PUBKEY_ALGO] [HASH_ALGO] [CLASS]
  [PRIMARY_KEY_FPR]` provides much more information on the signature,
  such as creation time, long fingerprint, and the algorithms used.

Some status flags understood by Enigmail have a "trapdoor" effect in
the sense that they can only trigger setting a flag which is not
deleted afterwards. In the case of error flags that is often useful
and the safe default, but for `TRUST_FULLY` and `TRUST_ULTIMATE` this
behaviour is actually dangerous.

We found two flaws in the way Enigmail handles status messages for
multiple signatures:

* If the status of the *last* signature is `GOODSIG`, `EXPKEYSIG` or
  `REVKEYSIG`, Enigmail will overwrite the signature details
  (fingerprint, creation time, algorithm identifiers) with the
  information from the *first* `VALIDSIG`, confusing the metadata of
  two signatures. For example, this allows us to change the state of a
  signature to good, expired or revoked, by adding a second signature
  with an appropriate state. Enigmail will display the details of the
  first signature, but the state of the second.
* If any of the (multiple) signatures is `TRUST_FULLY` or
  `TRUST_ULTIMATE`, and the *last* signature is good (can be expired or
  revoked), then Enigmail will display the information from the *first*
  `VALIDSIG` as trusted.

Put together, the attacker can trick Enigmail into assigning the trust
value of one signature to the details of another signature.

### Proof of concept

Simply generate two keys, and sign a message with both. Import the
public keys into a new keyring, and set full trust for one of
them. Sign a message first with the untrusted key, and second with the
trusted key. Enigmail will display the signature from the untrusted
key as trusted.

```
$ mkdir /tmp/trustconfusion
$ gpg --homedir /tmp/trustconfusion --quick-gen-key "Dubious Key <dubious@example.com>"
$ gpg --homedir /tmp/trustconfusion --quick-gen-key "Trusted Key <trusted@example.com>"
$ gpg --homedir /tmp/trustconfusion/ --export Trusted Dubious | gpg --import
$ gpg --edit-key Trusted
> lsign
> save
$ echo 'Do you remember me?' | gpg --homedir /tmp/trustconfusion --armor --sign -u Dubious -u Trusted | gpg --status-fd 2
...
[GNUPG:] GOODSIG 848346A87AEDB69D Dubious Key <dubious@example.com>
[GNUPG:] VALIDSIG 3A6744E8E65F3CAF3E0FB88A848346A87AEDB69D 2018-06-07 1528401495 0 4 0 1 8 00 3A6744E8E65F3CAF3E0FB88A848346A87AEDB69D
[GNUPG:] TRUST_UNDEFINED 0 classic
...
[GNUPG:] GOODSIG 645130A3A9C92809 Trusted Key <trusted@example.com>
[GNUPG:] VALIDSIG 05B34301E7FEB1328344789E645130A3A9C92809 2018-06-07 1528401495 0 4 0 1 8 00 05B34301E7FEB1328344789E645130A3A9C92809
[GNUPG:] TRUST_FULLY 0 classic
...
```

<a href="/images/enigmail-signature-spoof/signature-confusion.png"><img class="ui centered bordered image" src="/images/enigmail-signature-spoof/signature-confusion.png"></a>

To make the attack work, the victim has to import the keys generated
by the attacker and trust one of them. We show below several ways to
achieve this under more restrictive conditions (were the user id is
more suspicious than here).

### Mitigations

* Enigmail 2.0.7 keeps track of the context of signature metadata, and
  groups all information related to a single signature together (the
  `NEWSIG` status messages can be used to do that).


## Method II: Status message injection through user ids

This method allows an attacker to **spoof arbitrary signature details
and the trust status of the signature**. All signature details
displayed by Enigmail are under the control of the attacker. The
message box will be green and the envelope will carry a red wax seal
instead of a grey seal with a question mark.

### Root cause

This attack works due to the incomplete way Enigmail matches status
messages with regular expressions.

Again, we find synergy between two unrelated weak design choices in
GnuPG and Enigmail:

* Enigmail matches some (not all, see `TRUST_FULLY` below) GnuPG
  status messages like `GOODSIG` and `BADSIG` from *the end of a
  line, ignoring the beginning*.
* GnuPG emits the primary user ID of a signing key at the end of a
  `GOODSIG` status line, *without escaping whitespace*.

This enables us to **inject some GnuPG status lines into Enigmail by writing
them into the primary user ID of a signing key**.

Note: This is a list of status lines that can be injected this way due
to incomplete regular expressions: `GOODSIG`, `BADSIG`, `EXPSIG`,
`EXPKEYSIG`, `REVKEYSIG`, `ERRSIG`, `VALIDSIG`, `USERID_HINT` and
`ENC_TO`. In this attack, we only make use of `VALIDSIG` and
`GOODSIG`:

Unfortunately, we can not inject the `TRUST_*` status lines through
user ids, because Enigmail expects them to be at the beginning of a
status line (here Enigmail is more strict than with `VALIDSIG` or
`GOODSIG`).

### Basic idea of the attack (non-functional)

The main idea of the attack is to use a valid signature by a key with
a malicious user id that itself contains a `GOODSIG` or `VALIDSIG`
status line indicating an entirely different key. In this example, we
are using the details from the key of the Enigmail author.

```
$ gpg --status-fd 2 --quick-gen-key "GOODSIG DB1187B9DD5F693B Patrick Brunschwig <patrick@enigmail.net>"
...
[GNUPG:] KEY_CREATED B BF52C30EA443623C6299883FB81CB745B9100FC9
...
$ gpg --status-fd 2 --quick-gen-key "VALIDSIG 4F9F89F5505AC1D1A260631CDB1187B9DD5F693B 2018-05-31 1527721037 0 4 0 1 10 01 4F9F89F5505AC1D1A260631CDB1187B9DD5F693B"
...
[GNUPG:] KEY_CREATED B 552D4E1657548BF8C4B3A5643C2AA6885649565E
...
$ echo 'I am ordering a pizza.' | gpg --sign -u 552D4E1657548BF8C4B3A5643C2AA6885649565E | gpg --status-fd 2
...
[GNUPG:] GOODSIG B81CB745B9100FC9 GOODSIG DB1187B9DD5F693B Patrick Brunschwig <patrick@enigmail.net>
...                               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
[GNUPG:] GOODSIG 3C2AA6885649565E VALIDSIG 4F9F89F5505AC1D1A260631CDB1187B9DD5F693B 2018-05-31 1527721037 0 4 0 1 10 01 4F9F89F5505AC1D1A260631CDB1187B9DD5F693B
...                               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

This approach is not quite right, due to the way Enigmail processes the status lines. However, with a bit of extra work, it can be modified to a functional attack.

### Proof of concept

We saw above (in "Method I: Signature status confusion"), that
Enigmail uses the signature state (`GOODSIG`) of the last signature,
but the fingerprint and other information from the very first
`VALIDSIG` in the output.

Luckily, `GOODSIG` with the injected user id comes before the corresponding
`VALIDSIG`. This means that for a successful attack, we only
need to inject a proper `VALIDSIG` line in the first
signature. Injecting `GOODSIG` is entirely superfluous, and we can
rely on the valid, original `GOODSIG` from GnuPG to trigger the search
for `VALIDSIG`.

`VALIDSIG` must be injected in the primary user id of the very first
signature, because otherwise the correct `VALIDSIG` information
emitted by GnuPG for the first signature would be used instead.

Although we said that Enigmail *matches* `VALIDSIG` even in the middle
of a line, it *retrieves* the key details from the beginning of the
line (after stripping away the first 9 characters for `VALIDSIG␣'). So
we have to repeat the required arguments at the beginning of the line.

Here is a complete, stripped-down and working example for a user id
that can be used to spoof a signature in Enigmail:

```
x 1527763815 x x x 1 10 x 4F9F89F5505AC1D1A260631CDB1187B9DD5F693B VALIDSIG x x 0 4F9F89F5505AC1D1A260631CDB1187B9DD5F693B
  ^ creation              ^ 40-character fingerprint                              ^ 40-character fingerprint
                     ^ hash algorithm                                       ^^^^^ matches (\w+) (.*) (\d+)
                   ^ pubkey algorithm
```

There are two remaining issues:

* The attacker needs to get the public key into the victim's keyring,
  so that GnuPG can display the user id (which is not contained in the
  signed message).
* The attacker needs to convince Enigmail that the spoofed signature
  is made by a key that is trusted.

The assumption here is that the victim has already established
communication with the signer the attacker wants to spoof, and that
the victim has managed the trust settings accordingly. If either of
these is not true, it is easy to perform a man in the middle attack
anyway.

### Mitigations

#### For users

* Upgrade to [Enigmail 2.0.7](https://www.enigmail.net/)

#### For Enigmail developers

* Enigmail 2.0.7 matches status messages more rigorously by hardening
  the regular expressions.
* Enigmail 2.0.7 pays more attention to the context of status lines,
  especially in the case of multiple signatures. For example, it uses
  the `NEWSIG` boundary to clear internal data structures, and makes
  sure that `GOODSIG`, `VALIDSIG` and `TRUST_*` status lines are taken
  from the same signature context (between two `NEWSIG`s).
* Enigmail 2.0.7 uses the new key import GUI (that it uses for
  attached keys) also for inline keys, and displays the user id in a
  more distinguishable way.

## Injecting malicious signing keys into the victim's keyring

In both methods, the attacker must inject the signing key with the
malicious user id into the victim's keyring ahead of sending the
spoofed message. There are many ways to do this, and the best way
depends on the victim's configuration and behaviour.

The main point about this is that in the OpenPGP community it is
considered acceptable to import malicious keys into the keyring, as
long as they are not assigned a trust value. In fact, the tenor is
that it is unavoidable that the attacker can insert arbitrary keys into
the victim's keyring, and that alone is not considered a breach of the
security model.

These are some ways to inject the key using the keyserver network:

* In the simplest case, the attacker can simply upload the key to the
  public keyserver network and let the `auto-key-retrieve` option in
  GnuPG do all the work. The key is then downloaded and imported
  automatically, but the existance of the key is public knowledge.  
* Or the attacker can maintain their own keyserver and get it accepted
  by the SKS keyserver pool, which is used in GnuPG by default. When,
  at any time, the victim accesses the keyserver to download a key
  (either manually or with `auto-key-locate`), the malicious key can
  be added to the payload. This way the existance of the key is hidden
  to the network and other users.

These are some ways to inject the key by email:

* The attacker can simply send the key to the victim ahead of the
  attack as an inline key. This is an old feature in Enigmail with a
  bad user interface: The key can not be inspected before import,
  there is no easy way to undo the import, and the result of the
  import is shown in a hard-to-understand manner that hides the
  details of the attack quite nicely for most users (see
  screenshots). Also, interestingly, once the victim clicks `Import
  Key`, Enigmail will ask three times if the user cancels the first
  two confirmation dialogues. That nagging is working in favor of the
  attacker.

{{< gallery >}}
{{< figure link="/images/enigmail-signature-spoof/import-key-01.png" caption="Import inline key" >}}
{{< figure link="/images/enigmail-signature-spoof/import-key-02.png" caption="Enigmail does not show the content before import" >}}
{{< figure link="/images/enigmail-signature-spoof/import-key-03.png" caption="Can you spot the evil user id?" >}}
{{< /gallery >}}

* Sending the key as an attachment is less favorable for the attacker,
  because Enigmail asks the user to verify the primary user id before
  the import, and shows the import result in a nicer user
  interface. But if we replace the ignored parts in the malicious user
  id (`x`) with some nice looking strings, maybe we can trick the
  victim into accepting the key anyway? Judge for yourself by the
  screenshots.

{{< gallery >}}
{{< figure link="/images/enigmail-signature-spoof/import-attached-01.png" caption="Import attached key" >}}
{{< figure link="/images/enigmail-signature-spoof/import-attached-02.png" caption="Enigmail shows the content, but dsiplays embedded newlines and does not show boundary" >}}
{{< figure link="/images/enigmail-signature-spoof/import-attached-03.png" caption="Does this look suspicious? But now the key is already imported!" >}}
{{< /gallery >}}

It is also possible that the victim can be tricked into importing the
key at the command line, for example to verify some signature of a
software download. A single downloaded public key file can contain
several public keys, and the malicious user id can simply be lost in
the noise.

There are also new ways to inject keys, for example
[autocrypt](https://autocrypt.org/). We did not explore these for this
attack, because the above seem sufficient to illustrate the point.

Once the key is imported, it requires manual action by the victim to
get rid of it. The victim must manually identify the key and delete it from
the keyring. Until then, the victim is vulnerable to our attack.

## Injecting trusted, innocuous keys into the victim's keyring

Both methods above rely on the fact that we can get the victim to
trust at least one key generated by the attacker. This key can have
any user id, and is completely unrelated to the spoofed signature
details. Because exchanging such keys and trusting them is the normal
mode of operation for OpenPGP, this is usually not difficult to
accomplish. Nevertheless, we give some helpful advice here to spur the
reader's imagination.

We start with some advice: Forensically, the victim will be able to
identify the key that was used (it will be "burned"), so it is a good
idea to set up a new, trusted communication channel with the victim
under an innocuous user id.

In the simplest case, the attacker and the victim will exchange key
signatures directly. If the victim uses the "Web of Trust" policy in
GnuPG, the attacker might be able to infiltrate that.

Enthusiastic OpenPGP supporters are often happy to sign keys of
strangers at international conferences at so-called "key signing
parties". The signatures are made based on a cursory inspection of
international passports or identity cards, and it is unlikely that the
victim is familiar with passports and id cards from all recognized
nation states. Given the helpfulness of the community towards
activists from non-democratic countries, a social engineering attack
under an appropriate cover story might be possible.

{{< figure link="/images/enigmail-signature-spoof/keysigning-party.jpg" caption="A keysigning party at an international conference." >}}

Again, the goal here is to get the victim to sign a completely
unrelated and innocuous key with a properly formed user id consisting
of a name and email address. Eventually, the attacker sends the actual
attack email carrying the signature by the key with the malicious user
id and the signature by the trusted key. Enigmail will display the
spoofed signature metadata in the former and the trust status of the
latter.

### Additional Detail: Spoofing the spoofed keys user id

Normally, we expect the victim to have the key of the spoofed signer
in the local keyring (with an assigned trust value) due to previous
communication. But if that is not the case, we can inject it using the
above described methods.

However, the mail client is usually a long running program, and
Enigmail caches the information in the local key ring. A missing key
or stale cache can cause Enigmail to fall back to the user id in the
`GOODSIG` status line of the signature verification, instead of using
the user id from the key with the fingerprint in the spoofed
`VALIDSIG` line, defeating the attack by showing the wrong user id in
the output.

To get around this, we can add another signature by a key
with the following primary user id (this must be the last signature in
the message):

```
GOODSIG DB1187B9DD5F693B Patrick Brunschwig <patrick@enigmail.net>
```

We inject this key along with the other malicious key as described
above.  Enigmail will then fall back to the provided user id in case
the public key is missing from the local keyring or Enigmail's
cache.

<div style="clear: both"></div>
