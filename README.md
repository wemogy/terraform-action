# ![wemogy](assets/wemogy-logo.png) Terraform (GitHub Action)

A GitHub Action that connects to a remote Terraform backend in Azure, applies or plans the changes and outputs the Terraform Output variables. Currently only works for Microsoft Azure.

## Usage

```yaml
- uses: actions/checkout@v2

- name: Terraform
  uses: wemogy/terraform-action
  id: terraform
  with:
    working-directory: env/terraform
    client-id: ${{ secrets.AZURE_APP_ID }}
    client-secret: ${{ secrets.AZURE_PASSWORD }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    arm-access-key: ${{ secrets.TERRAFORM_BACKEND_ACCESS_KEY }}

- run: echo ${{ fromJSON(steps.terraform.outputs.output).my_output.value }}
```

## Inputs

| Input               | Description                                                                                      |
| ------------------- | ------------------------------------------------------------------------------------------------ |
| `working-directory` | **Required** The directory of your terraform scripts                                             |
| `workspace`         | The terraform workspace                                                                          |
| `apply`             | Apply the changes. If set to "false", only a Terraform plan will be executed. Defaults to "true" |
| `client-id`         | **Required** The Azure Service Pricipal Client ID                                                |
| `client-secret`     | **Required** The Azure Service Pricipal Secret                                                   |
| `tenant-id`         | **Required** The Azure Service Pricipal Tenant ID                                                |
| `arm-access-key`    | **Required** The Access Key to the Azure Storage Account that hosts the remote Terraform backend |

## Outputs

| Output   | Description                         |
| -------- | ----------------------------------- |
| `output` | The Terraform output in JSON format |
