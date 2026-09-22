---
title: Command reference
linkTitle: Reference
weight: 8
description: >-
  Every command and option of nspawn 0.2.0.
---

`nspawn --help` and `nspawn COMMAND --help` print the same information. Errors
are printed as `error: ...` on standard error and the exit status is 1;
`exec` exits with the status of the command it ran.

Commands that change the host need root: `pull`, `create`, `build`,
`images rm`, `start`, `stop`, `login` and `logout`. The others do not.

## Global options

These are accepted by every command and can also come from the environment:

| Option | Environment | Meaning |
| --- | --- | --- |
| `--registry REGISTRY` | `NSPAWN_REGISTRY` | Registry (hub) for image references without a host part. |
| `--ca-cert FILE` | `NSPAWN_CA_CERT` | Extra CA certificate (PEM) to trust when talking to the registry. |
| `--config FILE` | `NSPAWN_CONFIG` | Configuration file; see [Configuration](/docs/configuration/). |
| `-h`, `--help` | | Help. |
| `-V`, `--version` | | Version. |

## hub

Query the hub (an OCI registry).

### hub ls

```text
nspawn hub ls [FILTER] [--no-tags]
```

Lists the repositories of the registry with their tags. `FILTER` keeps the
repositories whose name contains that text; `--no-tags` skips the tag query of
every repository, which is faster on big registries. `hub list` is an alias.

### hub tags

```text
nspawn hub tags REPOSITORY
```

Prints the tags of one repository, one per line.

## search

```text
nspawn search TERM [--source hub|dockerhub] [-n LIMIT]
```

Finds images on the hub and on Docker Hub, like `docker search`, and prints
every hit with its source and the reference `pull` takes.

| Option | Meaning |
| --- | --- |
| `TERM` | Text to look for in image names. |
| `--source hub\|dockerhub` | Only one source instead of both. |
| `-n`, `--limit LIMIT` | Results per source. Default: 25. |

## login

```text
nspawn login [REGISTRY] [-u USERNAME] [--password-stdin]
```

Checks credentials against a registry, like `docker login`, and keeps them in
`/etc/nspawn/auth.json` for `pull`, `push`, `search` and `hub`.

| Option | Meaning |
| --- | --- |
| `REGISTRY` | Registry host, for example `docker.io` or `hub.nspawn.org`. Default: the hub. |
| `-u`, `--username USERNAME` | User name; asked for on the terminal when missing. |
| `--password-stdin` | Read the password from standard input instead of the terminal. |

## logout

```text
nspawn logout [REGISTRY]
```

Forgets the credentials stored for a registry (the hub by default).

## pull

```text
nspawn pull REFERENCE [-n NAME] [--backend BACKEND] [--mode MODE] [-f]
```

Downloads an image from the hub or another registry and makes it available to
systemd-machined.

| Option | Meaning |
| --- | --- |
| `REFERENCE` | `[registry/]repository[:tag|@digest]`, for example `fedora:44` or `docker.io/library/nginx`. |
| `-n`, `--name NAME` | Local image name. Default: derived from the reference, for example `fedora-44`. |
| `--backend auto\|overlay\|flat\|mstack` | How to assemble the image on this host. Default: `auto`, or the `backend` of the configuration file. App images are assembled as `overlay` even when `mstack` is chosen. |
| `--mode auto\|boot\|app` | Whether the image boots an init system or runs a single program. Default: `auto`. |
| `-f`, `--force` | Replace an existing image with the same name. |

## create

```text
nspawn create SOURCE NAME [--backend BACKEND] [--network bridge|veth|host] [-p HOST:CONTAINER[/udp]]...
              [--entrypoint PROGRAM] [-e VAR[=VALUE]]... [-v SOURCE:TARGET[:ro]]... [-f] [-- ARGUMENTS...]
```

Makes another machine from a local image, like `docker create`, without
touching the registry. The layers are shared with the source.

| Option | Meaning |
| --- | --- |
| `SOURCE` | Local image to start from: its name, or the reference it was pulled from. |
| `NAME` | Name of the new machine. |
| `--backend BACKEND` | How to assemble it. Default: like the source. |
| `--network bridge\|veth\|host` | Network of the new machine. Default: like the source. |
| `-p`, `--publish HOST:CONTAINER[/udp]` | Ports to publish on the host, like `start -p`. Not inherited from the source. |
| `--entrypoint PROGRAM` | Replace the image's entrypoint; an empty string runs the arguments alone. App images only. |
| `-e`, `--env VAR[=VALUE]` | Environment for the program, `VAR=value` or `VAR` copied from the calling shell, like `docker -e`. App images only. |
| `-v`, `--volume SOURCE:TARGET[:ro]` | Mount a host directory or a named volume, like `docker -v`. |
| `-f`, `--force` | Replace an existing machine with the same name. |
| `-- ARGUMENTS...` | App images: replace the image's cmd; they follow its entrypoint, as with docker. |

## build

```text
nspawn build [DIRECTORY] -t TAG [-n NAME] [-d DISTRIBUTION] [-r RELEASE]
             [--profile PROFILE]... [--backend BACKEND] [--mode MODE] [-f]
             [--keep-output] [-- MKOSI_ARGS...]
```

Builds an image with mkosi and makes it available locally, ready to push.
Needs mkosi.

| Option | Meaning |
| --- | --- |
| `DIRECTORY` | Directory with the mkosi configuration (`mkosi.conf`, `mkosi.conf.d`, ...). Default: `.`. |
| `-t`, `--tag TAG` | Reference for the result, for example `myapp:1` or `hub.example/team/app:2`. Required. |
| `-n`, `--name NAME` | Local image name. Default: derived from the tag. |
| `-d`, `--distribution DISTRIBUTION` | Distribution to build (`mkosi --distribution`). |
| `-r`, `--release RELEASE` | Release to build (`mkosi --release`). |
| `--profile PROFILE` | mkosi profile to enable; repeatable. |
| `--backend`, `--mode`, `-f` | As for `pull`. |
| `--keep-output` | Keep the mkosi output directory instead of deleting it after the import. |
| `-- MKOSI_ARGS...` | Extra arguments passed to mkosi verbatim. |

## push

```text
nspawn push IMAGE [--to REFERENCE]
```

Uploads a local image to the hub or another registry, with the credentials
stored for it. `IMAGE` is a local image name, or the reference it was pulled
from or built as. `--to` pushes it under a different reference than the one
recorded for the image. The destination needs a tag, not a digest.

## images

Manage local images.

### images ls

```text
nspawn images ls
```

Lists the local images known to systemd-machined, with nspawn's backend, origin
(`pull`, `build` or `create`) and source reference for the ones it installed.
`images list` is an alias.

### images rm

```text
nspawn images rm NAME...
```

Removes local images and the layers and blobs nobody uses any more. Refuses
the image of a running machine. Named volumes are kept.

## ps, machines ls

```text
nspawn ps [-a]
nspawn machines ls [-a]
```

Lists the running machines, like `docker ps`: image, mode, command, state,
uptime, leader PID, network and OS. `-a`, `--all` also lists the nspawn
machines that are not running. `machines list` is an alias of `machines ls`.

## start

```text
nspawn start NAME [--network bridge|veth|host] [-p HOST:CONTAINER[/udp]]...
             [--entrypoint PROGRAM] [-e VAR[=VALUE]]... [-v SOURCE:TARGET[:ro]]...
             [--image-command] [--no-wait] [-- ARGUMENTS...]
```

Boots an image as a machine. Every option is remembered for the next start.

| Option | Meaning |
| --- | --- |
| `--network bridge\|veth\|host` | Network of the machine: the bridge (default), a veth pair configured by systemd-networkd on the host (booted images only), or the host's own network. |
| `-p`, `--publish HOST:CONTAINER[/udp]` | Publish a port on the host, like `docker -p`. Repeatable; `none` forgets them all. |
| `--entrypoint PROGRAM` | Replace the image's entrypoint; an empty string runs the arguments alone. App images only. |
| `-e`, `--env VAR[=VALUE]` | Environment for the program, `VAR=value` or `VAR` copied from the calling shell. Repeatable; `none` forgets them. App images only. |
| `-v`, `--volume SOURCE:TARGET[:ro]` | Mount a host directory or a named volume. Repeatable; `none` forgets them. |
| `--image-command` | Forget the remembered entrypoint and arguments and run the image's own again. |
| `--no-wait` | Do not wait for a booted machine's init to be up before returning. Its registration is still awaited, so that ports and firewall rules can be applied. |
| `-- ARGUMENTS...` | App images: replace the image's cmd; they follow its entrypoint, as with docker. |

## stop

```text
nspawn stop NAME [-f] [-t SECONDS] [--no-wait]
```

Powers off a running machine: the image's stop signal to the program of an
app, a poweroff request to a booted machine. Stopping a machine that already
ended only cleans up after it.

| Option | Meaning |
| --- | --- |
| `-f`, `--force` | Kill every process at once instead of asking the machine to stop. |
| `-t`, `--timeout SECONDS` | App images: seconds to wait after the stop signal before killing the machine. Default: 10. |
| `--no-wait` | Return right after the stop request, without waiting for the machine to be gone and without the kill after the timeout. |

## exec

```text
nspawn exec MACHINE [-u USER] COMMAND...
```

Runs a command inside a running machine of either kind, attached to the
terminal, in the machine's namespaces, with the image's environment and the
`-e` variables. The program is found on the machine's `PATH`. Exits with the
command's status.

| Option | Meaning |
| --- | --- |
| `-u`, `--user USER` | User inside the machine. Default: `root`. |
| `COMMAND...` | Command and arguments. |

## shell

```text
nspawn shell MACHINE [-u USER]
```

Opens an interactive shell inside a running machine as `USER` (default
`root`): machined's login session for booted machines, `/bin/sh` in the
machine's namespaces for app machines.

## logs

```text
nspawn logs MACHINE [-f] [-n N] [--since WHEN] [-t] [--all] [--inside]
```

Shows what a machine printed, like `docker logs`.

| Option | Meaning |
| --- | --- |
| `-f`, `--follow` | Keep printing new output; starts from the last 10 lines unless `--lines` says otherwise. |
| `-n`, `--lines N` | Only the last N lines. |
| `--since WHEN` | Only output newer than this, in `journalctl --since` syntax, for example `"10 min ago"`. |
| `-t`, `--timestamps` | Prefix every line with its timestamp. |
| `--all` | Also show what systemd says about the machine's service: start, stop, failures. |
| `--inside` | Booted machines only: read the machine's own journal instead of its console output. |

## network

The bridge network shared by the machines.

### network up

```text
nspawn network up
```

Creates the bridge with its NAT rules and firewall exceptions, and brings the
published ports in line with the machines that run. `start` does it too; this
is useful at boot and for troubleshooting.

### network ls

```text
nspawn network ls
```

Lists the machines on the bridge with their addresses and published ports.
`network list` is an alias.

### Unit hooks

`nspawn network prepare NAME`, `nspawn network publish NAME` and
`nspawn network release NAME` are what the drop-in of
`systemd-nspawn@NAME.service` runs as `ExecStartPre`, `ExecStartPost` and
`ExecStopPost`. They are not meant to be typed and are hidden from `--help`.
