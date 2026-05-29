# busbar-actions/sf-org-delete

Delete a Salesforce scratch org and/or `OrgSnapshot` via direct API. Idempotent by default — safe in `if: always()` cleanup steps.

## What it does

1. Authenticates to the DevHub via [`busbar-auth`](../../busbar-extensions/crates/busbar-auth).
2. If `scratch-org-info-id` is given: queries the matching `ActiveScratchOrg` and deletes it (which deletes the scratch).
3. If `snapshot-id` is given: deletes the `OrgSnapshot` directly.

Either, or both, in one invocation.

## Inputs

| Input | Default | Description |
|---|---|---|
| `scratch-org-info-id` | `` | ScratchOrgInfo id (the `scratch-org-info-id` output from `sf-org-create`). |
| `snapshot-id` | `` | `OrgSnapshot` id (the `snapshot-id` output from `sf-snapshot-create`). |
| `ignore-missing` | `true` | Don't fail if the target doesn't exist. |
| `version` | `latest` | `sf-org-delete` release tag. |
| `binary-repo` | `busbar-actions/actions-dist` | Where to fetch the binary. |

## Env vars (from the Environment, per the busbar convention)

Same as `sf-org-create` / `sf-snapshot-create` — `SF_INSTANCE_URL` + the OIDC group, or `SF_ACCESS_TOKEN` for local dev.

## Example: end-of-run cleanup

```yaml
jobs:
  ci:
    runs-on: ubuntu-latest
    environment: devhub-pbo-scratch
    steps:
      - uses: busbar-actions/sf-org-create@v1
        id: org
        with:
          org-name: pr-${{ github.event.number }}
          admin-email: ci@example.com

      # … run tests, deploys, etc. …

      - uses: busbar-actions/sf-org-delete@v1
        if: always()
        with:
          scratch-org-info-id: ${{ steps.org.outputs.scratch-org-info-id }}
```
