---
title: Command reference
linkTitle: Reference
weight: 8
description: >-
  Every command and option of nspawn 1.1.0.
---

`nspawn --help` and `nspawn COMMAND --help` print the same information. Errors
are printed as `error: ...` on standard error and the exit status is 1;
`exec` exits with the status of the command it ran.

Every command but `daemon`, `completions` and the unit hooks is a call to the
[service](/docs/overview/#the-service) on the system bus, which asks polkit
whether the caller may take the action: the listings (`images ls`, `ps`,
`machines ls`, `inspect`, `network ls`, `volume ls`) ask for
`org.nspawn.inspect`; everything else, `search` and `hub` included, asks for
`org.nspawn.manage`. Both are for administrators by default, and root is never
asked.

## Global options

These are accepted by every command and can also come from the environment:

| Option | Environment | Meaning |
| --- | --- | --- |
| `--registry REGISTRY` | `NSPAWN_REGISTRY` | Registry (hub) for image references without a host part. |
| `--ca-cert FILE` | `NSPAWN_CA_CERT` | Extra CA certificate (PEM) to trust when talking to the registry. |
| `--config FILE` | `NSPAWN_CONFIG` | Configuration file; see [Configuration](/docs/configuration/). |
| `-h`, `--help` | | Help. |

`login` and `logout` take the registry as their argument, so there `--registry`
has to come before the subcommand: `nspawn --registry hub.example login -u me`.
`-V`, `--version` belongs to `nspawn` itself, not to the subcommands.

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
systemd-machined. On a terminal a bar shows how far each blob got, with its
size, speed and time left; in a pipe or a log only the lines are written.

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
              [--entrypoint PROGRAM] [-e VAR[=VALUE]]... [-v SOURCE:TARGET[:ro]]... [-l KEY=VALUE]...
              [--restart POLICY] [-m SIZE] [--cpus N] [--pids-limit N] [-f] [-- ARGUMENTS...]
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
| `-l`, `--label KEY=VALUE` | Label the machine, on top of the image's own labels, like `docker --label`. Not inherited from the source. |
| `--restart`, `-m`, `--cpus`, `--pids-limit` | Restart policy and limits, as for [start](#start). Not inherited from the source. |
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
recorded for the image. The destination needs a tag, not a digest. On a
terminal a bar shows how far each blob got, as with `pull`.

## images

Manage local images.

### images ls

```text
nspawn images ls [--json]
```

Lists the local images known to systemd-machined: name, type, and for the ones
nspawn installed the backend, origin (`pull`, `build` or `create`), source
reference, size and whether the image is read-only. `--json` prints the list as
the service returns it. `images list` is an alias.

### images rm

```text
nspawn images rm NAME...
```

Removes local images and the layers and blobs nobody uses any more. Refuses
the image of a running machine, or of one its restart policy is bringing back.
Named volumes are kept. A pulled image is a machine too, so this is the same as
[rm](#rm) without `--force`.

## rm

```text
nspawn rm [-f] NAME...
```

Removes machines, like `docker rm`: the record, the tree, the unit files, the
boot link a restart policy made, and the layers and blobs nobody else uses.
Every name is tried; one that cannot be removed is reported at the end. Named
volumes are kept, and each one kept is mentioned.

| Option | Meaning |
| --- | --- |
| `-f`, `--force` | Stop a running (or restarting) machine first, with SIGKILL, instead of refusing it. |

## ps, machines ls

```text
nspawn ps [-a] [--json]
nspawn machines ls [-a] [--json]
```

Lists the running machines, like `docker ps`: name, image, mode, command,
state, uptime, leader PID, network and OS. A machine whose restart policy is
bringing it back shows as `restarting`, even without `-a`. `-a`, `--all` also
lists the nspawn machines that are not running: `stopped`, or `starting` and
`closing` while their unit comes up or goes down. `--json` prints what the service returns for
each machine, its whole record included. `machines list` is an alias of
`machines ls`.

## inspect

```text
nspawn inspect NAME...
```

Prints everything nspawn knows about machines or images, running or not, as a
JSON array with one object per name, like `docker inspect`: the record (image
reference, digest, backend, mode, network, address, ports, volumes,
environment, command, labels, restart policy, limits) and, for a running
machine, its state, start time, leader PID and OS. The keys are those of the
[D-Bus interface](https://github.com/nspawn/nspawn/blob/master/docs/DBUS.md).

## start

```text
nspawn start NAME [--network bridge|veth|host] [-p HOST:CONTAINER[/udp]]...
             [--entrypoint PROGRAM] [-e VAR[=VALUE]]... [-v SOURCE:TARGET[:ro]]...
             [-l KEY=VALUE]... [--restart POLICY] [-m SIZE] [--cpus N] [--pids-limit N]
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
| `-l`, `--label KEY=VALUE` | Label the machine, on top of the image's own labels. Repeatable; `none` forgets them. |
| `--restart no\|on-failure\|always\|unless-stopped` | Restart policy, like `docker --restart`. `always` and `unless-stopped` also start the machine at boot; `nspawn stop` takes an `unless-stopped` machine off the boot list until the next `start`. Applied at the next start. |
| `-m`, `--memory SIZE` | Memory limit of the whole machine, like `docker -m`: `512m`, `2g` (at least `4m`), and as much swap again; `0` removes it. Applied at the next start. |
| `--cpus N` | CPU limit of the whole machine, like `docker --cpus`: `0.5`, `2`; `0` removes it. Applied at the next start. |
| `--pids-limit N` | Most processes and threads the machine may have (at least 16 for a booted machine); `0` removes the limit. Applied at the next start. |
| `--image-command` | Forget the remembered entrypoint and arguments and run the image's own again. |
| `--no-wait` | Do not wait for a booted machine's init to be up before returning. Its registration is still awaited, so that ports and firewall rules can be applied. |
| `-- ARGUMENTS...` | App images: replace the image's cmd; they follow its entrypoint, as with docker. |

## stop

```text
nspawn stop NAME [-f] [-t TIMEOUT] [--no-wait]
```

Powers off a running machine: the image's stop signal to the program of an
app, a poweroff request to a booted machine. Stopping a machine that already
ended only cleans up after it. A machine with a restart policy stays stopped,
and one that was waiting to be restarted is stopped too. An `unless-stopped`
machine is also taken off the boot list until the next `start`; an `always`
one stays enabled and starts again at the next boot (`rm` takes that away).

| Option | Meaning |
| --- | --- |
| `-f`, `--force` | Kill every process at once instead of asking the machine to stop. |
| `-t`, `--timeout TIMEOUT` | App images: seconds to wait after the stop signal before killing the machine. Default: 10. |
| `--no-wait` | Return right after the stop request, without waiting for the machine to be gone and without the kill after the timeout. |

## exec

```text
nspawn exec MACHINE [-u USER] COMMAND...
```

Runs a command inside a running machine of either kind, attached to the
terminal, in the machine's namespaces, with the image's environment and the
`-e` variables. The program is found on the machine's `PATH` and runs with the
machine's capabilities, like its own processes. Exits with the command's
status.

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

## cp

```text
nspawn cp MACHINE:PATH DESTINATION
nspawn cp SOURCE MACHINE:PATH
```

Copies files and directories between the host and a machine, like
`docker cp`. The machine may be running, or stopped if it is an overlay or flat
machine; a stopped mstack machine has no tree on the host until it runs.

- An existing directory as the destination receives the source under its own
  name; anything else is the name of the copy, whose parent must exist. A
  destination ending in `/` must be a directory.
- `DIR/.` as the source copies the contents of DIR instead of DIR itself.
- What goes into a machine belongs to its root, whatever user namespace the
  machine runs in; what comes out belongs to the user who ran `cp`. Modes and
  modification times are kept.
- Paths inside the machine are resolved inside it: a link there, absolute or
  not, never leads to the host. Links are copied as links; devices, sockets and
  fifos are left out.
- A relative path after `MACHINE:` starts at the machine's root. A local path
  with a colon is written `./a:b`.

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
nspawn network ls [--json]
```

Lists the machines on the bridge with their addresses and published ports.
`--json` prints the bridge and the machines as the service returns them.
`network list` is an alias.

## volume

Named volumes, the directories `-v NAME:/path` makes under
`/var/lib/nspawn/volumes`. Removing a machine keeps them.

### volume ls

```text
nspawn volume ls [--json]
```

Lists the named volumes with the machines whose records mount them, when each
was made and its path. `--json` prints the list as the service returns it.
`volume list` is an alias.

### volume create

```text
nspawn volume create NAME
```

Makes a volume ahead of its first use (`start` makes it otherwise): a
directory owned by root, mode 0755. One that exists already is fine. Names have
letters, digits, `_`, `.` and `-`, not starting with a dot.

### volume rm

```text
nspawn volume rm NAME...
```

Removes volumes no machine uses. A volume a machine still names is refused
until that machine is started with other volumes (or `-v none`) or removed.

### volume prune

```text
nspawn volume prune [-f]
```

Removes every volume no machine uses once a yes comes on standard input, as
docker does: from a script, whose input ends without one, nothing is removed.
`-f` removes them without asking.

## daemon

```text
nspawn daemon [--install] [--idle-exit SECONDS]
```

Serves `org.nspawn` on the system bus. The bus starts it on demand, so this is
not a command to type; `--install` is.

| Option | Meaning |
| --- | --- |
| `--install` | Write the bus policy, the polkit actions, the activation file and the unit that let the bus start this binary on demand, then return. A package does the same. |
| `--idle-exit SECONDS` | Exit after this long without a call or a running job. Default: 60; `0` keeps serving. |

## completions

```text
nspawn completions bash|elvish|fish|powershell|zsh
```

Writes the completions for that shell on standard output; they come from the
same definition the command line itself is built from. The packages install
them, so this is for a binary you built yourself:

```shell
nspawn completions bash > ~/.local/share/bash-completion/completions/nspawn
```

`man nspawn` is the same reference as this page, generated the same way.

### Unit hooks

`nspawn network prepare NAME`, `nspawn network publish NAME` and
`nspawn network release NAME` are what the drop-in of
`systemd-nspawn@NAME.service` runs as `ExecStartPre`, `ExecStartPost` and
`ExecStopPost`. They are not meant to be typed and are hidden from `--help`.
