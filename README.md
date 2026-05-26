# workflows-public

Reusable GitHub Actions workflows for CapRover deployments.

## Quick Start

1. Configure reusable workflow policy at organization level (requires `admin:org` token scope):

- Organization Settings -> Actions -> General
- Allow actions and reusable workflows from `ultra-mega-apps/workflows-public`

2. Define organization-level variables for caller repositories:

- `FRGINBOX_CAPROVER_URL`
- `STORAGE_CAPROVER_URL`

3. Define organization-level secrets for caller repositories:

- `FRGINBOX_CAPROVER_PASSWORD`
- `STORAGE_CAPROVER_PASSWORD`

4. In caller repositories, reference this reusable workflow with a stable ref (`@main` or a tag) and use `secrets: inherit`.
5. Run one deployment with `server=frginbox` and one with `server=storage` to validate both routes.

### Configure With gh CLI

**Prerequisites** — run once if not already logged in or if the scope is missing:

```bash
# gh auth login            # uncomment if not yet authenticated
gh auth refresh -h github.com -s admin:org
```

**1) Organization-level Actions policy**

Requires `admin:org` scope.

```bash
ORG="ultra-mega-apps"

gh api -X PUT "/orgs/${ORG}/actions/permissions" \
  -f enabled_repositories="all" \
  -f allowed_actions="all"

gh api "/orgs/${ORG}/actions/permissions" \
  --jq '"enabled_repositories: \(.enabled_repositories)  |  allowed_actions: \(.allowed_actions)"'
```

**2) Shared repo — access policy** _(private/internal repos only)_

The access policy API rejects public repos with HTTP 422.
This block checks the repo visibility first and only runs when private or internal.

```bash
ORG="ultra-mega-apps"
SHARED_REPO="workflows-public"

VISIBILITY=$(gh api "/repos/${ORG}/${SHARED_REPO}" --jq '.visibility')

if [[ "${VISIBILITY}" == "private" || "${VISIBILITY}" == "internal" ]]; then
  gh api -X PUT "/repos/${ORG}/${SHARED_REPO}/actions/permissions/access" \
    -f access_level="organization"
else
  echo "Skipping access policy: repo is ${VISIBILITY} (not needed for public repos)"
fi
```

**3) Organization-level variables**

```bash
ORG="ultra-mega-apps"

gh variable set FRGINBOX_CAPROVER_URL \
  --org "${ORG}" \
  --visibility all \
  --body "https://frginbox.example.com"

gh variable set STORAGE_CAPROVER_URL \
  --org "${ORG}" \
  --visibility all \
  --body "https://storage.example.com"
```

**4) Organization-level secrets**

```bash
ORG="ultra-mega-apps"

# Use env vars so secret values are not written into shell history.
export FRGINBOX_CAPROVER_PASSWORD="replace-me"
export STORAGE_CAPROVER_PASSWORD="replace-me"

printf "%s" "${FRGINBOX_CAPROVER_PASSWORD}" | gh secret set FRGINBOX_CAPROVER_PASSWORD \
  --org "${ORG}" \
  --visibility all \
  --body -

printf "%s" "${STORAGE_CAPROVER_PASSWORD}" | gh secret set STORAGE_CAPROVER_PASSWORD \
  --org "${ORG}" \
  --visibility all \
  --body -
```

**5) Validate**

```bash
ORG="ultra-mega-apps"

gh variable list --org "${ORG}"
gh secret list --org "${ORG}"
```

## Workflows

- `.github/workflows/caprover_deployment.yaml`
  - Server-based deploy.
  - Picks URL and password using the `server` input and per-server naming convention.
- `.github/workflows/manual_caprover_deployment.yaml`
  - Manual host/password deploy.
  - Useful for ad-hoc deployments.

## 1) Server-based reusable workflow

Workflow file:

- `.github/workflows/caprover_deployment.yaml`

### Inputs

- `branch` (required): Git branch to deploy.
- `server` (required): Server key, for example `frginbox` or `storage`.
- `use_default_app_name` (optional, default `true`): Use caller repository name as app name.
- `custom_app_name` (optional): Required when `use_default_app_name` is `false`.

### Required variables and secrets in the caller repository

Variables:

- `FRGINBOX_CAPROVER_URL`
- `STORAGE_CAPROVER_URL`

Secrets:

- `FRGINBOX_CAPROVER_PASSWORD`
- `STORAGE_CAPROVER_PASSWORD`

The workflow computes a key from `server`:

- `frginbox` -> `FRGINBOX_CAPROVER_URL` and `FRGINBOX_CAPROVER_PASSWORD`
- `storage` -> `STORAGE_CAPROVER_URL` and `STORAGE_CAPROVER_PASSWORD`

### Caller example

```yaml
name: CapRover Deploy

on:
  workflow_dispatch:
    inputs:
      branch:
        description: Git branch to deploy
        required: true
        type: choice
        options: [development, production]
      server:
        description: Environment to deploy
        required: true
        type: choice
        options: [frginbox, storage]
      use_default_app_name:
        description: Use the repository name as the CapRover app name
        required: false
        type: boolean
        default: true
      custom_app_name:
        description: Custom CapRover app name (if not using the repository name)
        required: false
        type: string

jobs:
  deploy:
    uses: ultra-mega-apps/workflows-public/.github/workflows/caprover_deployment.yaml@main
    with:
      branch: ${{ inputs.branch }}
      server: ${{ inputs.server }}
      use_default_app_name: ${{ inputs.use_default_app_name }}
      custom_app_name: ${{ inputs.custom_app_name }}
    secrets: inherit
```

## 2) Manual host/password reusable workflow

Workflow file:

- `.github/workflows/manual_caprover_deployment.yaml`

### Inputs

- `caprover_host` (required): CapRover server URL.
- `caprover_password` (required): CapRover app token/password.
- `app_name` (required): CapRover app name.
- `branch` (required): Git branch to deploy.

### Caller example

```yaml
name: Manual CapRover Deploy

on:
  workflow_dispatch:
    inputs:
      caprover_host:
        description: CapRover server host URL
        required: true
        type: string
      app_name:
        description: CapRover app name
        required: true
        type: string
      branch:
        description: Git branch to deploy
        required: true
        type: string

jobs:
  deploy:
    uses: ultra-mega-apps/workflows-public/.github/workflows/manual_caprover_deployment.yaml@main
    with:
      caprover_host: ${{ inputs.caprover_host }}
      caprover_password: ${{ secrets.CAPROVER_PASSWORD }}
      app_name: ${{ inputs.app_name }}
      branch: ${{ inputs.branch }}
```

## Adding a new server

For server-based deploys, to add a new server key like `edge-eu`:

1. Add caller variable `EDGE_EU_CAPROVER_URL`.
2. Add caller secret `EDGE_EU_CAPROVER_PASSWORD`.
3. Allow selecting `edge-eu` in the caller workflow input options.

No per-server mapping changes are required in caller `with` or `secrets` blocks.

## Notes

- Keep `workflows-public` references pinned to a stable ref (`main` or a tag).
- Prefer tags for production stability.
