---
title: Machines
weight: 4
description: >-
  Starting and stopping machines, run like docker run, entrypoints,
  environment, volumes and labels, restart policies and resource limits,
  running commands inside, copying files, output, usage and events, removing
  machines, and how boot and app images differ.
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
nspawn belongs in `/usr/local/bin` or `/usr/bin`. It also carries the machine's
restart policy and resource limits, when it has them, and for an app machine
the `ExecStart=` that runs systemd-nspawn through `nspawn attach-exec`, which
is how `run -i` and `run -t` hand the program an input or a terminal.

## start

```shell
sudo nspawn start NAME [--network NETWORK] [-p HOST:CONTAINER[/udp]]...
                  [--entrypoint PROGRAM] [-e VAR[=VALUE]]... [-v SOURCE:TARGET[:ro]]...
                  [-l KEY=VALUE]... [--restart POLICY] [-m SIZE] [--cpus N] [--pids-limit N]
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

- `--network` switches the machine between the bridge, a network of its own
  made with `network create`, a veth pair and the host's network; see
  [Networking](/docs/networking/).
- `-p HOST:CONTAINER[/udp]` publishes a port on the host, like docker. It needs
  the bridge network; `-p none` forgets all published ports.
- `--entrypoint`, `-e` and the arguments after `--` change what an **app**
  image runs; see [Command, entrypoint and environment](#command-entrypoint-and-environment).
  Boot images refuse them.
- `-v` mounts a host directory or a named volume into any kind of machine; see
  [Volumes](#volumes).
- `-l`/`--label`, `--restart`, `-m`, `--cpus` and `--pids-limit` label the
  machine, give it a restart policy and bound its resources; see
  [Labels](#labels) and [Restart policies and limits](#restart-policies-and-limits).

Machines are started by name. A reference (`fedora:44`, `docker.io/x`) is
refused with a hint to `pull` it or to `create` a machine from a local image.
Images that were not installed by nspawn, for example something created with
`machinectl import-tar`, can be started too: they get the stock template's veth
networking, and nspawn makes sure systemd-networkd runs on the host so that the
machine actually gets an address, but none of the flags above apply to them.

## run

```shell
sudo nspawn run [-d] [--rm] [-i] [-t] [OPTIONS] REFERENCE [COMMAND [ARGUMENT...]]
```

`run` makes a machine from an image and starts it, like `docker run`: `pull`,
or `create` from a local image with the same reference, then `start`, with the
options of both. What follows the image replaces an app's command, as with
docker. Without `-d` it stays with the machine:

- The machine's output follows until it ends, stdout and stderr together, a
  line at a time. It is read from the journal, so `nspawn logs NAME` shows it
  later, and the machine goes on should `run` be interrupted.
- `run` exits with the program's exit code, or 128 plus the signal it died of:
  130 after Ctrl-C, 137 after `kill`. Ctrl-C, SIGTERM, SIGHUP and SIGQUIT go to
  the program; a third Ctrl-C within a second leaves the machine running and
  returns.
- `-i` gives the program this standard input, `-t` a terminal; closing that
  terminal stops the machine, since nothing would read it any more.
- `--rm` removes the machine once it ends, and leaves nothing behind when the
  start fails. Named volumes stay, and so does an image the run had to pull,
  under the image's local name, as docker keeps images; without `--name` the
  machine gets a name of its own.

```shell
sudo nspawn run --rm docker.io/library/busybox:latest sh -c 'echo hi; exit 3'; echo $?
echo abc | sudo nspawn run -i --rm docker.io/library/busybox:latest wc -c
sudo nspawn run -it --rm docker.io/library/alpine:3 sh
sudo nspawn run -d --name web -p 8080:80 docker.io/library/nginx:latest
```

A booted image shows its console until it powers off (Ctrl-C powers it off).
`run -it` on one waits for its boot, opens a root shell and powers the machine
off when the shell ends, with the shell's exit code: `sudo nspawn run -it --rm
fedora:44` is a throwaway Fedora with its own systemd.

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
  deleted by `rm` or `images rm`. Names may contain letters, digits, `_`, `.`
  and `-`, and may not start with a dot.
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

Named volumes outlive the machines that use them. `nspawn volume ls` lists them
with the machines whose records mount them, `volume create NAME` makes one
ahead of its first use, and `volume rm` and `volume prune` remove the ones no
machine uses; a volume a machine still names is refused until that machine is
started with other volumes (or `-v none`) or removed.

## Labels

```shell
sudo nspawn start web --label caddy=web.example --label tier=front
```

Labels work as in docker. The image's own labels (`LABEL` in a Containerfile,
`OciLabels=` in a `mkosi.conf`) are read from its configuration when it is
pulled or built, and `-l`/`--label KEY=VALUE` on `start` or `create` adds the
machine's own on top, remembered like the ports; `--label none` forgets them.
nspawn does nothing with them itself: `nspawn inspect` and `ps --json` show the
merged `labels` and the image's `image_labels`, for tools that configure
themselves from them, a reverse proxy say.

## Restart policies and limits

```shell
sudo nspawn start web --restart unless-stopped -m 512m --cpus 1 --pids-limit 500
```

`--restart` takes docker's policies:

- `no`, the default: a machine that ends stays down.
- `on-failure`: restarted when its program, or its init, dies with an error.
- `always`: restarted whenever it ends, and started at boot.
- `unless-stopped`: like `always`, until `nspawn stop`, which also takes it
  off the boot list until the next `nspawn start`.

A machine that ends is started again after a second, then later and later, up
to half a minute, for as long as it keeps failing; `ps` shows it as
`restarting` meanwhile, and `nspawn stop` ends that. The ports, the address and
the volumes come back with it, since the unit's own hooks prepare every run.
`always` and `unless-stopped` enable the unit the way `machinectl enable` does,
and removing the machine takes that back. `machinectl stop` stops a machine for
good too, but does not take an `unless-stopped` one off the boot list, and a
`poweroff` from inside counts as ending under `always`.

`-m`/`--memory` (`512m`, `2g`), `--cpus` (`0.5`, `2`) and `--pids-limit` bound
the whole machine: they are the MemoryMax=, CPUQuota= and TasksMax= of its
unit, which is why nothing inside shows them. As with docker, `--memory` also
lets the machine use as much swap again (MemorySwapMax=), and no more. `0`
removes a limit.

Both are remembered like the ports and apply at the next start, and
`nspawn update` changes them without one, like `docker update`: a running
machine gets the new limits in its cgroup at once.

```shell
sudo nspawn update web -m 1g --cpus 2 --restart always
```

With a policy,
`stop --no-wait` of an app also lets systemd-nspawn's stub init send the
program SIGTERM and SIGHUP, since nobody stays behind to stop the unit later.

## ps

```shell
sudo nspawn ps [-a] [--json]      # same as: nspawn machines ls [-a] [--json]
```

```text
 MACHINE    IMAGE                     MODE  COMMAND                             STATE    UP  PID    NETWORK                  OS
 fedora-44  hub.nspawn.org/fedora:44  boot  init                                running  2h  48213  10.99.0.2                Fedora Linux 44 (Forty Four)
 web        docker.io/library/nginx   app   /docker-entrypoint.sh nginx -g ...  running  5m  51002  10.99.0.3 8080->80/tcp   Debian GNU/Linux 12 (bookworm)
 db         hub.nspawn.org/fedora:44  boot  init                                stopped  -   -      10.99.0.4                -
```

`ps` lists every container machined knows about (the virtual machines it also
registers, libvirt's among them, are left out); machines that nspawn did not
install show `-` in the image columns. `-a` adds the nspawn machines that are
not running. `COMMAND` is the effective entrypoint and arguments of an app,
and `NETWORK` the bridge address with the published ports, or `host` or
`veth`. A machine between two runs of its restart policy is listed as
`restarting` even without `-a`.

`ps --json` prints the same machines as the service returns them, each with its
whole record, and `nspawn inspect NAME...` prints one or more machines, running
or not, as a JSON array, like `docker inspect`: what a script or an agent
reads.

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

## cp

```shell
sudo nspawn cp ./nginx.conf web:/etc/nginx/
sudo nspawn cp web:/var/log/nginx ./nginx-logs
sudo nspawn cp ./site/. web:/usr/share/nginx/html
```

`cp` copies files and directories between the host and a machine, running or
not, with docker cp's rules:

- An existing directory as the destination receives the source under its own
  name; anything else is the name of the copy. A destination ending in `/`
  has to be a directory.
- `DIR/.` copies the contents of DIR rather than DIR.
- What goes in belongs to root inside the machine, whatever user namespace it
  runs in; what comes out belongs to whoever ran `cp`. Modes and modification
  times are kept.
- Paths inside the machine are resolved inside it, so a link there, absolute
  or not, never leads to the host. Links are copied as links; devices, sockets
  and fifos are left out.

A stopped overlay or flat machine can be copied into and out of; a stopped
`mstack` machine cannot, since its tree only exists while it runs. Where
SELinux enforces, a host directory mounted with `-v` keeps its own label, which
the service may not be allowed to write to (docker needs `:z` for the same);
named volumes are nspawn's own and always work.

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

## stats

```shell
sudo nspawn stats [NAME...] [--no-stream] [--json]
```

`stats` shows what each running machine uses, like `docker stats`, drawn again
every second: CPU (100% is one CPU busy), memory in use against the limit (the
host's memory without one), network and disk traffic, and processes. The
numbers are those of the cgroup of the machine's unit, which holds the whole
machine, and of its interfaces as the machine sees them. `--no-stream` prints
one table, `--json` one object per machine and reading.

```text
 NAME  CPU %  MEM USAGE / LIMIT  MEM %  NET I/O      BLOCK I/O      PIDS
 api   0.00%  2.4 MiB / 3.8 GiB  0.06%  372 B / 0 B  0 B / 4.0 KiB  3
 db    0.00%  2.4 MiB / 3.8 GiB  0.06%  892 B / 0 B  0 B / 4.0 KiB  3
 web   0.00%  2.4 MiB / 3.8 GiB  0.06%  522 B / 0 B  0 B / 4.0 KiB  3
```

## events

```shell
sudo nspawn events [--since WHEN] [--until WHEN] [-f KEY=VALUE]... [--json]
```

`events` reports what happens, like `docker events`: machines that start, die
(with their exit code), stop, are restarted, run out of memory or fail, and
what nspawn does: pulls, builds, creations, pushes, kills, updates and
removals, of networks and volumes too. It reads the journal, where systemd
logs every start and end of a machine's unit however it was started, and
nspawn logs what it does, so `--since` reads past events back. Filters take
`name=`, `type=` (`machine`, `network`, `volume`), `event=` and `label=KEY` or
`label=KEY=VALUE`: the same key given twice matches either value, different
keys must all match. `--json` prints one object per event.

```text
2026-09-24T20:44:51.767425Z network create backend (subnet=10.99.1.0/24)
2026-09-24T20:44:53.691583Z machine start db (image=docker.io/library/busybox:latest)
2026-09-24T20:44:56.730139Z machine create busybox (from=api, reference=docker.io/library/busybox:latest)
2026-09-24T20:44:57.105739Z machine start busybox
2026-09-24T20:44:57.125181Z machine die busybox (code=exited, exit_code=3)
2026-09-24T20:44:57.154522Z machine fail busybox (result=exit-code)
2026-09-24T20:44:57.381848Z machine remove busybox
2026-09-24T20:44:57.645267Z machine kill api (image=docker.io/library/busybox:latest, signal=1)
```

Times are in UTC. A machine's `die` carries the exit code systemd-nspawn gave:
255 for an app whose program died of a signal, as `stop` makes it.

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
With a restart policy the machine stays stopped for now: an `unless-stopped`
machine is also taken off the boot list until the next `start`, while an
`always` one still starts at the next boot.

## kill

```shell
sudo nspawn kill web                 # SIGKILL: stopped for good
sudo nspawn kill -s HUP web          # a signal for the program
```

`kill` sends a signal to machines, like `docker kill`. SIGKILL, the default,
stops the machine for good, as `stop --force` does. Any other signal goes to
the program of an app, or to the init of a booted machine, and the machine
lives on unless it ends of it; then its restart policy applies, except when the
signal was the machine's own stop signal, which counts as a stop, as with
docker.

## rm

```shell
sudo nspawn rm web2
sudo nspawn rm -f web        # stop it first, like docker rm -f
```

`rm` removes a machine: its record, its tree, its unit files, the boot link a
restart policy made, and the layers nobody else uses. A pulled image is a
machine too, so `rm` and `images rm` remove the same thing; `rm -f` stops a
running machine first where both refuse otherwise. Named volumes are kept, and
`rm` says which.
