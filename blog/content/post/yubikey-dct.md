+++
categories = ["notes", "docker"]
date = "2016-06-30T22:19:39-04:00"
description = ""
keywords = ["yubikey", "docker"]
title = "Getting YubiKey working with Docker Content Trust on OS X"

+++

# Getting YubiKey working with Docker Content Trust on OS X

This wasn't as straightforward as I would've liked, so I'm documenting this here.

First, some super-useful sources:

- [Diogo Mónica's blog posting from DockerCon EU 2015][diogo-blog-dct] (_DOCKER CONTENT TRUST GETS HARDWARE SIGNING_)
- [docker/notary#779][notary-GH-779] (_Improve docs for typical CI usage with docker+notary_)
- The [YubiKey PIV CLI tool v1.4.0 zip][piv-tool-dl]
- [docker/toolbox#461][toolbox-GH-461] (from here I gleaned that
  `/usr/local/docker/lib` is understood by `notary`)

I only figured this out because the folks above did most of the work!

## Preparation...

### Install notary

Just a matter of grabbing the latest from https://github.com/docker/notary/releases
and putting the `notary` binary in the path somewhere (like `~/bin` in my case).

### Install the YubiKey PIV shared libraries

- First download the [PIV CLI tool zip][piv-tool-dl] and extract it somewhere
```console
$ curl -OL https://developers.yubico.com/yubico-piv-tool/Releases/yubico-piv-tool-1.4.0-mac.zip
...
$ unzip yubico-piv-tool-1.4.0-mac.zip
```

- We only need the contents of `lib`, and it turns out it can simply be copied
  into `/usr/local/docker/lib`:
```console
$ mkdir -p /usr/local/docker/lib
$ cp -av lib/ /usr/local/docker/lib
```

- That's it - now make sure this worked:
```console
$ DOCKER_CONTENT_TRUST=1 docker -D pull alpine
Using default tag: latest
DEBU[0000] reading certificate directory: /Users/myuser/.docker/tls/notary.docker.io
DEBU[0000] Initialized PKCS11 library /usr/local/docker/lib/libykcs11.dylib and started HSM session
...
```

- The key is that `started HSM session` bit

## Configuration...

Ok now we're ready to create a key:

- `notary key generate`
  - give it a complex passphrase stored in a password manager (I like
    [1Password](https://1password.com))
- Verify that it's there:
```console
$ notary key list
ROLE   GUN    KEY ID        LOCATION
------------------------------------------
root         deadbeef....   file (/Users/myuser/.docker/trust/private)  
root         deadbeef....   yubikey    
```
  - that second entry says it's in the yubikey.

- Maybe a good idea to back up the root key now (to a _secure offline location!_), with `notary key backup`
```console
$ notary key backup /Volumes/SomeUSBStick/archivename.zip
...
```
  - this will re-encrypt with a new passphrase into a zip file. Obviously we gotta keep that passphrase handy too!

- Last, we can delete the on-disk key. It's unnecessary since we'll use the YubiKey's copy.
```console
$ rm ~/.docker/trust/private/root_keys/deadbeef....key
```

## Signing on push

And now for the magic:

```console
$ export DOCKER_CONTENT_TRUST=1
$ docker build -t hairyhenderson/dct-test:latest
...
$ docker push hairyhenderson/dct-test:latest
The push refers to a repository [docker.io/hairyhenderson/dct-test]
e520aba5adf5: Layer already exists
917d421641b4: Layer already exists
e9d17ff7ca16: Layer already exists
63bcbfab7dc5: Layer already exists
af21bd4b5bb1: Layer already exists
4b0b3b9ff599: Layer already exists
d1e800db26c7: Layer already exists
42755cf4ee95: Layer already exists
latest: digest: sha256:deadbeef25599d9a37bd452ba6918ac198693fe4b9f9822d0a4bf54735469f04 size: 1974
Signing and pushing trust metadata
Please touch the attached Yubikey to perform signing.
Enter passphrase for new repository key with ID 7ff917d (docker.io/hairyhenderson/dct-test):
Repeat passphrase for new repository key with ID 7ff917d (docker.io/hairyhenderson/dct-test):
Please touch the attached Yubikey to perform signing.
Finished initializing "docker.io/hairyhenderson/dct-test"
Successfully signed "docker.io/hairyhenderson/dct-test":latest
```

This is as far as I've gotten... Maybe more to come later!

[diogo-blog-dct]: https://blog.docker.com/2015/11/docker-content-trust-yubikey/
[notary-GH-779]: https://github.com/docker/notary/issues/779
[piv-tool-dl]: https://developers.yubico.com/yubico-piv-tool/Releases/yubico-piv-tool-1.4.0-mac.zip
[toolbox-GH-461]: https://github.com/docker/toolbox/issues/461
