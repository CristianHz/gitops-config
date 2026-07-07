# Foundry GitHub OIDC Runbook (gitops-config)

This runbook configures GitHub Actions in `CristianHz/gitops-config` to deploy the Foundry template through OIDC (no client secret).

Workflow reference:

- `.github/workflows/foundry-provision.yml`

---

## 1. Prerequisites

- Azure CLI installed and logged in.
- Permission to create Entra app registrations.
- Permission to assign Azure RBAC roles on your test subscription.
- GitHub admin access to set repository Actions variables.

---

## 2. Set shell variables (copy/paste)

> Replace values before running.

```bash
# Required inputs
SUBSCRIPTION_ID="00000000-0000-0000-0000-000000000000"
TENANT_ID="00000000-0000-0000-0000-000000000000"
APP_NAME="gh-gitops-foundry-provisioner"
GITHUB_OWNER="CristianHz"
GITHUB_REPO="gitops-config"

# Branch used by workflow trigger
GITHUB_BRANCH="main"
```

Optional: set active subscription context.

```bash
az account set --subscription "$SUBSCRIPTION_ID"
```

---

## 3. Create Entra App Registration + Service Principal

```bash
APP_ID=$(az ad app create \
  --display-name "$APP_NAME" \
  --query appId -o tsv)

echo "APP_ID=$APP_ID"

SP_OBJECT_ID=$(az ad sp create \
  --id "$APP_ID" \
  --query id -o tsv)

echo "SP_OBJECT_ID=$SP_OBJECT_ID"
```

Notes:

- `APP_ID` is the client ID used in GitHub variable `AZURE_CLIENT_ID`.
- `TENANT_ID` goes to `AZURE_TENANT_ID`.

---

## 4. Create Federated Credential for GitHub OIDC

Create temporary file:

```bash
cat > federated-credential.json <<EOF
{
  "name": "github-${GITHUB_OWNER}-${GITHUB_REPO}-${GITHUB_BRANCH}",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:${GITHUB_OWNER}/${GITHUB_REPO}:ref:refs/heads/${GITHUB_BRANCH}",
  "audiences": [
    "api://AzureADTokenExchange"
  ]
}
EOF
```

Apply federated credential:

```bash
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters @federated-credential.json
```

Validation:

```bash
az ad app federated-credential list --id "$APP_ID" -o table
```

---

## 5. Assign Azure RBAC roles

### Fast start (recommended for first test)

Grant `Owner` on test subscription:

```bash
az role assignment create \
  --assignee-object-id "$SP_OBJECT_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Owner" \
  --scope "/subscriptions/${SUBSCRIPTION_ID}"
```

### Least privilege option (after first test)

Use both roles:

- `Contributor`
- `User Access Administrator`

At subscription or target resource-group scope.

```bash
az role assignment create \
  --assignee-object-id "$SP_OBJECT_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Contributor" \
  --scope "/subscriptions/${SUBSCRIPTION_ID}"

az role assignment create \
  --assignee-object-id "$SP_OBJECT_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "User Access Administrator" \
  --scope "/subscriptions/${SUBSCRIPTION_ID}"
```

---

## 6. Configure GitHub repository variables

In GitHub:

- Repository: `CristianHz/gitops-config`
- Settings > Secrets and variables > Actions > Variables

Create:

- `AZURE_CLIENT_ID` = `<APP_ID>`
- `AZURE_TENANT_ID` = `<TENANT_ID>`

No client secret is needed.

---

## 7. Verify GitHub Actions repo permissions

In repository settings:

- Settings > Actions > General > Workflow permissions
- Set `Read and write permissions`

Why: workflow writes status files back to repo.

---

## 8. Prepare first real request file

Copy example request and fill real values:

- `platform/requests/foundry/team-research-dev.example.yaml` -> `platform/requests/foundry/team-research-dev.yaml`

Required fields to edit:

- `spec.template.version` (existing tag in app-stacks, e.g. `v1.0.0`)
- `spec.azure.subscriptionId`
- `spec.azure.location` (must be allowed in policy)
- `spec.azure.resourceGroupName`
- `spec.params.baseName` (6-8 lowercase alphanumeric)
- `spec.params.yourPrincipalId` (36-char Entra object ID; recommended to use a group object ID)

Optional:

- `spec.params.telemetryOptOut`

---

## 9. Execute first test

1. Commit and push request file to `main`.
2. Wait for workflow `foundry-provision`.
3. Validate status output file:
   - `platform/status/foundry/<request-name>.yaml`

Expected:

- `status.state: success`
- `status.outputs.webAppUrl` populated

---

## 10. Quick troubleshooting

### A) `azure/login` fails with OIDC token/federation errors

- Verify federated credential subject exactly matches:
  - `repo:CristianHz/gitops-config:ref:refs/heads/main`
- Verify workflow branch trigger and branch name.
- Verify `AZURE_CLIENT_ID` and `AZURE_TENANT_ID` values.

### B) RBAC/authorization failures during deploy

- Ensure SP has enough rights:
  - First test: `Owner` at subscription scope.
- If using least privilege, ensure both:
  - `Contributor`
  - `User Access Administrator`

### C) Request rejected by policy validation

- Check `platform/requests/foundry/policy.yaml` allowlists.
- Confirm `metadata.name` format `<team>-<env>`.

### D) Status file not updated

- Confirm repo Action workflow permission is `Read and write`.
- Confirm workflow has `contents: write` (already set).

---

## 11. Cleanup notes

When you finish testing, you can remove broad rights and re-scope RBAC to least privilege.

Optional cleanup commands:

```bash
# Remove all role assignments for SP at subscription scope
az role assignment delete \
  --assignee-object-id "$SP_OBJECT_ID" \
  --scope "/subscriptions/${SUBSCRIPTION_ID}"

# Delete service principal
az ad sp delete --id "$APP_ID"

# Delete app registration
az ad app delete --id "$APP_ID"
```

---

## 12. Security recommendations (post-validation)

- Keep one identity per automation purpose.
- Restrict scope to RG where possible.
- Use Entra group object ID for `yourPrincipalId` instead of user IDs.
- Protect `main` branch with PR approvals.
- Keep policy allowlists strict in `platform/requests/foundry/policy.yaml`.
