---
title: "SigSpoof 4: Bypassing signature verification in Yarn package manager (CVE-2018-12556)"
date: 2019-03-21
author: Marcus Brinkmann
menu:
  main:
    parent: Blog
tags: [legacy]
---

This attack on GnuPG signature verification is specific to yarn, the
package manager. It can give a powerful attacker the ability to
replace the Yarn installation with arbitrary code. There are
additional protections in place, so if you are using Yarn, you
probably do not need to worry too much about it.

<!--more-->

Previously, we showed how to spoof ["encrypted"
messages](/blog/encryption-spoof) that were not actually encrypted,
and how to spoof "trusted" signatures on messages that were not
actually signed, exploiting bugs in [GnuPG](/blog/gpg-signature-spoof)
and [Enigmail](/blog/enigmail-signature-spoof). We also demonstrated
that insufficient signature verification with GnuPG can lead to
information leak and remote code execution with
[pass](/blog/pass-signature-spoof).

<img class="ui centered medium rounded image" src="/images/yarn-signature-spoof/sigspoof-4.png">

<div class="ui right floated basic segment" style="padding: 0">
<div class="ui card">
  <div class="content">
      <span class="right floated avatar image">
        <img src="/images/avatar/lukele.jpg">
      </span>
    <a class="header">Lukas Pitschl</a>
    <div class="meta">
      <span class="date">Hacker, Vienna (Austria)</span>
    </div>
    <div class="description">
        Lukas is lead developer of GPGMail and can fix security problems in iMail <a href="https://twitter.com/jensvoid/status/1004015269742825472">better than Apple</a> by reverse engineering and binary patching! If you are using macOS, check out <a href="https://gpgtools.org/">GPG Suite</a>!
    </div>
  </div>
  <div class="extra content">
    <a href="https://twitter.com/lukele">
      <i class="twitter icon"></i>
      lukele
    </a>
  </div>
</div>
</div>

Previous signature spoofs have focussed on injecting status messages
or bypassing regular expressions. This time, we show an even more
trivial bypass due to a failure to pin the certificate at all. We are
exploiting the incompleteness of the GnuPG command line interface,
which makes it difficult for developers to support OpenPGP securely.

*This work was done in collaboration with Lukas Pitschl (see info box)
from the GPGTools project. Again, thanks also to [Simon
Wörner](https://github.com/SWW13) for help with the CVEs and
discussions!*

## tl;dr

I found a critical vulnerability in Yarn, the package manager:

> [CVE-2018-12556](https://www.cvedetails.com/cve/CVE-2018-12556): The
> signature verification routine in install.sh in yarnpkg/website
> through 2018-07-06 only verifies that the yarn release is signed by
> any (arbitrary) key in the local keyring of the user, and does not pin
> the signature to the yarn release key, which allows remote attackers
> to sign tampered yarn release packages with their own key.

<!--
You can protect yourself:

* Upgrade to the Yarn `install.sh` script from XXX 
-->

### Identifiers

This vulnerability is tracked under the following identifiers:

* [CVE-2018-12556](https://www.cvedetails.com/cve/CVE-2018-12556)

## Scope of the attack

Yarn, the package manager, delivers an installation script and an
installation package from the [`yarnpkg.com`
website](https://github.com/yarnpkg/website). The integrity of these
files is protected by HTTPS/TLS, but there is also an attempt for
additional end-to-end integrity protection through a GnuPG signature
by a Yarn release key.

This GnuPG signature verification in Yarn is insufficient, because it
does not pin the signature to the yarn release key.

An attacker who can inject a key under his control into the user's
keyring is able to bypass the signature verification in Yarn, allowing
the attacker to tamper with or replace the installation file.

This attack breaks the integrity protection of yarn completely, under
the following conditions:

* The attacker must be able to inject a key into the target's keyring, and
* the attacker can tamper with or replace the Yarn installation
  package (e.g. the nightly build).

## Lack of certificate pinning

This method is even more trivial than the previous method to bypass
signature verification in [`pass`](/blog/pass-signature-spoof),
because Yarn does not even attempt to do certificate pinning at all.

### Root cause

The attack works because Yarn `install.sh` calls GnuPG to verify the
signature of the installation package, but it does not verify that the
signature was made by the Yarn release key. **Any signature that can
be verified is accepted.** To verify a signature, GnuPG only requires
the public key to be in the user's keyring. Thus, **the Yarn
installation package can be signed with any key from the user's public
keyring.**

## Walkthrough

Instead providing a proof of concept, which requires an attack on the
integrity of the release file on the Yarn download server, we walk
through the source code of the `install.sh` script to demonstrate the
fault.

Yarn recommends under [Alternatives
Installation](https://yarnpkg.com/en/docs/install#alternatives-stable)
to execute the following shell command line:

```
$ curl -o- -L https://yarnpkg.com/install.sh | bash
```

This will download and execute the [installation
script](https://github.com/yarnpkg/website/blob/master/install.sh),
which contains the following code (extract):

```
gpg_key=E074D16EB6FF4DE3

# Verifies the GPG signature of the tarball
yarn_verify_integrity() {
  # Grab the public key if it doesn't already exist
  gpg --list-keys $gpg_key >/dev/null 2>&1 || (curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | gpg --import)

  if [ ! -f "$1.asc" ]; then
    printf "$red> Could not download GPG signature for this Yarn release. This means the release can not be verified!$reset\n"
    yarn_verify_or_quit "> Do you really want to continue?"
    return
  fi

  # Actually perform the verification
  if gpg --verify "$1.asc" $1; then
    printf "$green> GPG signature looks good$reset\n"
  else
    printf "$red> GPG signature for this Yarn release is invalid! This is BAD and may mean the release has been tampered with. It is strongly recommended that you report this to the Yarn developers.$reset\n"
    yarn_verify_or_quit "> Do you really want to continue?"
  fi
}
```

First, the script checks for the presence of the release key
(`E074D16EB6FF4DE3`) in the local keyring, and downloads and imports
the key if it is missing.  The installation script will then verify
the detached signature in the `.asc` file for the installation packet.
Unfortunately, `gpg --verify` accepts any signature by a key in the
local keyring, and the exit status will be `0` (successful validation)
as long as the signature is good.  The key does not need to be trusted
by the user, it can even be expired.

If the key is not available in the local keyring, verification will
fail.  However, GnuPG can be configured to retrieve missing keys
automatically via several methods (for example, public keyserver or
WKD).  If the user configured GnuPG in such a manner, the attacker can
simply make the key available in this manner.

## Attacking the integrity of the release file

The description above raises a question: If the attacker can tamper
with the installation package, can't the attacker also tamper with the
public key file and the installation script on the website?

The installation package can come from any of these sources:

* `https://nightly.yarnpkg.com/latest.tar.gz`
* `https://yarnpkg.com/latest-rc.tar.gz`
* `https://yarnpkg.com/downloads/$version/yarn-v$version.tar.gz`
* `https://yarnpkg.com/latest.tar.gz`

The installation script and the public key come from these sources:

* `https://yarnpkg.com/install.sh`
* `https://dl.yarnpkg.com/debian/pubkey.gpg`

This means that at least the nightly builds are not co-located with
either the installation script or the public key.  Package signatures
may also be replaced on their way from the build server to the final
download location.

## Mitigations

Although GnuPG has several hundred command line options, it does not
offer a straightforward way to verify a signature against a specific
key.  In practice, there are three strategies to get around that
limitation:

### Setting the keyring

One way is to store the key in a file and use that file as a keyring.
Care has to be taken to disable the default keyring and options,
automatic key download from keyservers, as well as to enable batch
processing.

```
gpg_tmp=`mktemp -t yarn.gpg.XXXXXXXXXX`
curl -f -sS https://dl.yarnpkg.com/debian/pubkey.gpg | gpg --dearmor > "$gpg_tmp/pubkey.gpg"
gpg --no-options --no-default-keyring --no-auto-key-retrieve --no-auto-check-trustdb --trust-model=always --batch --no-tty --keyring "$gpg_tmp/pubkey.gpg" --verify -- "$1.asc" $1
```

This approach is often used, for example in Debian and systemd.
However, the exit code of `gpg` is not a reliable indicator for
operational success for all operations, so some applications that use
GnuPG more broadly than just for signature verification ignore the
exit code alltogether.

### Parse status lines

More difficult, but also more versatile, is parsing the output of the
`--status-fd` interface and looking for lines starting with "[GNUPG:]
VALIDSIG".  This approach is followed by `pass`, the Simple Password
Store, and also allows identifying specific subkeys.

### Temporary home directory

Another versatile approach is to use a separate home directory, with
its own configuration files and keyring databases.

```
gpg_tmp=`mktemp -t yarn.gpg.XXXXXXXXXX`
curl -f -sS https://dl.yarnpkg.com/debian/pubkey.gpg | gpg --homedir "$gpg_tmp/pubkey.gpg" --import
gpg --homedir "$gpg_tmp/pubkey.gpg" --no-auto-key-locate --no-auto-check-trustdb --trust-model=always --batch --no-tty --verify -- "$1.asc" $1
```

This approach is independent of the internal keyring format, but it
requires a temporary home directory.

<div style="clear: both"></div>
