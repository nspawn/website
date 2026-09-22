---
title: Documentation
linkTitle: Docs
description: >-
  How to install nspawn, pull images from the hub, run machines and apps, wire
  their network, build your own images and configure the tool.
menu: { main: { weight: 20 } }
---

nspawn manages [systemd-nspawn](https://www.freedesktop.org/software/systemd/man/latest/systemd-nspawn.html)
machines the way docker manages containers: images come from an OCI registry
(the hub), are stored as shared layers and are started, inspected and stopped
through the D-Bus APIs of systemd-machined and systemd. This documentation
describes nspawn {{% param version %}}.

If you are new, start with the [overview](/docs/overview/) and then
[get started](/docs/getting-started/). The [command reference](/docs/reference/) lists
every command and option.
