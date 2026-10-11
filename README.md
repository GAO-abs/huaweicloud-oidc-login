# Huawei Cloud OIDC Login

A GitHub Action that logs a workflow in to Huawei Cloud using GitHub's OIDC token. It returns temporary credentials (access key, secret key and security token), so no long-lived AK/SK has to be stored in GitHub secrets.

## How it works

1. The action asks GitHub for an OIDC identity token for the current run.
2. It sends that token to Huawei Cloud STS (`POST /v5/agencies/assume-with-oidc`), naming your identity provider and trust agency.
3. Huawei Cloud verifies the token and returns temporary credentials.
4. The action masks the credentials in the logs and passes them to later steps as environment variables and step outputs.

## Prerequisites

The Huawei Cloud side must be set up before the action can log in:

- an OIDC identity provider in IAM for `https://token.actions.githubusercontent.com`
- a trust agency that trusts that provider, with an `oidc:sub` condition limiting it to your repository
- an identity policy on the agency granting what the pipeline needs

<!-- TODO: link to the console setup guide -->
See the console setup guide for the steps.

The calling workflow or job must also grant the `id-token: write` permission (see Usage).

## Usage

```yaml
permissions:
  id-token: write   # lets the job request an OIDC token
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Huawei Cloud
        uses: GAO-abs/huaweicloud-oidc-login@v1
        with:
          account-id: ${{ vars.HW_ACCOUNT_ID }}
          agency-name: github-actions
          region: af-south-1

      # Later steps are authenticated through HW_ACCESS_KEY,
      # HW_SECRET_KEY and HW_SECURITY_TOKEN.
```


## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `account-id` | Yes | | Huawei Cloud account ID that owns the identity provider and the trust agency. |
| `agency-name` | Yes | | Name of the trust agency to assume. |
| `region` | Yes | | Region of the STS endpoint to call, e.g. `af-south-1`. |
| `provider-name` | No | `GitHubActions` | Name of the OIDC identity provider in IAM. |
| `audience` | No | `huawei-cloud-service` | Audience requested for the GitHub token. Must match the audience on the identity provider and the trust agency. |
| `session-name` | No | `gha-<run id>-<run attempt>` | Agency session name, shown in audit logs. 2 to 128 characters: letters, digits and `_+=,.@-`. |
| `duration-seconds` | No | `3600` | Credential lifetime in seconds, from 900 up to the agency's maximum session duration. |
| `export-env` | No | `true` | Set to `false` to skip the environment variables and use the step outputs only. |

## Outputs

| Output | Description |
|---|---|
| `access-key` | Temporary access key (AK). |
| `secret-key` | Temporary secret key (SK). |
| `security-token` | Security token that must accompany the temporary AK/SK. |
| `expiration` | Expiry time of the credentials, UTC, ISO 8601. |

## Environment variables

Unless `export-env` is `false`, the action sets these for every later step in the job:

| Variable | Value |
|---|---|
| `HW_ACCESS_KEY` | Temporary access key |
| `HW_SECRET_KEY` | Temporary secret key |
| `HW_SECURITY_TOKEN` | Security token |

All three must be used together. Temporary credentials are rejected without the security token.

## Examples

### Using the outputs instead of environment variables

```yaml
      - name: Log in to Huawei Cloud
        id: hwc
        uses: GAO-abs/huaweicloud-oidc-login@v1
        with:
          account-id: ${{ vars.HW_ACCOUNT_ID }}
          agency-name: github-actions
          region: af-south-1
          export-env: "false"

      - name: Call Huawei Cloud with KooCLI
        run: |
          hcloud ECS NovaListServers \
            --cli-region=af-south-1 \
            --cli-access-key="${{ steps.hwc.outputs.access-key }}" \
            --cli-secret-key="${{ steps.hwc.outputs.secret-key }}" \
            --cli-security-token="${{ steps.hwc.outputs.security-token }}"
```

### A different agency per environment

```yaml
      - uses: GAO-abs/huaweicloud-oidc-login@v1
        with:
          account-id: ${{ vars.HW_ACCOUNT_ID }}
          agency-name: ${{ github.ref_name == 'main' && 'github-actions-prod' || 'github-actions-dev' }}
          region: af-south-1
```

### Shorter-lived credentials

```yaml
      - uses: GAO-abs/huaweicloud-oidc-login@v1
        with:
          account-id: ${{ vars.HW_ACCOUNT_ID }}
          agency-name: github-actions
          region: af-south-1
          duration-seconds: "900"
```

## Troubleshooting

When login fails, the action prints Huawei Cloud's response and the `sub` claim of the run's token, then stops the job.

| Symptom | Likely cause | Fix |
|---|---|---|
| `No OIDC token endpoint available` | The job has no `id-token: write` permission. | Add the `permissions` block shown in Usage. |
| STS returns a permission or access-denied error | The run's `sub` does not satisfy the agency's `oidc:sub` condition. | Compare the printed `sub` with the condition and adjust the condition. The match is case-sensitive. |
| STS returns an audience error | The `audience` input differs from the audience on the identity provider or the agency. | Use the same string in all three places. |
| STS returns a not-found error | Wrong `account-id`, `agency-name` or `provider-name`. | Check the names and the account ID against the IAM console. |
| `Could not reach https://sts.<region>...` | The region has no STS endpoint, or the name is misspelled. | Pick a region from the STS endpoint list in API Explorer. |
| Later steps get authentication errors | The step uses the AK/SK without the security token, or the credentials expired. | Pass all three values; raise `duration-seconds` for long jobs. |

Common `sub` formats issued by GitHub:

| Trigger | `sub` |
|---|---|
| Push to a branch | `repo:<org>/<repo>:ref:refs/heads/<branch>` |
| Tag | `repo:<org>/<repo>:ref:refs/tags/<tag>` |
| Pull request | `repo:<org>/<repo>:pull_request` |
| Job with an `environment` | `repo:<org>/<repo>:environment:<name>` |

## Security notes

- The credentials are masked in logs, but any step in the job can read them. Run the login step only in jobs that need it.
- Pull requests from forks cannot request an OIDC token, so the action fails there by design.
- Pin the action to a release tag or a commit SHA rather than a branch.
- Keep the agency's identity policy to the minimum the pipeline needs.

## Requirements

- A runner with `bash`, `curl` and `jq`. GitHub-hosted Ubuntu and macOS runners include all three.
- Outbound HTTPS access from the runner to `sts.<region>.myhuaweicloud.com`.

## References

- [AssumeAgencyWithOIDC API reference](https://support.huaweicloud.com/intl/en-us/api-iam5/AssumeAgencyWithOIDC.html)
- [Obtaining Temporary Security Credentials (IAM User Guide)](https://support.huaweicloud.com/intl/en-us/usermanual-iam5/iam_01_1237.html)
- [GitHub: About security hardening with OpenID Connect](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
