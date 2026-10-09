# `rearm-helm-action`

## About

GitHub Action to version and publish a Helm chart to an OCI-compliant registry (or ECR / ChartMuseum), and submit the release metadata to [ReARM](https://github.com/relizaio/rearm).

This action:

- Updates `Chart.yaml` `version:` to the ReARM-generated version
- Commits and pushes that change back to the repository
- Packages and publishes the Helm chart
- Computes the chart archive SHA256 digest and submits release metadata to ReARM

## Usage

```yaml
steps:
  - uses: actions/checkout@v4

  - uses: relizaio/rearm-helm-action@<release>
    with:
      rearm_api_id: <api-id-obtained-from-rearm>
      rearm_api_key: <api-key-obtained-from-rearm>

      registry_username: <registry-username>
      registry_password: <registry-password>
      registry_host: <registry-host>

      helm_chart_name: <chart-directory-name>
```

### Minimal workflow example

```yaml
name: Publish Helm chart

on:
  push:
    branches: [ main ]

permissions:
  contents: write

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: relizaio/rearm-helm-action@<release>
        with:
          rearm_api_id: ${{ secrets.REARM_API_ID }}
          rearm_api_key: ${{ secrets.REARM_API_KEY }}

          registry_username: ${{ secrets.REGISTRY_USERNAME }}
          registry_password: ${{ secrets.REGISTRY_PASSWORD }}
          registry_host: ${{ secrets.REGISTRY_HOST }}

          helm_chart_name: mychart
```

## Inputs

The action supports the following inputs:

- `registry_username`: Username for chart repository.
- `registry_password`: Password for chart repository.
- `registry_host`: Host for chart repository.
- `helm_chart_name`: Name of the helm chart (chart directory name).
- `rearm_api_id`: ReARM API ID (leave empty with `rearm_auth: github-oidc`).
- `rearm_api_key`: ReARM API KEY (leave empty with `rearm_auth: github-oidc`).
- `rearm_api_url`: ReARM API URL, optional, default: `https://demo.rearmhq.com`.
- `rearm_auth`: Credential mode, optional: `key` (API ID and key), or `github-oidc` (the job's GitHub identity token; needs `id-token: write` and a ReARM trust rule). Empty picks the mode from the credentials present.
- `rearm_org`: ReARM organization UUID, optional; only for `github-oidc` when several organizations trust the same repository.
- `path`: Path relative to the root of the repo (default is `.`).
- `ci_metadata`: Metadata for CI run, optional, default: `GitHub`.
- `rearm_component_id`: Component UUID for this repository if an org-wide key is used, optional.
- `registry_type`: Type of registry, optional, default: `OCI`. Supported: `OCI`, `ECR`, `CHARTMUSEUM`.
- `aws_region`: AWS region, required when `registry_type` is `ECR`.
- `enable_sbom`: Generates SBOM and stores it along with the artifact, optional, default: `false`.
- `enable_public_cosign_sigstore`: Sign SBOMs using public sigstore via cosign, optional, default: `false`.
- `successful_build_lifecycle`: Status on build success to be set on ReARM (`DRAFT` or `ASSEMBLED`), optional, default: `ASSEMBLED`.
- `enable_securesbom`: Enable SecureSBOM by ShiftLeftCyber signing of SBOMs, optional, default: `false`.
- `securesbom_pub_key_id`: Public key ID to sign with SecureSBOM by ShiftLeftCyber, optional, default: empty.
- `securesbom_host`: SecureSBOM (by ShiftLeftCyber) host, optional, default: empty.
- `securesbom_api_key`: SecureSBOM (by ShiftLeftCyber) API key, optional, default: empty.


### Federated (keyless) authentication

Instead of an API key, the action can authenticate to ReARM with the identity token GitHub issues to the job (`rearm_auth: github-oidc`). Grant the job `id-token: write`, create a trust rule for this repository in the ReARM organization, and leave `rearm_api_id` and `rearm_api_key` out:

```yaml
permissions:
  contents: write
  id-token: write
steps:
- uses: relizaio/rearm-helm-action@<ref>
  with:
    rearm_auth: github-oidc
    rearm_api_url: https://your.rearm.instance
    registry_username: <chart-registry-username>
    registry_password: <chart-registry-password>
    registry_host: <chart-registry-host>
    helm_chart_name: <chart-name>
```

Set `rearm_org` only when several ReARM organizations trust the same repository. The ReARM Mirror takes the same settings through `rearm_mirror_auth` and `rearm_mirror_org`, and needs its own trust rule on the mirror instance.

## Notes

- This action performs a `git commit` and `git push` of the updated `Chart.yaml` version. Your workflow typically needs `permissions: contents: write`, and branch protection rules must allow the workflow token to push (or you must use a custom token via `actions/checkout`).
- `registry_type: OCI` publishes via `helm registry login` and `helm push ... oci://...`.
- `registry_type: ECR` uses AWS ECR login; provide `aws_region` and AWS credentials via `registry_username` / `registry_password`.
- `registry_type: CHARTMUSEUM` uses a helper container to push to ChartMuseum.

