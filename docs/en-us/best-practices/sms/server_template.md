# Deploy Server Template

## Application Scenario

Server Migration Service (SMS) is a server migration service provided by Huawei Cloud, supporting migration of physical servers, virtual machines, or servers from other cloud platforms to Huawei Cloud, achieving seamless business migration. Server templates are important resources of SMS service, used to define target server configuration information, including availability zones, specifications, networks, and other parameters. By creating server templates, configuration templates for target servers can be provided for migration tasks, achieving standardization and automation of the migration process. This best practice introduces how to use Terraform to automatically deploy server templates, including querying availability zones and creating server templates.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### Resources

- [Server Template Resource (huaweicloud_sms_server_template)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sms_server_template)

### Resource/Data Source Dependencies

```text
data.huaweicloud_availability_zones
    └── huaweicloud_sms_server_template
```

> Note: Server templates need to specify availability zones for creating target servers. Availability zone information in the current region can be obtained by querying the availability zones list data source.

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones List

Add the following script to the TF file (such as main.tf) to query the availability zones list:

```hcl
# Query availability zones list data source
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
- This data source does not require input parameters and will automatically query all availability zones in the current region

> Note: The availability zones list is used to specify availability zones when creating server templates later.

### 3. Create Server Template

Add the following script to the TF file (such as main.tf) to create a server template:

```hcl
variable "server_template_name" {
  description = "The server template name"
  type        = string
}

# Create server template resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_sms_server_template" "test" {
  name              = var.server_template_name
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
}
```

**Parameter Description**:
- **name**: Server template name, assigned by referencing the input variable `server_template_name`
- **availability_zone**: Availability zone, assigned by referencing the first availability zone name from the availability zones list data source, using `try` function to handle possible null values

> Note: Server templates are used to define target server configurations, including availability zones, specifications, and other information. Server template IDs need to be referenced when creating migration tasks. Availability zones need to be selected according to actual requirements, ensuring sufficient resources in the target region.

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Server template configuration (Required)
server_template_name = "tf_test_sms_server_template"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="server_template_name=tf_test_sms_server_template"`
2. Environment variables: `export TF_VAR_server_template_name=tf_test_sms_server_template`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the server template
4. Run `terraform show` to view the created server template

## Reference Information

- [Huawei Cloud SMS Product Documentation](https://support.huaweicloud.com/sms/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Server Template](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/sms/server-template)
