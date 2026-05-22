# InkBridge Packaging

> This repository is public only because GitHub requires reusable
> workflows called from public callers to live in a public repository.
> The contents are tightly tied to InkBridge's self-hosted runners and
> internal infrastructure, and are not expected to be useful to anyone
> outside the organisation.

GitHub workflow actions used by other InkBridge Networks repositories
to package Python or Lua modules and publish them to the InkBridge
package repositories, and to mirror GitHub repositories back to the
self-hosted git server.

All consumer-side setup (publish target URLs, pip / luarocks client
config, current org admins to contact for secret access) lives on
the infrastructure wiki:
<https://infra.wiki.internal.networkradius.com/services/package-repos>.

## Secrets

| Language | Key Secret       |
| -------- | ---------------- |
| Lua      | LUAROCKS_PUBLISH |
| Python   | PYPI_PUBLISH     |
| Mirror   | MIRROR_PUBLISH   |

All publishing scripts need access to the SSH private key for the
matching git-backed index. Access is granted per repository - see
the wiki for the request procedure.

## Environment

`VERSION` is exported to user-provided commands. Its value is the
current ref with a leading `v` stripped, when the ref looks like a
version. Not provided in `mirror.yml`.

## Python

Add `publish.yml` to `.github/workflows/`:

```yaml
name: Publish package to internal PyPI

on:
  release:
    types: [published]

jobs:
  package:
    uses: InkbridgeNetworks/package-repo/.github/workflows/python-package.yml@master
    with:
      package_name: <pypi package name>

      # How many commits to pull down from the src repository.
      # default: 1
      fetch_depth: 1

      # May be omitted if pyproject.toml is in the repo root.
      # default: '.'
      working_directory: <dir containing pyproject.toml>

      # A project file to make substitutions in; any @@VERSION@@
      # markers are replaced with the current tag version.
      project_file: <path to pyproject.toml>
    secrets: inherit
```

## Lua

Add `publish.yml` to `.github/workflows/`:

```yaml
name: Publish package to internal luarocks

on:
  release:
    types: [published]

jobs:
  package:
    uses: InkbridgeNetworks/package-repo/.github/workflows/lua-package.yml@master
    with:
      # .rockspec files need a format that indicates their version.
      # Can be as simple as `make`.
      build_cmd: <command to prepare .rockspec files>

      # How many commits to pull down from the src repository.
      # default: 1
      fetch_depth: 1

      # May be omitted if .rockspec files are in the repo root.
      # default: '.'
      working_directory: <dir containing .rockspec files>
    secrets: inherit
```

GitHub Actions searches for any `.rockspec` in `working_directory`
and runs `luarocks pack`. The resulting `.rock` files are committed
to the internal luarocks index.

## Mirroring GitHub repos

```yaml
on:
  push:
    branches:
      - '**'

jobs:
  mirror:
    if: github.event_name == 'push' && github.repository_owner == 'InkbridgeNetworks'
    uses: InkbridgeNetworks/package-repo/.github/workflows/mirror.yml@master
    with:
      # Defaults to the primary InkBridge git server if unset.
      mirror_host: <FQDN of git host>

      # Defaults to the GitHub repo name if unset.
      mirror_repo: <name of git repo to mirror to (without .git suffix)>
    secrets: inherit
```

The mirror deploy key needs `RW` permission in gitolite for each
repository being mirrored.
