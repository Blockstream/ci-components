# ci-components

Shared reusable GitHub Actions workflows. Patch a job here once instead of in
every repository.

## Versions

Same scheme as our GitLab CI components: semver tags (`v1.2.0`) plus a moving
major branch (`v1`). Pin the major branch:

```yaml
uses: Blockstream/ci-components/.github/workflows/publish.yml@v1
```

- Changes land on `main` by pull request.
- A release is a pull request from `main` into `v1`, then a `v1.x.y` tag on
  the merge commit. Breaking changes start a `v2` branch.
- Release tags are immutable. Never create a tag named `v1`: it would shadow
  the branch.

Some consumers only trust the workflow at `refs/heads/v1` (for example to
grant registry credentials or to verify signatures), so a merge into `v1`
ships to every caller.

## `publish.yml`: build, publish and sign a container image

Three stages:

1. `prepare` validates public inputs and selects the build matrix without
   credentials or a checkout.
2. `build` checks out the caller at the triggering commit and builds an OCI
   archive. It runs in the `<environment>-build` environment (or
   `build-environment`) and has no OIDC token or registry login. With no
   `platforms` input, it runs once on `ubuntu-24.04` without `--platform`.
   Otherwise each platform builds separately on its native runner.
3. `publish` runs no caller code. It pushes only the OCI data from `build`,
   authenticates through workload identity federation and signs the digest
   with keyless cosign once per run. Platform archives are checked and merged
   into one OCI index before publishing. It runs in the `environment` environment.

The tag is `<commit>-<UTC yyyymmddHHMMSS>`, prefixed `prod-` when
`environment` is `production`. Non-empty `tags` replaces that derived tag
entirely, including the production prefix.

### Caller

```yaml
on:
  push:
    branches: [main]

permissions: {}

jobs:
  publish:
    permissions:
      contents: read
      id-token: write
    uses: Blockstream/ci-components/.github/workflows/publish.yml@v1
    with:
      # Each repository brings its own identity, e.g. repository variables.
      workload-identity-provider: ${{ vars.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      service-account: ${{ vars.GCP_SERVICE_ACCOUNT }}
      image: ${{ vars.IMAGE_NAME }}
      environment: production
      dockerfile: Dockerfile
      build-arg-vars: |
        PUBLIC_URL
      build-secrets: |
        NPM_TOKEN
    secrets: inherit # only when build-secrets or build-arg-secrets are used
```

### Inputs

| Input | Default | Use |
|---|---|---|
| `workload-identity-provider` | empty | Publisher identity provider; falls back to the same-named secret |
| `service-account` | empty | Publisher service account; falls back to the same-named secret |
| `image` | empty | Image without tag, or repository prefix when using `image-name`; falls back to the same-named secret |
| `image-name` | empty | Public relative image path appended to the resolved `image`, e.g. `plugins/example` |
| `tags` | empty | Newline-separated explicit tags; replaces the derived tag |
| `moving-tags` | empty | Newline-separated subset of `tags` that may change digest |
| `platforms` | empty | Newline-separated `linux/amd64` and/or `linux/arm64`; empty preserves the original single build |
| `build-args` | empty | Newline-separated literal `KEY=VALUE` pairs; public values only |
| `environment` | (required) | Environment of the publish job (deployment branch policy) |
| `build-environment` | `<environment>-build` | Environment of the build job; holds build variables and secrets |
| `context` | `.` | Build context within the checkout |
| `dockerfile` | `Dockerfile` | Dockerfile within the checkout |
| `target` | | Dockerfile target |
| `build-arg-vars` | | Variable names to pass as `--build-arg` |
| `build-secrets` | | Secret names to pass as BuildKit secrets (`RUN --mount=type=secret,id=NAME`) |
| `build-arg-secrets` | | Deprecated: secret names to pass as `--build-arg` |

Every build also gets `--build-arg SOURCE_COMMIT=<full commit SHA>`; declare
`ARG SOURCE_COMMIT` to use it. Named variable and secret values are passed
through the Docker process environment. Literal `build-args` values are passed
as individual command arguments, without shell evaluation; spaces and extra
`=` characters are preserved. Keys must match `[A-Za-z_][A-Za-z0-9_]*`.
Duplicate argument names and overrides of `SOURCE_COMMIT` are rejected.

Tags must match `^[A-Za-z0-9_][A-Za-z0-9._-]{0,127}$`. Explicit tags are immutable
unless listed in `moving-tags`. Before any push, every tag is checked: a matching
digest needs no update; a different digest on an immutable tag fails the job.
Only explicit registry responses indicating absence permit creating a tag.
Authentication and network errors fail closed. Empty lines are ignored; other
whitespace is significant. Duplicate tags and platforms are rejected.

`linux/amd64` uses `ubuntu-24.04`; `linux/arm64` uses `ubuntu-24.04-arm`.
No other platforms are accepted. Every platform build explicitly passes
`--platform`; publishing pushes one index and applies all requested tags to it.

`image-name` must match
`^[a-z0-9]+([._-][a-z0-9]+)*(/[a-z0-9]+([._-][a-z0-9]+)*)*$`.
For example, a secret prefix `<registry>/<project>/<repository>` and public
`image-name: plugins/example` resolve to
`<registry>/<project>/<repository>/plugins/example`. This supports several
images from one caller repository, with a public image name selected by the
caller's release logic. Distinct image names also receive distinct artifact
names. Without `image-name`, retain one image per environment per workflow run.

Outputs remain `digest` and `image` (`<effective-image>@<digest>`). When
`platforms` is set, the digest and signature identify the index. Otherwise they
identify the original single-build result.

### Publisher secrets

The optional secrets `workload-identity-provider`, `service-account`, and
`image` provide fallbacks for the same-named inputs. Each non-empty input takes
precedence; publishing fails clearly if both sources are empty. Prefer secrets
for private publisher settings in public callers, whose workflow logs are public.
Publisher settings are resolved and validated in `publish`; the plain
`image-name` is also validated before building. Secret-derived registry and
image names are masked, including derived hostnames.

Publisher secrets cannot be combined with `build-secrets` or
`build-arg-secrets`, including through `secrets: inherit`. The existing named
build-secret feature serializes the secrets context, and workflow expressions
cannot filter arbitrary keys out of it. The workflow rejects this combination
and withholds the context from `build` whenever any publisher secret is present.
Existing callers using publisher inputs and no same-named publisher secrets
retain the named build-secret feature. Public `build-args` and `build-arg-vars`
work with either publisher settings source.

GitHub may suppress the reusable workflow's `image` output when it contains a
secret. Consumers using a secret prefix can combine that secret, `image-name`,
and the public `digest` output themselves within a job that needs the reference.

### Versioned multi-platform release

This caller uses placeholder secret names and a public image path. A tag such as
`v1.2.3` publishes an immutable version tag for both platforms. Serialize
publishes for a given image so two runs cannot race between checking and writing
a tag; registry-enforced immutability can protect against other publishers too.

```yaml
on:
  push:
    tags: ['v*']

permissions: {}

concurrency:
  group: publish-example
  cancel-in-progress: false

jobs:
  release:
    permissions:
      contents: read
      id-token: write
    uses: Blockstream/ci-components/.github/workflows/publish.yml@v1
    with:
      environment: production
      image-name: plugins/example
      tags: ${{ github.ref_name }}
      platforms: |
        linux/amd64
        linux/arm64
      build-args: |
        VERSION=${{ github.ref_name }}
    secrets:
      workload-identity-provider: ${{ secrets.PUBLISHER_IDENTITY_PROVIDER }}
      service-account: ${{ secrets.PUBLISHER_SERVICE_ACCOUNT }}
      image: ${{ secrets.REGISTRY_PREFIX }}
```

### Moving `latest` on the default branch

Replace `main` with the caller's default branch. This example publishes an
immutable commit tag and permits only `latest` to move. Use the same concurrency
group as other workflows publishing this image.

```yaml
on:
  push:
    branches: [main]

permissions: {}

concurrency:
  group: publish-example
  cancel-in-progress: false

jobs:
  latest:
    permissions:
      contents: read
      id-token: write
    uses: Blockstream/ci-components/.github/workflows/publish.yml@v1
    with:
      environment: production
      image-name: plugins/example
      tags: |
        ${{ github.sha }}
        latest
      moving-tags: latest
      platforms: |
        linux/amd64
        linux/arm64
    secrets:
      workload-identity-provider: ${{ secrets.PUBLISHER_IDENTITY_PROVIDER }}
      service-account: ${{ secrets.PUBLISHER_SERVICE_ACCOUNT }}
      image: ${{ secrets.REGISTRY_PREFIX }}
```

The workflow sets no identity of its own. Which repository, ref and
environment may publish is enforced by the caller's identity provider
condition, the service account's workload identity binding and the
environment's deployment branch policy, not by this file.

Verify a published image:

```sh
cosign verify \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity https://github.com/Blockstream/ci-components/.github/workflows/publish.yml@refs/heads/v1 \
  --certificate-github-workflow-repository <owner>/<caller-repo> \
  <image>@<digest>
```

Always check `--certificate-github-workflow-repository`: this workflow is
public, so the identity alone does not say which repository built the image.
