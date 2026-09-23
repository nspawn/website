---
title: Networking
weight: 5
description: >-
  The nspawn0 bridge, published ports, DNS and machine names, how apps get
  their network, firewalls, and the veth and host alternatives.
---

Every machine records which network it uses. `start --network` changes it, and
the choice sticks for the next start.

| Network | Default for | What the machine gets |
| --- | --- | --- |
| `bridge` | Every image nspawn installs | A fixed address on the `nspawn0` bridge, NAT to the outside, published ports, the names of the other machines and of the host. Needs nothing from the host's own network manager. |
| `host` | | The host's network namespace, like `docker run --network host`: the machine sees the host's interfaces and binds to the host's ports. Works for both kinds of image. |
| `veth` | Images not installed by nspawn | The classic systemd-nspawn setup: a virtual ethernet pair whose host end is configured by systemd-networkd through the stock `80-container-ve.network`. Booted images only. |

## The bridge

On the first `start` of a bridged machine (or with `nspawn network up`, which
is handy at boot) nspawn creates the bridge with the first address of the
subnet, enables IPv4 forwarding and installs the nftables table `ip nspawn`
with masquerading for the subnet. The defaults are the bridge `nspawn0` and the
subnet `10.99.0.0/24`; both, and the DNS servers, can be changed in
[the configuration file](/docs/configuration/). An interface that already has
that name is only taken over when it is a bridge nspawn made, or an empty one;
`bridge = "docker0"` is refused rather than acted on. Nothing else on the host
is touched, so it works the same with systemd-networkd, NetworkManager or no
network manager at all.

Each machine gets a fixed address from the subnet, remembered with its record.
A **booted** machine receives it through a `.network` file that nspawn
generates and bind-mounts at `/run/systemd/network/10-host0.network`, for the
systemd-networkd inside to apply, together with the DNS servers: the host's
upstream resolvers by default, `dns` from the configuration if set, and public
resolvers as a last resort, with a warning, when none can be determined.

An **app** machine has nothing inside to configure an interface, so nspawn
builds its network namespace before the program starts: `ip netns`, a veth pair
on the bridge, the address and the default route, handed to systemd-nspawn with
`NamespacePath=`, plus a generated `/etc/resolv.conf`. The namespace lives at
`/run/netns/nspawn-NAME` while the machine runs and goes away with it.

A generated `/etc/hosts`, mounted into every bridged machine, resolves the names
of the other machines on the bridge and `host.nspawn.internal` for the host.
On hosts with systemd 258 or newer, machined also lets the host resolve machine
names by itself. The bridge carries IPv4 only, so neither the machines nor the
bridge get an IPv6 link-local address, and a machine's name leads to its bridge
address: `ping web` from the host answers from `10.99.0.x`.

```shell
sudo nspawn network ls
```

```text
nspawn0 10.99.0.0/24 (gateway 10.99.0.1, host name host.nspawn.internal)
 MACHINE    ADDRESS    PORTS          STATE
 fedora-44  10.99.0.2  -              running
 web        10.99.0.3  8080->80/tcp   running
 db         10.99.0.4  -              stopped
```

## Published ports

```shell
sudo nspawn start web -p 8080:80 -p 5353:53/udp
```

Each `-p HOST:CONTAINER[/udp]` becomes a DNAT entry in the `ip nspawn` table.
The port is reachable from other hosts, from the host's own addresses and from
`127.0.0.1` (through `route_localnet`, as docker does without its userland
proxy); binding to a single host address is not supported. The entries are
installed once the machine is registered and removed when it ends, by the unit
hooks, so they also go away after a crash or when the program exits on its
own. A port another running machine publishes, or one a service of the host
already listens on, is refused before the machine starts, and a refused port
is not remembered.

The list is remembered for the machine: `nspawn start web` next time publishes
the same ports, and `-p none` forgets them all. Ports need the bridge network;
a machine on the host's network listens on the host's ports directly.

## Firewalls

- **firewalld**: the bridge is placed in the `trusted` zone at runtime, which
  also lets published ports through. The binding does not survive
  `firewall-cmd --reload`; the next `start` or `nspawn network up` puts it
  back.
- **docker** (in its default iptables mode) and **ufw** set the `FORWARD`
  policy to `DROP`, which would silence every machine on the bridge. `start`
  then adds two rules to the `DOCKER-USER` chain, which docker reserves for
  that, or to the top of `FORWARD` itself: anything out of the bridge, and into
  the bridge only what was published or belongs to a connection a machine
  opened. That needs the `iptables` command, which those tools bring with them.
- Where something else drops forwarded traffic, a hand-written nftables
  firewall or those same rules without the `iptables` command to edit them,
  `start` says so and the exception has to be made by hand. Without it the
  machines reach nothing beyond the bridge and published ports answer on this
  host alone.

## veth

`--network veth` keeps the classic systemd-nspawn behaviour for booted
machines: systemd-nspawn creates a virtual ethernet pair and systemd-networkd on
the host brings up the host end (`ve-NAME`), gives it an address, serves DHCP
to the machine and masquerades its traffic, all through the stock
`80-container-ve.network`. App images refuse it, since nothing inside would
configure the pair.

Because that needs systemd-networkd, `start` activates it when the host has no
`.network` files of its own in `/etc/systemd/network` or `/run/systemd/network`,
and refuses with an explanation when it has, so that nspawn never takes over
interfaces another network manager is handling. A masked systemd-networkd is an
error. With firewalld, `ve-NAME` is bound to the `trusted` zone while the
machine runs; otherwise the default zone would drop the machine's DHCP
requests.

## host

`--network host` sets `VirtualEthernet=no`: the machine shares the host's
network namespace, sees the host's interfaces and binds to the host's ports.
Published ports do not apply, and an app that runs this way keeps its user
namespace.
