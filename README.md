# publish-pypi-package-action

[![Test Local Action](https://github.com/durandtibo/publish-pypi-package-action/actions/workflows/test-local.yaml/badge.svg)](https://github.com/durandtibo/publish-pypi-package-action/actions/workflows/test-local.yaml)
[![Test Stable Action](https://github.com/durandtibo/publish-pypi-package-action/actions/workflows/test-stable.yaml/badge.svg)](https://github.com/durandtibo/publish-pypi-package-action/actions/workflows/test-stable.yaml)
[![License](https://img.shields.io/badge/license-BSD--3--Clause-blue)](LICENSE)
[![Latest release](https://img.shields.io/github/v/tag/durandtibo/publish-pypi-package-action?label=release)](https://github.com/durandtibo/publish-pypi-package-action/tags)

A composite GitHub Action that verifies a built Python package before publishing it to PyPI, and blocks
the workflow if that verification fails.

Point it at a workflow that already uploaded a `dist` artifact (e.g. from
[`durandtibo/build-pypi-package-action`](https://github.com/durandtibo/build-pypi-package-action)), and it
downloads that artifact, checks it hasn't been tampered with, verifies its Sigstore signatures on tag
pushes, and publishes it to PyPI via Trusted Publishing (OIDC) — no API tokens required.

## Why use it

Publishing to PyPI is the last step before a bad or tampered artifact becomes permanently installable by
everyone. This action exists to catch the usual ways that goes wrong _before_ the publish step runs:

- The `dist` workflow artifact was corrupted or tampered with between the build job and the publish job.
- A tag release is published without verifying the signatures produced at build time.
- Publishing relies on a long-lived API token instead of short-lived OIDC credentials.

If any of these are true, the action fails the job and nothing gets published.

## ⚠️ Known issue: this action does not currently work

The "Publish package to PyPI" step wraps
[`pypa/gh-action-pypi-publish`](https://github.com/pypa/gh-action-pypi-publish), which internally generates a
Docker container action at runtime and locates its image via the `github.action_repository`/
`github.action_ref` contexts. Those contexts are not resolved correctly when a Docker-backed composite
action is invoked from _inside another_ composite action
([`actions/runner#2473`](https://github.com/actions/runner/issues/2473)) — and that's exactly what this
action does, since it's itself a composite action wrapping `pypa/gh-action-pypi-publish`.

In practice, the generated trampoline action resolves to
`ghcr.io/durandtibo/publish-pypi-package-action:<sha>` (this repo's own image, which does not exist) instead
of `ghcr.io/pypa/gh-action-pypi-publish:<sha>`, and the job fails with a Docker pull error such as:

```
docker: Error response from daemon: Head "https://ghcr.io/v2/durandtibo/publish-pypi-package-action/manifests/<sha>": denied
```

This is a known, unsupported usage pattern upstream — see
[`pypa/gh-action-pypi-publish#291`](https://github.com/pypa/gh-action-pypi-publish/issues/291): the
maintainers explicitly note that calling `pypa/gh-action-pypi-publish` from within another composite action
is untested and unsupported, and there's no fix available at the runner level.

**Until this is resolved, don't use this action's publish step.** Call
`pypa/gh-action-pypi-publish` directly from your workflow instead (see
[`coola`'s `release-pypi.yaml`](https://github.com/durandtibo/coola/blob/main/.github/workflows/release-pypi.yaml)
for a working example that inlines the checksum/Sigstore verification steps and calls the publish action
straight from the workflow, with no wrapping composite action).

## How it works

1. Downloads the `dist` workflow artifact (wheel, sdist, `SHA256SUMS`) uploaded by an earlier job.
2. **Guard — tampering:** verifies every distribution against `SHA256SUMS` with `sha256sum --check`; fails
   closed before publishing anything if they don't match.
3. **Guard — tag pushes:** on a tag ref, downloads the `signatures` workflow artifact and verifies each
   distribution's Sigstore signature against the calling workflow's OIDC identity
   (`uvx sigstore verify identity`). Skipped on non-tag refs.
4. Publishes the verified distributions to PyPI via Trusted Publishing
   ([`pypa/gh-action-pypi-publish`](https://github.com/pypa/gh-action-pypi-publish)).

## Requirements

The calling workflow must provide:

| Requirement                                 | Notes                                                                              |
| ------------------------------------------- | ---------------------------------------------------------------------------------- |
| A `dist` workflow artifact                  | Must contain the wheel, sdist, and a `SHA256SUMS` file covering them.              |
| A `signatures` workflow artifact (tag refs) | Must contain the `*.sigstore*` bundles for each distribution. Only needed on tags. |
| `id-token: write` permission                | Required for PyPI Trusted Publishing (and Sigstore verification on tag refs).      |
| A PyPI Trusted Publisher                    | Registered for this repo, the calling workflow's filename, and its environment.    |

The calling job must also check out the repo (`actions/checkout`) before this action runs.

## Inputs

| Name                | Description                                                                                                                                                                                                      | Required | Default |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ------- |
| `workflow-filename` | Filename (not path) of the calling workflow, e.g. `release-pypi.yaml`. Must match the filename registered with the Trusted Publisher, since it's used to verify the Sigstore certificate identity on tag pushes. | Yes      | —       |
| `repository-url`    | Target package index URL, passed through to `pypa/gh-action-pypi-publish`. Leave unset to publish to PyPI, or set to `https://test.pypi.org/legacy/` to publish to TestPyPI instead.                             | No       | `""`    |

## Usage

### Basic

```yaml
- uses: actions/checkout@v7

- name: Publish package
  uses: durandtibo/publish-pypi-package-action@v0.0.1
  with:
    workflow-filename: release.yaml
  permissions:
    id-token: write
```

### Publish to TestPyPI

```yaml
- name: Publish package
  uses: durandtibo/publish-pypi-package-action@v0.0.1
  with:
    workflow-filename: ci.yaml
    repository-url: https://test.pypi.org/legacy/
```

### End-to-end release workflow

Builds with [`durandtibo/build-pypi-package-action`](https://github.com/durandtibo/build-pypi-package-action),
signs with Sigstore on tag pushes, then publishes — signature verification only runs on the tag push, since
that's the only ref this action signs and verifies:

```yaml
name: Release

on:
  workflow_dispatch:
  push:
    tags:
      - "v*"

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write # Sigstore keyless signing
    steps:
      - uses: actions/checkout@v7

      - name: Build package
        uses: durandtibo/build-pypi-package-action@v0.0.2

      - name: Sign distributions with Sigstore
        if: startsWith(github.ref, 'refs/tags/')
        run: uvx sigstore sign dist/*.tar.gz dist/*.whl

      - name: Upload signature bundles
        if: startsWith(github.ref, 'refs/tags/')
        uses: actions/upload-artifact@v7
        with:
          name: signatures
          path: dist/*.sigstore*

  publish:
    needs: build
    runs-on: ubuntu-latest
    environment: pypi
    permissions:
      id-token: write # PyPI Trusted Publishing
    steps:
      - uses: actions/checkout@v7

      - name: Publish package
        uses: durandtibo/publish-pypi-package-action@v0.0.1
        with:
          workflow-filename: release.yaml
```

## License

Distributed under the [BSD 3-Clause License](LICENSE).
