# Deploy Organization

## Application Scenario

Software Repository for Container (SWR) is a container image hosting service provided by Huawei Cloud, supporting storage, management, and distribution of Docker images, helping you achieve rapid deployment and continuous integration of container applications. Organizations are fundamental resources of SWR service, used to manage and organize container image repositories. By creating organizations, multiple image repositories can be created under the organization, achieving classified management and permission control of images. This best practice introduces how to use Terraform to automatically deploy organizations, including organization basic information configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [Organization Resource (huaweicloud_swr_organization)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/swr_organization)

### Resource/Data Source Dependencies

```text
huaweicloud_swr_organization
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Create Organization

Add the following script to the TF file (such as main.tf) to create an organization:

```hcl
variable "organization_name" {
  description = "The organization name"
  type        = string
}

# Create organization resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_swr_organization" "test" {
  name = var.organization_name
}
```

**Parameter Description**:
- **name**: Organization name, assigned by referencing the input variable `organization_name`

> Note: Organization names must be unique in SWR service, used to identify and manage container image repositories. After creating an organization, image repositories can be created under that organization.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Organization configuration (Required)
organization_name = "tf_test_swr_organization_name"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="organization_name=tf_test_swr_organization_name"`
2. Environment variables: `export TF_VAR_organization_name=tf_test_swr_organization_name`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the organization
4. Run `terraform show` to view the created organization

## Reference Information

- [Huawei Cloud SWR Product Documentation](https://support.huaweicloud.com/swr/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Organization](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/swr/organization)
