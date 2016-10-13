+++
categories = [ "github", "crypto" ]
date = "2016-10-06T22:45:49-04:00"
description = "How I set up GPG-signed commits with Keybase"
keywords = ["gpg", "keybase", "git"]
title = "Verified GitHub commits with Keybase"

+++

A _while_ back, GitHub [announced](https://github.com/blog/2144-gpg-signature-verification)
support for GPG signature verification. Now, git has supported signing commits
and tags for a while, but it was a pain to manually verify signed commits, so I'm
not sure the average developer even considered it.

I saw the announcement, and messed around with it for a few minutes then forgot
about it, because I don't use GPG very often, and it hadn't yet clicked that
Keybase is really just a GPG management tool...

[Keybase](https://keybase.io) is great - it pretty much solves that pesky PGP/GPG
web-of-trust problem without forcing you to go to those awkward key signing parties
(are those still a thing?). Also it makes it dead-simple to encrypt/decrypt stuff,
especially now that they've launched the [Keybase File System](https://keybase.io/docs/kbfs).

Unfortunately, there isn't a super-simple path from Keybase to GitHub verified
commits, largely because there's two other tools involved (and frankly, doing
the _real_ work): Git and GnuPG.

So, for anyone else who wants to do this, but especially for my own memory, here's
how I set up Git and GnuPG on my Mac to sign commits with my Keybase identity...

## Pre-requisites

The tools I'm using are:

- Git
- GnuPG (especially `gpg` and `gpg-agent`)
- Keybase
- [Pinentry for GPG on Mac](https://github.com/GPGTools/pinentry-mac) (which is
  a part of GPGTools)

I installed all of these (except Keybase) with [Homebrew](https://brew.sh):

```console
$ brew install git gnupg pinentry-mac
...
```

I installed Keybase from https://keybase.io/download.

## Setup

I'm going to leave setting Keybase up as an exercise to the reader. My process
was a little convoluted:

1. figuring out how to get my GPG key out of Keybase
2. importing my Keybase key into `gpg`
3. configuring `git`
4. adding my key to GitHub
5. figuring out I needed to use `gpg-agent`
6. realizing that `gpg-agent` expires passphrases
7. finding `pinentry-mac`

So, I've put together a bit of a shortcut. Let's get started...

### Getting Keybase to cough up my key pair

The first step's easy:

```console
$ keybase pgp export > keybase-public.key
```

This gets my public key, but I need the secret key too. To do that,
there's a couple options - Keybase lets you export the private key (after
authenticating) from the web UI, but an easier commandline way is:

```console
$ keybase pgp export -s > keybase-private.key
```

It'll prompt for the passphrase.

Ok, now we have the key pair!

### Importing the key pair into the GPG keychain

This is probably the most involved step,

First the public:

```console
$ gpg --import keybase-public.key
```

Now private:

```console
$ gpg --allow-secret-key-import --import keybase-private.key
```

Ok, cool. If I type `gpg --list-secret-keys` now, I see my key:

```console
$ gpg --list-secret-keys
/Users/hairyhenderson/.gnupg/secring.gpg
-----------------------------
sec   4096R/9AB1B536 2014-11-19
uid                  keybase.io/dhenderson <dhenderson@keybase.io>
ssb   2048R/5DCE5AFA 2014-11-19
ssb   2048R/30CF03B1 2014-11-19
```

The one important bit to take away from that is the key ID - the bit after the `/`
in the `sec` entry: `9AB1B536` in my case.

In my case I only had one e-mail address associated with it - `dhenderson@keybase.io`.
I don't have any access to that mail account, and I certainly never sign commits
with that address. So I need to add my regular e-mail address to the key:

```console
$ gpg --edit <key id>
gpg (GnuPG) 2.0.30; Copyright (C) 2015 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

pub  4096R/9AB1B536  created: 2014-11-19  expires: never       usage: SCEA
                     trust: unknown       validity: unknown
sub  2048R/5DCE5AFA  created: 2014-11-19  expires: 2022-11-17  usage: S   
sub  2048R/30CF03B1  created: 2014-11-19  expires: 2022-11-17  usage: E   
[unknown] (2)  keybase.io/dhenderson <dhenderson@keybase.io>

gpg>
```

At this point, enter `adduid` and it'll prompt for various information:

```console
gpg> adduid
Real name: Dave Henderson
Email address: another@address.com
Comment:
You selected this USER-ID:
    "Dave Henderson <another@address.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
```

It'll prompt for the passphrase and then you're back at the `gpg>` prompt.

Now, one more change - not _strictly_ necessary, but will avoid weird warnings
later: set my own identity to ultimate trust. Use the `trust` command:

```console
gpg> trust
pub  4096R/9AB1B536  created: 2014-11-19  expires: never       usage: SCEA
                     trust: unknown      validity: unknown
sub  2048R/5DCE5AFA  created: 2014-11-19  expires: 2022-11-17  usage: S   
sub  2048R/30CF03B1  created: 2014-11-19  expires: 2022-11-17  usage: E   
[unknown] (1)  Dave Henderson <another@address.com>
[unknown] (2)  keybase.io/dhenderson <dhenderson@keybase.io>

Please decide how far you trust this user to correctly verify other users' keys
(by looking at passports, checking fingerprints from different sources, etc.)

  1 = I don't know or won't say
  2 = I do NOT trust
  3 = I trust marginally
  4 = I trust fully
  5 = I trust ultimately
  m = back to the main menu

Your decision? 5
```

Finish the session with the `save` command.

Ok, last step here: update Keybase with the updated key:

```console
$ keybase pgp update
▶ INFO Posting update for key 22600e46075b206041e8f730407ccdee9ab1b536.
▶ INFO Update succeeded for key 22600e46075b206041e8f730407ccdee9ab1b536.
```

### Configuring Git

This part's pretty straightforward. I just want to tell Git which key to use
by default:

```console
$ git config --global user.signingkey 9AB1B536
```

### Configuring GitHub

This part's _really_ straightforward. All I needed to do is open
https://github.com/settings/keys and create a new key by pasting in the contents
of my public key.

### Configuring `gpg-agent`

Almost there! I had to create a new `~/.gnupg/gpg-agent.conf` file. The contents
are:

```
use-standard-socket
pinentry-program /usr/local/bin/pinentry-mac
```

Then in `~/.gnupg/gpg.conf`, I added a line that says:

```
use-agent
```

That's it! Ready to sign commits now!

## Signing commits

And now, the fruit of our labours. I can just use the `-S` flag when I make a
commit:

```console
$ git commit -S -m "whoa, there's cryptographic proof this was written by me"
```

Now, when I first looked at this I wondered if `-S` was a "stronger" version of
`-s`, but it turns out they're completely unrelated. I've been used to "signing"
my commits already with the `git commit -s` flag, but this really only adds a
`Signed-off-by` line, which usually just implies acceptance of a [DCO](http://developercertificate.org/).

I guess I need to adjust my thinking about this, because - `-s` _really_ means
"sign off", which is pretty obvious once you look at the long form: `--signoff`.

On the other hand, `-S` (or `--gpg-sign`) adds a _cryptographic_ signature to
the commit. Cryptographically signing commits means something completely
different: It gives some proof that the commit was written by the person who
owns a certain GPG key.

When this is combined with GitHub's signature verification support, you get
fairly irrefutable proof that a commit was signed by the person who owns a given
GitHub account.

And then, we come full circle when we combine this with Keybase's various proof
mechanisms, which can prove that the person who owns a certain twitter account,
a certain website, and a certain reddit account is _also_ the same person who
owns that GitHub account.

👋🔏
