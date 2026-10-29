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

Two jobs:

1. `build` checks out the caller at the triggering commit and builds an OCI
   archive. It runs in the `<environment>-build` environment (or
   `build-environment`) and has no OIDC token.
2. `publish` runs no caller code. It pushes only the OCI data from `build`,
   authenticates through workload identity federation and signs the digest
   with keyless cosign. It runs in the `environment` environment.

The tag is `<commit>-<UTC yyyymmddHHMMSS>`, prefixed `prod-` when
`environment` is `production`.

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
| `workload-identity-provider` | (required) | `projects/<number>/locations/global/workloadIdentityPools/<pool>/providers/<provider>` |
| `service-account` | (required) | Publisher, `<name>@<project>.iam.gserviceaccount.com` |
| `image` | (required) | Image without tag, `<region>-docker.pkg.dev/<project>/<repository>/<image>` |
| `environment` | (required) | Environment of the publish job (deployment branch policy) |
| `build-environment` | `<environment>-build` | Environment of the build job; holds build variables and secrets |
| `context` | `.` | Build context within the checkout |
| `dockerfile` | `Dockerfile` | Dockerfile within the checkout |
| `target` | | Dockerfile target |
| `build-arg-vars` | | Variable names to pass as `--build-arg` |
| `build-secrets` | | Secret names to pass as BuildKit secrets (`RUN --mount=type=secret,id=NAME`) |
| `build-arg-secrets` | | Deprecated: secret names to pass as `--build-arg` |

Every build also gets `--build-arg SOURCE_COMMIT=<full commit SHA>`; declare
`ARG SOURCE_COMMIT` to use it. Values are passed through the Docker process
environment, never on the command line. One image per environment per
workflow run.

Outputs: `digest`, `image` (`<image>@<digest>`).

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
