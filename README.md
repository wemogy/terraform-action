# ![wemogy](assets/wemogy-logo.png) Terraform (GitHub Action)

A GitHub Action that connects to a remote Terraform backend in Azure, applies or plans the changes and outputs the Terraform Output variables. Currently only works for Microsoft Azure.

The action authenticates against the state storage account via Azure AD (`use_azuread_auth`) — no storage account access key is required.

> [!CAUTION]
> The identity (Service Principal or federated/OIDC identity) needs a **data-plane** role such as **Storage Blob Data Contributor** on the backend storage account or container. Control-plane roles like _Contributor_ or _Owner_ are **not** sufficient — `terraform init` will fail with a `403` on the state blob if this role is missing.

## Usage

### OIDC (recommended)

For every branch a federated credential is created with the subject
`repo:wemogy/<repository_name>:ref:refs/heads/<branch>`. The identity's client ID,
tenant ID and subscription ID (but **no** client secret) are distributed to the target
repository as the `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` and `AZURE_SUBSCRIPTION_ID`
variables.

The consuming workflow logs in via OIDC, so it needs the `id-token: write` permission:

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: actions/checkout@v4

  - name: Terraform
    uses: wemogy/terraform-action@1.6.2
    id: terraform
    with:
      working-directory: env/terraform
      client-id: ${{ vars.AZURE_CLIENT_ID }}
      tenant-id: ${{ vars.AZURE_TENANT_ID }}
      subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      backend-storage-account-name: myterraformstorage
      backend-container-name: terraform-state
      backend-key: terraform.tfstate

  - run: echo ${{ fromJSON(steps.terraform.outputs.output).my_output.value }}
```

Leaving `client-secret` empty switches the action into OIDC mode automatically.

### Client secret (legacy)

```yaml
- uses: actions/checkout@v4

- name: Terraform
  uses: wemogy/terraform-action@1.6.2
  id: terraform
  with:
    working-directory: env/terraform
    client-id: ${{ secrets.AZURE_APP_ID }}
    client-secret: ${{ secrets.AZURE_PASSWORD }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    backend-storage-account-name: myterraformstorage
    backend-container-name: terraform-state
    backend-key: terraform.tfstate

- run: echo ${{ fromJSON(steps.terraform.outputs.output).my_output.value }}
```

## Passing Terraform input variables

The action runs Terraform with `-input=false`, so any declared variable without a
value or default will cause the run to fail (instead of hanging on an interactive
prompt). Provide your Terraform variables via `TF_VAR_<name>` environment variables on
the step — they are inherited by the action's Terraform commands automatically:

```yaml
- name: Terraform
  uses: wemogy/terraform-action@1.6.2
  env:
    TF_VAR_azure_subscription_id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
  with:
    working-directory: env/terraform
    client-id: ${{ vars.AZURE_CLIENT_ID }}
    tenant-id: ${{ vars.AZURE_TENANT_ID }}
    subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
    # ...
```

## Performance

The action is tuned to keep the overhead around `terraform apply` small:

- **Provider cache** – downloaded providers are cached with `actions/cache`, keyed on
  your `.terraform.lock.hcl`. Commit the lock file for the most stable cache hits
  (`terraform providers lock -platform=linux_amd64` adds the hashes for the GitHub runner).
- **No implicit upgrade** – `terraform init` uses the provider versions from the lock
  file. Set `upgrade: true` to get the old `terraform init -upgrade` behavior.
- **No Azure CLI login** – Terraform authenticates on its own via the `ARM_*`
  environment variables (including OIDC), so the Azure CLI is only logged in when
  `azure-cli-login: true` is set.
- **Plan once** – with `plan: true` and `apply: true` (or `destroy: true`), the plan is
  saved and applied as-is instead of being computed a second time.
- **Pinned Terraform** – set `terraform-version` to skip the "latest" lookup and get
  reproducible runs.

If a run spends a long time in the apply phase even though nothing changed in your code,
check the plan output for attributes that change on every run (drift), and consider
`resource_provider_registrations = "none"` in your `azurerm` provider block to skip
the resource provider registration check at provider start-up.

## Inputs

| Input               | Description                                                                                      |
| ------------------- | ------------------------------------------------------------------------------------------------ |
| `working-directory` | **Required** The directory of your terraform scripts                                             |
| `workspace`         | The terraform workspace                                                                          |
| `plan`              | Plan the changes. Defaults to "false"                                                            |
| `apply`             | Apply the changes. Defaults to "true"                                                            |
| `destroy`           | Destroy the changes. Defaults to "false". Does not work, when `apply` is set to `true`.          |
| `force`             | Enforce changes, even if prevent_destroy is set to 'true'                                        |
| `client-id`         | **Required** The Azure Service Principal / Managed Identity Client ID                            |
| `client-secret`     | The Azure Service Principal Secret. Leave empty to authenticate via OIDC (federated credentials) |
| `tenant-id`         | **Required** The Azure Service Principal Tenant ID                                               |
| `subscription-id`   | The Azure Subscription ID. Required when authenticating via OIDC                                 |
| `backend-storage-account-name` | **Required** The name of the Azure Storage Account that hosts the remote Terraform backend |
| `backend-container-name` | **Required** The name of the storage container that hosts the remote Terraform backend |
| `backend-key`       | **Required** The name of the state file in the backend container                                 |
| `terraform-version` | The Terraform CLI version to install. Defaults to "latest"                                       |
| `cache-providers`   | Cache downloaded providers between runs (keyed on `.terraform.lock.hcl`). Defaults to "true"     |
| `upgrade`           | Run `terraform init -upgrade` instead of using the versions from `.terraform.lock.hcl`. Defaults to "false" |
| `azure-cli-login`   | Also log in to the Azure CLI (only needed if your Terraform code calls `az`). Defaults to "false" |

## Outputs

| Output   | Description                         |
| -------- | ----------------------------------- |
| `output` | The Terraform output in JSON format |
