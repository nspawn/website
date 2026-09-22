---
title: Getting started
weight: 2
description: >-
  Requirements, installation from source, a first machine pulled from the hub
  and an app from Docker Hub.
---

## Requirements

- A host with **systemd-nspawn** and **systemd-machined**. Any recent systemd
  works; 259 and 261 are the versions the test suite runs against. cgroup v2 is
  required.
- **overlayfs** for the `overlay` backend, which is the default on hosts older
  than systemd 261 and what app images always use. Without it images are
  extracted as flat directories.
- **iproute2** and **nftables** (`ip` and `nft`) for the bridge network. Nothing
  else: the bridge does not need systemd-networkd or NetworkManager on the host.
  Only `--network veth` needs systemd-networkd.
- **Root** for the commands that change the host: `pull`, `create`, `build`,
  `images rm`, `start`, `stop`, `login` and `logout` write below
  `/var/lib/machines`, `/var/lib/nspawn`, `/etc/systemd` and `/etc/nspawn`.
  Listing, `search`, `logs`, `exec`, `shell` and `hub` do not need it.
- **D-Bus inside a booted machine** for `shell`, which uses machined's login
  session; the hub images have it. `exec` enters the machine's namespaces and
  needs nothing inside.
- **mkosi** only if you want to [build images](/docs/building/).

## Installation

nspawn is a single binary. Build it with a Rust toolchain of version 1.85 or
newer:

```shell
git clone https://github.com/nspawn/nspawn.git
cd nspawn
cargo build --release
sudo install -Dm755 target/release/nspawn /usr/local/bin/nspawn
nspawn --version
```

Install it in `/usr/local/bin` or `/usr/bin`, not below a home directory: the
unit of every machine calls nspawn by the path it was installed from, and on
SELinux hosts a system service is refused a binary under `/home`. The release
profile uses thin LTO and strips the binary, so the result is small and depends
on nothing but glibc.

## A first machine

Find an image. `search` asks the hub and Docker Hub and prints, for every hit,
the reference `pull` takes:

```shell
nspawn search fedora
```

```text
 SOURCE          NAME                        DESCRIPTION                                  STARS  OFFICIAL
 hub.nspawn.org  fedora                      tags: 43, 44                                 -      -
 Docker Hub      docker.io/library/fedora    Official Docker builds of Fedora             1300   yes
 ...
```

Pull one. References without a registry part go to the hub, and the local name
is derived from the reference unless you pass `--name`:

```shell
sudo nspawn pull fedora:44
```

```text
hub.nspawn.org/fedora:44: manifest sha256:3f9c... with 1 layer(s), assembling as overlay
blob sha256:8a1e...: downloading
blob sha256:c0de...: downloading
image fedora-44 (boot image) is ready: nspawn start fedora-44
```

The blobs went to `/var/lib/nspawn`, the root file system of the machine is
mounted at `/var/lib/machines/fedora-44`, `/etc/systemd/nspawn/fedora-44.nspawn`
holds the settings nspawn boots it with, and a drop-in of
`systemd-nspawn@fedora-44.service` makes the unit call nspawn around its life.

```shell
sudo nspawn start fedora-44
nspawn ps
```

```text
 MACHINE    IMAGE                     MODE  COMMAND  STATE    UP   PID    NETWORK    OS
 fedora-44  hub.nspawn.org/fedora:44  boot  init     running  12s  48213  10.99.0.2  fedora
```

`start` returns once the machine's own systemd is up, so a command can follow
right away. `exec` runs it inside and brings back its exit code; `shell` opens
a login session as root:

```shell
nspawn exec fedora-44 -- systemctl is-system-running
nspawn shell fedora-44
```

The hub images log in as `root` without a password on the console. What the
machine printed is in `logs`, and `--inside` reads the journal of the machine
itself:

```shell
nspawn logs fedora-44
nspawn logs fedora-44 --inside -n 50
```

Stop it and, when you no longer need it, remove it. Removing an image also frees
the layers and blobs that no other image references:

```shell
sudo nspawn stop fedora-44
sudo nspawn images rm fedora-44
```

## An app from Docker Hub

Images without an init system run as apps: the entrypoint from the image runs
under a stub init, on the bridge like any other machine, so `-p` publishes its
ports on the host:

```shell
nspawn search nginx --source dockerhub
sudo nspawn pull docker.io/library/nginx:latest --name web
sudo nspawn start web -p 8080:80
curl -sI http://localhost:8080/ | head -1
nspawn logs web -f
sudo nspawn stop web
```

The port and any other flag given to `start` are remembered, so the next
`sudo nspawn start web` publishes it again. `stop` sends the image's stop
signal (`SIGQUIT` for nginx) to the program and kills the machine after ten
seconds if it is still there; `-t` changes the grace period.

Docker Hub limits anonymous pulls per address. `sudo nspawn login docker.io -u USER`
keeps your credentials for that registry only; see
[Registries and credentials](/docs/images/#registries-and-credentials).

## Next steps

- [Images and the hub](/docs/images/): references, search, credentials, backends,
  boot and app detection, `create`, where things are stored.
- [Machines](/docs/machines/): `start`, `stop`, `exec`, `shell`, `logs`, `ps`,
  entrypoints, environment and volumes.
- [Networking](/docs/networking/): the bridge, published ports, veth and host
  networking, firewalls.
- [Building images](/docs/building/): `build` and `push`.
- [Configuration](/docs/configuration/): `/etc/nspawn/nspawn.toml`, environment
  variables and flags.
