---
title: Building images
linkTitle: Building
weight: 6
description: >-
  Building an image with mkosi through nspawn build, and uploading it to the hub
  or another registry with push.
---

## build

```shell
sudo nspawn build [DIRECTORY] -t REFERENCE [--name NAME]
                  [-d DISTRIBUTION] [-r RELEASE] [--profile PROFILE]...
                  [--backend BACKEND] [--mode MODE] [--force] [--keep-output]
                  [-- MKOSI_ARGUMENTS...]
```

`build` runs [mkosi](https://github.com/systemd/mkosi) on a directory that has
a `mkosi.conf` or a `mkosi.conf.d` (the current directory by default), with
`--format=oci`, and imports the result exactly as `pull` would import an image
from a registry. It needs root and mkosi in `PATH`. The same configuration
tree that works with mkosi on its own works here; `--distribution`,
`--release`, `--profile` and anything after `--` are passed through.

The command line that mkosi receives is:

```text
mkosi --directory=DIRECTORY --format=oci --compress-output=zstd
      --output-directory=/var/lib/nspawn/builds/NAME-TIMESTAMP
      --cache-directory=/var/lib/nspawn/cache/mkosi
      --image-id=REPOSITORY [--distribution=...] [--release=...] [--profile=...]...
      [MKOSI_ARGUMENTS...] --force build
```

The `/` characters of the repository are replaced by `-` in the image id. The
package cache is kept between builds; the output directory is deleted after the
import unless `--keep-output` is given.

When mkosi is done, nspawn finds the OCI layout it produced, adds a few
annotations to the manifest (`org.opencontainers.image.version` with the tag,
`org.opencontainers.image.ref.name` with the full reference, and
`org.nspawn.builder`), verifies every blob against its digest, copies the blobs
into the store and installs the image with the chosen backend, with `build` as
its origin. The image can be started right away, cloned with `create` or
pushed.

A minimal configuration for a bootable Fedora machine, taken from the test
suite of nspawn:

```ini
[Distribution]
Distribution=fedora
Release=44

[Output]
ImageId=e2e-built

[Content]
Bootable=no
SELinuxRelabel=no
RootPassword=root
Packages=
        systemd
        systemd-networkd
        systemd-resolved
        dbus-broker
        passwd
        util-linux
```

`systemd-networkd` is what configures `host0` on the bridge, and `dbus-broker`
(or `dbus`) is what lets `shell` reach the machine's systemd. Then:

```shell
sudo nspawn build -t team/app:1 ./app
sudo nspawn start team-app-1
```

## push

```shell
nspawn push IMAGE [--to REFERENCE]
```

`IMAGE` is a local image name or the reference it was pulled from or built as.
By default the image is pushed under its own reference; `--to` pushes it under
another one, for example to retag it or to send it to a different registry:

```shell
nspawn push team/app:1
nspawn push team-app-1 --to registry.example/team/app:2
```

`push` authenticates first, with the credentials stored for that registry, and
tells you which `nspawn login` to run when the registry wants some it does not
have. The blobs that are already on the registry are skipped and the manifest
is uploaded last. A push needs a tag, not a digest, and the image's manifest
and blobs must still be in the store (`images rm` removes them). See
[Registries and credentials](../images/#registries-and-credentials).

## The hub images

The images the team publishes on `hub.nspawn.org` are built the same way, from
the definitions in
[nspawn/mkosi-definitions](https://github.com/nspawn/mkosi-definitions). Use
them as a starting point for your own images, or open a pull request there to
change what the hub ships.
