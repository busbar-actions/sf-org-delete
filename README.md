> [!WARNING]
> **`busbar-actions` is under heavy active development — expect breaking changes.**
> These repositories are public, but **not ready for use yet** — please don't depend on them.
> A pilot is starting soon: **[star and watch the busbar-actions organization](https://github.com/busbar-actions)** for the launch of Discussions and the pilot announcement.

# busbar-actions/sf-org-delete

Delete a Salesforce scratch org and/or `OrgSnapshot` via direct API. Idempotent by default — safe to drop into an `if: always()` cleanup step.

## What it does

The action installs the prebuilt `sf-org-delete` binary (via `busbar-actions/setup`) and runs it. The binary owns all logic and UX:

1. Authenticates to the DevHub via `busbar-auth` — **self-minting via GitHub OIDC by default** (set `target-instance` + grant `id-token: write`); the binary exchanges the runner's OIDC id-token for a short-lived DevHub session in-process. A handed-in `SF_ACCESS_TOKEN`/`SF_INSTANCE_URL` (via the `sf-access-token`/`sf-instance-url` inputs) is an optional local-dev override only.
2. If `scratch-org-info-id` is given: queries the matching `ActiveScratchOrg` and issues a `DELETE` on it, which deletes the scratch org.
3. If `snapshot-id` is given: issues a `DELETE` on the `OrgSnapshot` record directly.
4. Writes `GITHUB_OUTPUT`, a job-summary table, and notice annotations for any skipped (already-absent) targets.
5. Disposes the session — revoking the OIDC-minted DevHub token and zeroizing it.

Either selector, or both, may be passed in one invocation. With neither selector and `ignore-missing: true` (the default), the run is a clean no-op.

## Usage

```yaml
jobs:
  ci:
    runs-on: ubuntu-latest
    environment: devhub-pbo-scratch
    permissions:
      id-token: write   # REQUIRED — self-mint via OIDC, no token handoff
      contents: read
    steps:
      - uses: busbar-actions/sf-org-create@main
        id: org
        with:
          target-instance: ${{ vars.SF_INSTANCE_URL }}   # Busbar-equipped DevHub (busbar-pilot-demo2)
          org-name: pr-${{ github.event.number }}
          admin-email: ci@example.com

      # … run tests, deploys, etc. …

      - uses: busbar-actions/sf-org-delete@main
        if: always()   # tear down even when the job fails
        with:
          target-instance: ${{ vars.SF_INSTANCE_URL }}
          scratch-org-info-id: ${{ steps.org.outputs.scratch-org-info-id }}
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `target-instance` | no¹ | `` | **PRIMARY OIDC path.** Instance URL of the Busbar-equipped DevHub (e.g. `busbar-pilot-demo2`) to self-mint a short-lived token against. Maps to `SF_INSTANCE_URL`. |
| `scratch-org-info-id` | no | `` | `ScratchOrgInfo` id; its matching `ActiveScratchOrg` is deleted (which deletes the scratch). Use the `scratch-org-info-id` output from `sf-org-create`. |
| `snapshot-id` | no | `` | `OrgSnapshot` id, deleted directly. Use the `snapshot-id` output from `sf-snapshot-create`. |
| `ignore-missing` | no | `true` | Do not fail when nothing matches (idempotent cleanup). When `false`, an absent target or empty selector set is an error. |
| `eca-client-id` | no | `` | Optional OIDC tuning → `ECA_CLIENT_ID`. Baked default. |
| `token-handler` | no | `` | Optional OIDC tuning → `TOKEN_HANDLER_APEX`. Defaults to `BBGitHubTokenExchangeHandler`. |
| `oidc-audience` | no | `` | Optional OIDC tuning → `OIDC_AUDIENCE`. Defaults to the target instance URL. |
| `sf-instance-url` | no | `` | **Optional local-dev/advanced override** of the DevHub instance URL; wins over `target-instance`. |
| `sf-access-token` | no | `` | **Optional local-dev/advanced override only.** A pre-obtained DevHub token; when set the binary skips OIDC self-minting. Leave empty in CI. |
| `version` | no | `latest` | Release tag of the `sf-org-delete` binary to install. |
| `binary-repo` | no | `busbar-actions/actions-dist` | GitHub repo hosting the prebuilt binary releases. |

¹ `target-instance` is required for the default OIDC path (or supply the `sf-instance-url`/`sf-access-token` local-dev override).

At least one of `scratch-org-info-id` / `snapshot-id` should be provided. With neither and `ignore-missing: true`, the action is a no-op; with `ignore-missing: false`, it errors.

## Outputs

| Output | Description |
|---|---|
| `deleted-scratch-org` | `"true"` when a scratch org was deleted. |
| `deleted-snapshot` | `"true"` when a snapshot was deleted. |
| `scratch-org-info-id` | Echoes the requested `ScratchOrgInfo` id when provided. |
| `active-scratch-org-id` | The `ActiveScratchOrg` id that was deleted, when one was found. |
| `snapshot-id` | Echoes the requested `OrgSnapshot` id when provided. |

## Auth & permissions — OIDC self-mint (default)

> [!NOTE]
> **DevHub prerequisite.** In-process OIDC → DevHub requires a DevHub that has the
> **Busbar managed package installed and a trust rule** for the calling repo's
> workflow. The pilot DevHub `busbar-pilot-demo2` is set up this way — point
> `target-instance` at it (supplied via the workflow Environment's
> `SF_INSTANCE_URL` variable). You can also use the `sf-instance-url` +
> `sf-access-token` override inputs locally.

This action **self-mints** its DevHub token: the binary exchanges the runner's
GitHub OIDC id-token for a short-lived Salesforce session **in-process**, issues
the DELETEs, then revokes + zeroizes it (`SalesforceSession::dispose`). **No
Salesforce access token is ever handed to a script, written to `GITHUB_ENV`, or
passed as an action input/output.** There is **no** `org-auth` handoff step.

- Set **`target-instance`** (the Busbar-equipped DevHub URL, e.g. `busbar-pilot-demo2`) → `SF_INSTANCE_URL`, and grant **`permissions: id-token: write`** so the runner can mint the OIDC id-token.
- `busbar-auth` exchanges that OIDC token at the org via `BUSBAR_ECA_CLIENT_ID`/`eca-client-id`, `BUSBAR_TOKEN_HANDLER`/`token-handler`, `BUSBAR_OIDC_AUDIENCE`/`oidc-audience`. Busbar in the org must already trust this repo's workflow (`sf busbar trust request approve`); an untrusted repo gets a pending-trust error.
- **Local-dev/advanced override:** set the `sf-instance-url` + `sf-access-token` inputs; when `sf-access-token` is non-empty the OIDC path is skipped and the handed-in token is used directly (zeroized, not revoked, at exit).

The DevHub token is **owned by this action** end-to-end — minted (or handed-in), used only to issue the DELETEs, and disposed at the end of the run. Nothing is handed off to downstream steps, so no separate job-end cleanup is required for this action's token.

## Observability

The binary emits, when running under GitHub Actions:

- `GITHUB_OUTPUT` entries (above) for downstream steps.
- A **job summary** table reporting what was deleted/skipped.
- **Notice annotations** when a target was already absent and cleanup was skipped.

Diagnostic progress lines are written to stderr. On failure the binary annotates an `::error` and exits non-zero.
