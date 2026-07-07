# Foundry request files

This folder stores GitOps request manifests for provisioning Microsoft Foundry environments via platform automation.

## Process

1. Developer creates a request file using the template below.
2. Platform team reviews naming, quota, security metadata, and approvals.
3. After merge, automation (Step 3 workflow) provisions Azure resources with `azd`.

## Automation prerequisites

The workflow `.github/workflows/foundry-provision.yml` requires:

- GitHub Actions variable `AZURE_CLIENT_ID` (OIDC app registration client ID)
- GitHub Actions variable `AZURE_TENANT_ID`
- Federated credential from this repo to that app registration

The workflow triggers on `main` when a non-example file under `platform/requests/foundry/*.yaml` changes.

## Policy enforcement

The workflow validates requests against `platform/requests/foundry/policy.yaml`.

Current checks include:

- `apiVersion` must be `platform/v1`
- `kind` must be `FoundryInstanceRequest`
- `metadata.name` must match `<team>-<env>` where `env` is `dev`, `staging`, or `prod`
- template repo/path must match policy
- team/environment/location/data classification must be allowed by policy
- required ownership fields based on policy flags

## Status write-back

After provisioning, workflow writes status to:

- `platform/status/foundry/<request-name>.yaml`

The status file includes request metadata, run URL, result state, and selected outputs.

## File naming

Use `<team>-<env>.yaml`, for example:

- `team-research-dev.yaml`
- `team-research-staging.yaml`

## Minimal required fields

- `spec.template.version`
- `spec.azure.subscriptionId`
- `spec.azure.location`
- `spec.azure.resourceGroupName`
- `spec.params.baseName` (6-8 lowercase chars)
- `spec.params.yourPrincipalId` (36-char Entra object ID)

## Example

```yaml
apiVersion: platform/v1
kind: FoundryInstanceRequest
metadata:
  name: team-research-dev
spec:
  template:
    repo: CristianHz/app-stacks
    path: templates/microsoft-foundry-basic
    version: v1.0.0
  azure:
    subscriptionId: 00000000-0000-0000-0000-000000000000
    location: eastus2
    resourceGroupName: rg-chat-basic-abc123
  params:
    baseName: abc123
    yourPrincipalId: 11111111-1111-1111-1111-111111111111
    telemetryOptOut: false
  ownership:
    team: team-research
    requestor: dev1@contoso.com
    approver: platform-owner@contoso.com
    costCenter: CC-1234
    dataClassification: Internal
```
