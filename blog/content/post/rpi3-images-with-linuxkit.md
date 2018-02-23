+++
title = "Building images for Raspberry Pi 3 with LinuxKit"
description = ""
keywords = ["arm", "linuxkit", "docker", "moby", "rpi", "arm64"]
categories = ["docker", "raspberry pi"]
date = "2017-10-04T16:51:00-04:00"
draft = true
+++

# Building images for Raspberry Pi 3 with LinuxKit!

Links:
https://github.com/worksonarm/cluster/
https://github.com/moby/tool
https://github.com/DieterReuter/workshop-raspberrypi-64bit-os
https://github.com/hypriot/image-builder
https://github.com/hypriot/image-builder-rpi
https://github.com/DieterReuter/image-builder-rpi64
https://github.com/hypriot/image-builder-raw
https://www.raspberrypi.org/documentation/configuration/config-txt/boot.md
https://www.raspberrypi.org/documentation/linux/kernel/building.md

https://raspberrypi.stackexchange.com/a/7238/9644

https://gist.github.com/rn/08d13b7ed30ab5fbde9a6dcaa24831ce

General steps involved:
- get a 64-bit Linux running on the RPi3
- get the `moby` tool built on the RPi3
- use `moby` to build a LinuxKit image with basic stuff
  - challenge: figure out the bootloader

This all started with an [_innocent_ question](https://dockercommunity.slack.com/archives/C51K0GDU7/p1507135315000111) in the `#linuxkit` channel in the
Docker Community Slack workspace:

> hairyhenderson [12:41]
>
> _Does anyone know if it's possible to use LinuxKit to build an image for a Raspberry Pi 3? (ARMv8, so either 32 or 64-bit would be fine)_

## Bootstrapping

Right now there's no way to "cross-assemble" [LinuxKit][] images for different
target architectures (i.e. build an `aarch64` image on an `amd64` system), so I need 
to set up an `aarch64` computer capable of building and running the `moby` tool.

One option (which costs money) is to use a cloud-based VM, for example with
[Packet.net][] or [Scaleway][]. You don't need it for a long time - just long enough 
to build the `moby` tool and run it, so the cost could be quite reasonable.

The option I'm choosing is free, because I already own a Raspberry Pi 3, but involves 
a bit more setup.

### Getting console access

_(This step is optional, and it's probably easier to just plug in a keyboard
and monitor)_

Since WiFi won't automatically connect, I'm going to connect to the serial console
with the help of an [Adafruit FTDI Friend][].

Then I connect the FTDI Friend to the Pi:
- Ground to Ground
- Rx to TxD
- Tx to RxD

I start up a console, using `screen`:

```console
$ screen /dev/cu.usbserial-AH02F0IR 115200
```

Now I'm ready to issue commands to the Pi once it boots.

### Install a 64-bit OS

First I need to boot a 64-bit OS. Right now, Raspbian (the default and most 
well-supported Linux distribution for the Pi) is 32-bit-only. So, I'll have to go with
something a little more experimental.

Thankfully, [Dieter Reuter][] has done a lot of the work for us in creating a series
of articles and GitHub repos on how to create a 64-bit RPi3 image based on [HypriotOS][].

To get this running, I downloaded the latest released image build from https://github.com/DieterReuter/image-builder-rpi64/releases (the `.img.zip` file), then I used Hypriot's [`flash`](https://raw.githubusercontent.com/hypriot/flash/master/Darwin/flash) script to write the image to my SD card:

```console
$ curl -sSLO https://github.com/DieterReuter/image-builder-rpi64/releases/download/v20171007-083124/hypriotos-rpi64-v20171007-083124.img.zip
$ unzip hypriotos-rpi64-v20171007-083124.img.zip
$ flash -s <my SSID> -p <WiFi password> -d disk2 hypriotos-rpi64-v20171007-083124.img
```

We now have a Pi3 running 64-bit HypriotOS, with WiFi!

### Building the `moby` and `linuxkit` tools

<!-- A super-convenient feature of HypriotOS is that it's designed from the ground up to
run Docker. As of right now, the image has a pretty old version of Docker: 1.13.1,
but it'll work for our purposes.

This makes building the `moby` binary pretty straightforward. And, because I can, I'm going to build it as part of a `docker build`. First I create a simple `Dockerfile`:

```dockerfile
FROM golang:1.9

RUN go get -u github.com/moby/tool/cmd/moby
```

Then we build it:

```console
$ docker build -t moby .
Sending build context to Docker daemon 31.23 kB
Step 1/2 : FROM golang:1.9
 -- -> 6fc925adbae4
Step 2/2 : RUN go get -u github.com/moby/tool/cmd/moby
 -- -> Running in a003a181af0e
 -- -> a29e77d6df06
Removing intermediate container a003a181af0e
```

And then we can extract the binary:

```console
$ docker create --name m moby
$ docker cp m:/go/bin/moby .
$ ./moby version
moby verison unknown
commit: unknown
```

Awesome! We're done bootstrapping, we can start building LinuxKit images now! -->

We need to build `moby` and `linuxkit` for the `arm64`/`linux` architecture/os. Thankfully, Go makes this super simple:

```console
$ go get -u github.com/moby/tool/cmd/moby
$ cd ~/go/src/github.com/moby/tool/cmd/moby
$ GOARCH=aarch64 GOOS=linux go build
$ go get -u github.com/linuxkit/linuxkit/src/cmd/linuxkit
$ cd ~/go/src/github.com/linuxkit/linuxkit/src/cmd/linuxkit
$ GOARCH=aarch64 GOOS=linux go build
```

Now we can just `scp` them to the Pi.

## Building an image with LinuxKit

As a test, I've pulled down the `minimal.yml` example spec from the
[LinuxKit repo][LinuxKit].

First I'm going to try building a `raw` image to see what I can do with it...

```console
$ ./moby build -format kernel+initrd minimal.yml
...
fatal error: runtime: out of memory
```

Uh oh! Looks like 1GB of RAM isn't enough to build a LinuxKit image.

No problem - cloud to the rescue! I'm using Packet.net's [Type 2A][] servers,
which are _very_ powerful (96 ARMv8 cores at 2GHz, 128GBs of RAM, and 20Gbps network), but reasonably priced - especially with spot pricing.

It does get tedious manually installing software on bare-metal servers, so for now
I'm using a User Data script that looks like this (and is mostly ripped off from 
[Rolf Neugebauer](https://gist.github.com/rn/08d13b7ed30ab5fbde9a6dcaa24831ce):

```sh
#!/bin/bash
apt-get update
# Install some utilities that'll be useful later
apt-get install -y \
  jq \
  expect \
  kvm \
  apt-transport-https \
  ca-certificates \
  curl \
  software-properties-common
  # automake \
  # libglib2.0-dev \
  # bison \
  # flex

# Add Docker CE arm64 testing
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository \
   "deb [arch=arm64] https://download.docker.com/linux/ubuntu \
   xenial \
   test"
apt-get update
apt-get install -y docker-ce

# Add a user
USER=hairyhenderson
adduser --disabled-password $USER
usermod -aG docker $USER
usermod -aG kvm $USER
usermod -aG sudo $USER

# share the SSH key
mkdir -p /home/$USER/.ssh
chmod 700 /home/$USER/.ssh
cp /root/.ssh/authorized_keys /home/$USER/.ssh/
chmod 600 /home/$USER/.ssh/authorized_keys
chown -R $USER:$USER /home/$USER/.ssh

touch /var/lib/cloud/instance/warnings/.skip

echo '%sudo ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/nopasswd-sudo-group
echo -ne '\nPermitRootLogin no\n' >> /etc/ssh/sshd_config
systemctl restart ssh.service

# Update qemu
# wget http://download.qemu-project.org/qemu-2.10.0-rc1.tar.xz
# tar xvJf qemu-2.10.0-rc1.tar.xz
# cd qemu-2.10.0-rc1
# ./configure
# make
```

```console
docker pull hairyhenderson/rpi3-dev

```


[LinuxKit]: https://github.com/linuxkit/linuxkit
[Packet.net]: https://www.packet.net/bare-metal/servers/type-2a/
[Type 2A]: https://www.packet.net/bare-metal/servers/type-2a/
[Scaleway]: https://www.scaleway.com/armv8-cloud-servers/
[Alpine downloads]: https://alpinelinux.org/downloads/
[Alpine install instructions]: https://wiki.alpinelinux.org/wiki/Create_a_bootable_SDHC_from_a_Mac
[Adafruit FTDI Friend]: https://www.adafruit.com/product/284
