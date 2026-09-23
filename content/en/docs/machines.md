---
title: Machines
weight: 4
description: >-
  Starting and stopping machines, entrypoints, environment and volumes,
  running commands inside, reading their output, and how boot and app images
  differ.
---

Every one of these commands is a call to the
[service](/docs/overview/#the-service) on the system bus, which asks polkit
whether you may: `sudo` always works, and a rule can let your group through
without a password. The examples here use `sudo`.

Every machine nspawn starts is the systemd unit `systemd-nspawn@NAME.service`,
registered with systemd-machined under its name. `machinectl list`,
`machinectl status NAME`, `systemctl status systemd-nspawn@NAME` and
`journalctl -u systemd-nspawn@NAME` all work on it; nspawn adds the
docker-like commands on top.

The unit also carries a drop-in, `nspawn-hooks.conf`, that calls nspawn around
the machine's life: `ExecStartPre` prepares its network and settings,
`ExecStartPost` publishes its ports once it is registered, and `ExecStopPost`
releases everything however the machine ended. So `machinectl start NAME`,
`systemctl enable systemd-nspawn@NAME` for a machine that comes up at boot, a
program that exits on its own or a crash all behave like `nspawn start` and
`nspawn stop`. The drop-in names the nspawn binary that wrote it, which is why
nspawn belongs in `/usr/local/bin` or `/usr/bin`.

## start

```shell
sudo nspawn start NAME [--network bridge|veth|host] [-p HOST:CONTAINER[/udp]]...
                  [--entrypoint PROGRAM] [-e VAR[=VALUE]]... [-v SOURCE:TARGET[:ro]]...
                  [--image-command] [--no-wait] [-- ARGUMENTS...]
```

`start` regenerates `/etc/systemd/nspawn/NAME.nspawn` from the machine's
record, sets up the [network](/docs/networking/) it is configured for, starts the
unit through systemd and waits up to 30 seconds for the machine to register
with machined. For a booted machine it then waits, up to 20 more seconds, until
the machine's own systemd is listening, so that a command can follow right
away; `--no-wait` skips that last wait. A program that runs and returns at once
is reported as such, not as a failure.

Everything given to `start` is remembered for the machine, so a plain
`nspawn start NAME` next time reuses the network, the ports, the command, the
variables and the volumes of the last run:

- `--network` switches the machine between the bridge, a veth pair and the
  host's network; see [Networking](/docs/networking/).
- `-p HOST:CONTAINER[/udp]` publishes a port on the host, like docker. It needs
  the bridge network; `-p none` forgets all published ports.
- `--entrypoint`, `-e` and the arguments after `--` change what an **app**
  image runs; see [Command, entrypoint and environment](#command-entrypoint-and-environment).
  Boot images refuse them.
- `-v` mounts a host directory or a named volume into any kind of machine; see
  [Volumes](#volumes).

Machines are started by name. A reference (`fedora:44`, `docker.io/x`) is
refused with a hint to `pull` it or to `create` a machine from a local image.
Images that were not installed by nspawn, for example something created with
`machinectl import-tar`, can be started too: they get the stock template's veth
networking, and nspawn makes sure systemd-networkd runs on the host so that the
machine actually gets an address, but none of the flags above apply to them.

## Boot machines

A boot image is started with `Boot=yes`: systemd-nspawn runs the image's init
as PID 1, like `machinectl start` does, and the machine joins the bridge with
an address that its own systemd-networkd configures.

- `exec` enters the machine's namespaces directly, like `docker exec`, so
  nothing is needed inside: no D-Bus, no PAM. The exit code comes back and the
  program is found on the machine's `PATH`.
- `shell` opens machined's login session (`OpenMachineShell`), which needs
  D-Bus inside the machine; the hub images have it. Right after `start` the
  machine's D-Bus may not be up yet, so `shell` retries for up to 20 seconds
  instead of failing.
- `stop` asks machined to power the machine off and repeats the request every
  two seconds until the machine is gone, for up to a minute; the repetition
  covers the window right after boot in which the init has not installed its
  signal handlers yet. `--force` kills every process at once.
- With volumes, a small unit mounted into the machine, `nspawn-volumes.service`,
  holds `local-fs.target` until all of them are mounted, so services find their
  data in place whatever the backend.

## App machines

An app image is started with `Boot=no` and `ProcessTwo=yes`: nspawn's stub init
is PID 1 and the image's entrypoint runs as PID 2 with the environment, working
directory, user and stop signal from the OCI config. Its network namespace is
built on the host before the program starts, so the network is there from the
first instruction, as in docker. For the same reason an app on the bridge runs
without a user namespace (`PrivateUsers=no`), which is also docker's default;
capabilities, seccomp and the other namespaces still apply.

- `exec` and `shell` enter the namespaces of the machine's leader process, on a
  pseudo terminal, with the image's environment. `shell` runs `/bin/sh`, and
  `-u USER` switches user for both.
- `stop` sends the image's stop signal (`StopSignal` in the config, `SIGTERM`
  by default) to the program itself, waits `--timeout` seconds (10 by default)
  and kills the machine if it is still there, like `docker stop`. A stop
  through systemd (`systemctl stop`, shutdown) reaches the stub init as a
  poweroff request, which it answers with `SIGTERM` to the program.
- A program that exits on its own ends the machine; its network and ports are
  released by the unit hooks, and `nspawn stop` on it afterwards only clears
  what it left behind and says `NAME was not running`.

## Command, entrypoint and environment

What an app runs is decided exactly as with docker:

- The arguments after `--` replace the image's `cmd` and follow its
  `entrypoint`. `nspawn start web -- nginx -T` still runs
  `/docker-entrypoint.sh` first, as `docker run nginx nginx -T` would.
- `--entrypoint PROGRAM` replaces the entrypoint with one program; the
  arguments after `--` follow it. `--entrypoint ""` drops the entrypoint, so
  the arguments run alone.
- Both are remembered, like the command of a docker container.
  `--image-command` forgets them and runs the image's own entrypoint and cmd
  again.
- `-e VAR=value` adds a variable on top of the image's, and `-e VAR` copies it
  from the shell that runs nspawn. The later value of a variable wins, `-e none`
  forgets them all, and `exec` sees the same environment as the program. Names
  follow the usual rules (letters, digits and `_`, not starting with a digit).
- The working directory, the user and the stop signal come from the image. Of
  docker's `uid:gid` form of the user, the uid part is used; the gid comes from
  the image's `passwd`.

`ps` shows the effective command; `nspawn start NAME -- true` is a quick way to
check that an image runs at all.

## Volumes

```shell
sudo nspawn start web -v /srv/www:/usr/share/nginx/html:ro -v pgdata:/var/lib/postgresql
```

`-v SOURCE:TARGET[:ro]` is docker's syntax:

- An absolute `SOURCE` is a host directory (or file) that has to exist already,
  as with podman: `start` refuses a path that is not there rather than making
  one. A `SOURCE` without a leading `/` is a **named volume** that nspawn
  keeps under `/var/lib/nspawn/volumes/NAME`, created on first use and never
  deleted by `images rm`. Names may contain letters, digits, `_`, `.` and `-`.
- `TARGET` is an absolute path inside the machine, other than `/`. The same
  target cannot be mounted twice.
- `:ro` mounts it read-only; `:rw` is the default. Paths with whitespace are
  not supported.
- `-v` is repeatable and remembered; `-v none` forgets every volume.

In machines that run with private users (booted machines on overlay and flat,
which is the default), the mount is idmapped, so root inside owns what it
writes on the host, as docker users expect. On `mstack` machines nspawn attaches
the volumes from the host right after the machine's init starts, since
systemd-nspawn cannot idmap binds under managed user namespaces; that is what
`nspawn-volumes.service` waits for, for up to two minutes, failing visibly
otherwise. On overlay and flat the volumes come from the settings file and are
there before the init even runs.

## ps

```shell
sudo nspawn ps [-a]      # same as: nspawn machines ls [-a]
```

```text
 MACHINE    IMAGE                     MODE  COMMAND                             STATE    UP  PID    NETWORK                  OS
 fedora-44  hub.nspawn.org/fedora:44  boot  init                                running  2h  48213  10.99.0.2                fedora
 web        docker.io/library/nginx   app   /docker-entrypoint.sh nginx -g ...  running  5m  51002  10.99.0.3 8080->80/tcp   debian
 db         hub.nspawn.org/fedora:44  boot  init                                stopped  -   -      10.99.0.4                -
```

`ps` lists every machine machined knows about; machines that nspawn did not
install show `-` in the image columns. `-a` adds the nspawn machines that are
not running. `COMMAND` is the effective entrypoint and arguments of an app,
and `NETWORK` the bridge address with the published ports, or `host` or
`veth`.

## exec and shell

```shell
sudo nspawn exec MACHINE [-u USER] COMMAND...
sudo nspawn shell MACHINE [-u USER]
```

`exec` runs one command inside a running machine of either kind, attached to
your terminal, and exits with the command's status, so it works in scripts and
pipelines (what goes through stdin and stdout is byte exact). The program is
looked up on the machine's `PATH`, the image's environment and the `-e`
variables apply, and the working directory is the image's. Neither D-Bus nor
anything else is needed inside.

`shell` opens an interactive shell as `root` (or `-u USER`): machined's login
session for booted machines, `/bin/sh` in the machine's namespaces for apps.

## logs

```shell
sudo nspawn logs MACHINE [-f] [-n N] [--since WHEN] [-t] [--all] [--inside]
```

systemd-nspawn sends what the machine writes to its console to the journal of
`systemd-nspawn@MACHINE.service`, and `logs` reads it with `journalctl`. By
default only the machine's own output is shown, from every run of the unit,
earlier ones included:

- `-f` keeps printing new output, starting from the last 10 lines unless `-n`
  says otherwise. `-n N` shows the last N lines; `--since "10 min ago"`
  accepts anything `journalctl --since` does; `-t` prefixes each line with its
  timestamp.
- `--all` also shows what systemd logged about the unit: start, stop,
  failures.
- `--inside` reads the journal of a **booted** machine itself
  (`journalctl --machine`), which is where the services running inside log.

`logs` works for stopped machines too, since the journal keeps what they wrote.

## stop

```shell
sudo nspawn stop NAME [-f] [-t SECONDS] [--no-wait]
```

Stops the machine as described above for boot and app machines, waits until it
is gone (a minute at most for a booted machine), stops the unit so that the
image can be removed right away, clears the failure a signal-killed program
leaves on the unit, releases the firewall exceptions of a veth machine and
removes the machine's published ports and network namespace. `--no-wait`
returns right after the request, without the kill after `--timeout`; the unit
hooks release the network when the machine ends. `-f` kills every process at
once, like `docker kill`. Stopping a machine that already ended is not an
error.
