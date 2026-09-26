---
title: About nspawn.org
linkTitle: About
description: The project, the people behind it, and what this site does with your data.
menu: { main: { weight: 10 } }
---

{{% blocks/cover title="About nspawn.org" height="auto td-below-navbar" color="primary" %}}

<!-- prettier-ignore -->
{{% _param description %}}
{.display-6}

{{% /blocks/cover %}}

{{% blocks/section color="white" %}}

<div class="td-content" style="max-width: 50rem; margin: 0 auto;">

## The project

nspawn.org started in 2019 as a hub of systemd-nspawn images: distributions
built with mkosi, signed, and installed with `machinectl pull-tar` or through a
small wrapper script. Today it is an OCI registry, `hub.nspawn.org`, and
`nspawn`, a tool that manages systemd-nspawn machines with a docker-like
command line. Everything is open: the
[tool](https://github.com/nspawn/nspawn), the
[image definitions](https://github.com/nspawn/mkosi-definitions) and
[this site](https://github.com/nspawn/nspawn.org).

## The team

| | | |
| --- | --- | --- |
| **Christian Rebischke** | Germany | chris@shibumi.dev |
| | PGP | `6DAF 7B80 8F9D F251 3962 0000 D214 61E3 DFE2 060D` |
| **Eduard Tolosa** | Colombia | edu4rdshl@protonmail.com |
| | PGP | `8D19 E962 4180 8487 38B9 4833 3A57 4A40 09F5 53E5` |

## Contact

Join `#nspawn-org` on [Matrix](https://matrix.to/#/#nspawn-org:matrix.org) or
on [Libera.Chat](https://web.libera.chat/#nspawn-org), write to the team at
team@nspawn.org, or open an issue in the repository the matter belongs to; the
[community](/community/) page lists them.

## Privacy {#privacy}

This site does not collect any user data. No IP addresses are stored, no user
agent or timestamp is logged, no cookies are used to identify visitors, and the
site is served over HTTPS only. The search box works locally in your browser
and sends nothing anywhere. Therefore no data is collected and none can be
passed on to third parties.

</div>

{{% /blocks/section %}}
