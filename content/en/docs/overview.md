---
title: Overview
weight: 1
description: >-
  What nspawn is, what it is not, and how images, machines, the hub and systemd
  fit together.
---

## What nspawn is

nspawn is a command line tool that manages
[systemd-nspawn](https://www.freedesktop.org/software/systemd/man/latest/systemd-nspawn.html)
machines with a docker-like workflow:

- Images are [OCI images](https://github.com/opencontainers/image-spec) that
  come from a registry. The default registry is the hub at `hub.nspawn.org`,
  where the nspawn.org team publishes images of common distributions built with
  [mkosi](https://github.com/systemd/mkosi). Docker Hub and any other registry
  work as well: `search` looks on the hub and on Docker Hub at once, and
  `login` keeps credentials per registry, the way `docker login` does.
- Layers are downloaded once, verified against their digests and shared between
  the machines that use them. A machine's root file system is assembled from
  them by a [backend](/docs/images/#backends): an overlayfs mount, a native
  `systemd.mstack` directory, or a flat copy. `create` makes more machines from
  an image that is already local, without touching the registry.
- Machines are started, inspected and stopped through the D-Bus APIs of
  [systemd-machined](https://www.freedesktop.org/software/systemd/man/latest/systemd-machined.service.html)
  and of systemd itself. `machinectl` and `importctl` are never called, and
  there is no daemon of nspawn's own: every machine is an ordinary
  `systemd-nspawn@NAME.service` unit that `machinectl`, `systemctl` and
  `journalctl` see like any other. A drop-in makes that unit call nspawn before
  the machine starts, once it runs and after it ends, so `machinectl start`, a
  unit enabled at boot or a crash get the same network setup and cleanup as
  `nspawn start` and `nspawn stop`.
- The docker flags you know apply to any kind of machine: `-p` publishes
  ports, `-e` sets variables, `-v` mounts host directories or named volumes,
  `--entrypoint` and the arguments after `--` change what an app runs.
- `build` runs mkosi on a directory with a `mkosi.conf` and imports the result
  as a local image; `push` uploads an image to a registry, skipping the layers
  that are already there.

It is a single binary written in Rust. Its own state lives under
`/var/lib/nspawn`, the assembled machines under `/var/lib/machines`, and
everything it generates on the host is a plain systemd unit, drop-in or
`.nspawn` settings file that you can read.

## Machines and apps

nspawn tells two kinds of images apart when it installs them:

| Kind | What the image contains | How it runs |
| --- | --- | --- |
| **boot** | An init system (systemd) and no other entrypoint, like the hub images | Booted with `--boot`, as `machinectl start` does. `shell` opens machined's login session inside; `exec` enters the machine's namespaces and needs nothing from it. |
| **app** | Anything else, for example an image from Docker Hub | Its entrypoint runs as PID 2 under nspawn's stub init, with the environment, working directory, user and stop signal from the OCI config. Its network namespace is prepared by nspawn before the program starts, so the network is there from the first instruction. `exec` and `shell` enter the namespaces. |

The kind is detected at `pull` or `build` time and can be forced with
`--mode boot` or `--mode app`. Both kinds join the bridge network, are listed
by `ps`, stopped by `stop` and read by `logs`; the differences are described
in [Machines](/docs/machines/).

## Networking

Every machine joins `nspawn0`, a docker0 style bridge that nspawn manages
itself: fixed addresses, NAT, published ports (`-p 8080:80`) and a generated
`/etc/hosts` with the names of the other machines. `--network host` shares the
host's network instead, and `--network veth` gives booted machines the classic
systemd-nspawn virtual ethernet pair configured by systemd-networkd. See
[Networking](/docs/networking/).

## The pieces

| Piece | Where | Role |
| --- | --- | --- |
| `nspawn` | [github.com/nspawn/nspawn](https://github.com/nspawn/nspawn) | The command line tool this documentation is about. |
| The hub | `hub.nspawn.org` | An OCI registry with the images the team publishes. It is the default registry of the tool. |
| mkosi definitions | [github.com/nspawn/mkosi-definitions](https://github.com/nspawn/mkosi-definitions) | The public mkosi configuration the hub images are built from. |
| Blog | [blog.nspawn.org](https://blog.nspawn.org/) | Release notes and news. |

## What nspawn is not

- It is not a container runtime of its own: systemd-nspawn runs the machines,
  systemd supervises them, machined tracks them. nspawn only drives them.
- It is not an orchestrator. There is no compose file, no service discovery
  beyond the names on the bridge, no scheduling.
- It does not build images by itself: `build` needs
  [mkosi](https://github.com/systemd/mkosi) installed on the host.
- It does not sign or verify signatures of images. Every blob is checked against
  the sha256 digest in the manifest while it downloads, and registries are
  reached over HTTPS only.
