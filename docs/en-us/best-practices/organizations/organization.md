# Deploy Organization

## Application Scenario

Huawei Cloud Organizations is a multi-account management service that helps enterprises centrally manage multiple Huawei Cloud accounts through organizational structure, achieving centralized resource management and unified permission control. An organization is the top-level resource of the Organizations service, used to create the root node of a multi-account architecture, and supports enabling policy types and configuring tags for the root organization.

This best practice is suitable for scenarios where you need to create an Organizations organization from scratch, enable policy types such as Service Control Policy (SCP), and set tags for the root organization. This best practice will introduce how to use Terraform to automatically deploy an Organizations organization, including organization creation and configuration of policy types and tags.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [Organizations Organization Resource (huaweicloud_organizations_organization)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/organizations_organization)

### Resource/Data Source Dependencies

```
huaweicloud_organizations_organization
```

> Note: The Organizations organization is an independent resource that does not depend on other resources or data sources. Each Huawei Cloud account can create only one organization. After creation, you can continue to create organizational units and accounts under the organization.

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) document for configuration introduction.

### 2. Create Organizations Organization

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an Organizations organization resource:

```hcl
variable "enabled_policy_types" {
  description = "The list of Organizations policy types to enable in the Organization Root"
  type        = list(string)
}

variable "root_tags" {
  description = "The key/value to attach to the root"
  type        = map(string)
}

# Create an Organizations organization resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_organizations_organization" "test" {
  enabled_policy_types = var.enabled_policy_types
  root_tags            = var.root_tags
}
```

**Parameter Description**:
- **enabled_policy_types**: List of policy types to enable at the organization root, assigned by referencing the input variable enabled_policy_types, for example, set to ["service_control_policy"] to enable service control policies
- **root_tags**: Tags attached to the organization root, assigned by referencing the input variable root_tags, used for classifying and managing the root organization

### 3. Preset Input Parameters for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration values. These input parameters need to be entered manually during subsequent deployment.
Terraform also provides a method to preset these configurations through a `terraform.tfvars` file to avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Organization configuration
enabled_policy_types = ["service_control_policy"]

root_tags = {
  key   = "value"
  owner = "terraform"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the contents when executing terraform commands; other names require adding `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read variable values from this file

In addition to using a `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command line parameters: `terraform apply -var='enabled_policy_types=["service_control_policy"]'`
2. Environment variables: `export TF_VAR_enabled_policy_types='["service_control_policy"]'`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform uses variable values in the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the organization
4. Run `terraform show` to view details of the created organization

## Reference Information

- [Huawei Cloud Organizations Product Documentation](https://support.huaweicloud.com/intl/en-us/organizations/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Organizations Organization](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/organizations/organization)
