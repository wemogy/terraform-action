# ![wemogy](assets/wemogy-logo.png) Terraform (GitHub Action)

A GitHub Action that connects to a remote Terraform backend in Azure, applies or plans the changes and outputs the Terraform Output variables. Currently only works for Microsoft Azure.

## Usage

```yaml
- uses: actions/checkout@v2

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
    backend-access-key: ${{ secrets.TERRAFORM_BACKEND_ACCESS_KEY }}

- run: echo ${{ fromJSON(steps.terraform.outputs.output).my_output.value }}
```

## Inputs

| Input               | Description                                                                                      |
| ------------------- | ------------------------------------------------------------------------------------------------ |
| `working-directory` | **Required** The directory of your terraform scripts                                             |
| `workspace`         | The terraform workspace                                                                          |
| `plan `             | Plan the changes. Defaults to "false"                                                            |
| `apply`             | Apply the changes. Defaults to "true"                                                            |
| `destroy`           | Destroy the changes. Defaults to "false". Does not work, when `apply` is set to `true`.          |
| `force`.            | "Enforce changes, even if prevent_destroy is set to 'true'"                                      |
| `client-id`         | **Required** The Azure Service Pricipal Client ID                                                |
| `client-secret`     | **Required** The Azure Service Pricipal Secret                                                   |
| `tenant-id`         | **Required** The Azure Service Pricipal Tenant ID                                                |
| `backend-access-key`    | **Required** The Access Key to the Azure Storage Account that hosts the remote Terraform backend |

## Outputs

| Output   | Description                         |
| -------- | ----------------------------------- |
| `output` | The Terraform output in JSON format |
