# Contributing

Fixes and new pages are welcome as pull requests to
[nspawn/website](https://github.com/nspawn/website). Every documentation page
has an "Edit this page" link that opens the right file on GitHub; for anything
bigger, clone the repository and preview it locally as described in
[README.md](README.md).

Please keep the tone of the existing pages: direct, in plain English, with a
command or an example where one helps. Document what nspawn does today; ideas
for the tool itself belong in [nspawn/nspawn](https://github.com/nspawn/nspawn).

The [code of conduct](https://github.com/nspawn/coc) of the nspawn
organization applies here.

## Maintainer notes

### Dependencies

The lockfile (`package-lock.json`) and the generated theme manifest
(`packages/hugoautogen/package.json`) are committed so that installs are
reproducible: `npm run install:safe` (`npm ci --ignore-scripts`) installs
exactly what they pin. Dependabot proposes updates weekly, except for
`hugo-extended`, Bootstrap and Font Awesome, which are updated as described
below.

### Upgrade Docsy

To the latest tagged release:

```shell
npm run update:docsy:mod
```

Or to the latest commit on Docsy's main branch:

```shell
npm run update:docsy:main
```

Both update `go.mod`/`go.sum`, regenerate `packages/hugoautogen/package.json`
and `package-lock.json` (this needs `go`, since it runs `hugo mod`), and
install the result. Commit whatever changed, and read the release notes: Docsy
raises its minimum Hugo version from time to time, which is what
`module.hugoVersion.min` in `hugo.yaml` and the `hugo-extended` pin have to
follow.

### Update Hugo

```shell
npm run update:hugo     # bumps the hugo-extended pin
npm run approve:hugo    # approves its install script for the new version
```

Script-enabled installs fail until the new version is approved, so run both
before building again.

### Develop against a local Docsy

With a Docsy clone in the sibling folder declared in `docsy.work`, prefix the
npm script with `local`:

```shell
npm run local -- serve
```

The server then watches the local theme, so theme edits reload live.
