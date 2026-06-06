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

## Outputs

| Output   | Description                         |
| -------- | ----------------------------------- |
| `output` | The Terraform output in JSON format |
