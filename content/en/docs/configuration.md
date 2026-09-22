---
title: Configuration
weight: 7
description: >-
  The configuration file, environment variables and global flags, what wins
  when they disagree, and what is remembered per machine.
---

nspawn needs no configuration to work: the defaults point at the hub, use
`/var/lib/machines` and `/var/lib/nspawn`, pick the best backend for the host
and run the bridge on `10.99.0.0/24`. Everything below is optional.

## Precedence

1. Command line flags and their environment variables (`--registry` or
   `NSPAWN_REGISTRY`, `--ca-cert` or `NSPAWN_CA_CERT`).
2. The configuration file.
3. The built-in defaults.

## The configuration file

The file is `/etc/nspawn/nspawn.toml`, read when it exists; `--config FILE` or
`NSPAWN_CONFIG` names another one, which then must exist. Every key is
optional, and unknown keys are an error so that a typo does not silently fall
back to a default.

| Key | Type | Default | Meaning |
| --- | --- | --- | --- |
| `registry` | string | `hub.nspawn.org` | Registry for references without a host part. |
| `ca_cert` | path | none | Extra CA certificate (PEM) to trust when talking to the hub. |
| `backend` | `auto`, `overlay`, `flat` or `mstack` | `auto` | Backend for `pull` and `build` when they are not given `--backend`. |
| `machines_dir` | absolute path | `/var/lib/machines` | Where machines are assembled. |
| `state_dir` | absolute path | `/var/lib/nspawn` | Blobs, layers, records, volumes and everything else nspawn keeps. |
| `bridge` | string, 1 to 15 letters, digits, `-` or `_` | `nspawn0` | Name of the bridge the machines join. It must be free, or a bridge nspawn made. |
| `subnet` | IPv4 CIDR, prefix 8 to 30 | `10.99.0.0/24` | Subnet of the bridge; its first address is the bridge's own. |
| `dns` | list of IPv4 addresses | the host's upstream servers | DNS servers handed to bridged machines. They must be reachable from the bridge: no loopback, no IPv6. |

An example:

```toml
# /etc/nspawn/nspawn.toml
registry = "registry.example:5000"
ca_cert = "/etc/pki/tls/certs/example-ca.pem"
backend = "overlay"

bridge = "br-lab"
subnet = "172.30.5.0/24"
dns = ["172.30.5.1", "9.9.9.9"]
```

Changing `bridge` or `subnet` affects machines started afterwards; the address
recorded for a machine is reassigned from the new subnet on its next start.
When a machine is installed or started with `--config`, the unit hooks it gets
carry the same `--config`, so the machine keeps using that file whoever starts
it.

## Environment variables

| Variable | Same as |
| --- | --- |
| `NSPAWN_REGISTRY` | `--registry` |
| `NSPAWN_CA_CERT` | `--ca-cert` |
| `NSPAWN_CONFIG` | `--config` |

They are convenient for scripts and for the end-to-end tests, which run against
a private registry with a private CA:

```shell
NSPAWN_REGISTRY=hub.nspawn.test:8443 NSPAWN_CA_CERT=/etc/zot/ca.crt nspawn hub ls
```

## Credentials

`nspawn login` keeps registry credentials in `/etc/nspawn/auth.json`, and
credentials left by `docker login` or `podman login` are picked up from their
usual files. See [Registries and credentials](/docs/images/#registries-and-credentials).

## Per-machine choices

Some settings belong to a machine rather than to the host, and are given when
it is created or started; every one of them is remembered until it is changed:

- `--backend` and `--mode` on `pull` and `build`; `--backend` on `create`.
- `--name` on `pull` and `build`, to choose the local name.
- `--network`, `-p`, `-e`, `-v`, `--entrypoint` and the arguments after `--`
  on `start` and `create`. `-p none`, `-e none`, `-v none` and
  `--image-command` forget what was remembered.
