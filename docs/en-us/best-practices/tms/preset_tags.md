# Deploy Preset Tags

## Application Scenario

Tag Management Service (TMS) is a tag management service provided by Huawei Cloud, supporting adding, modifying, and deleting tags for cloud resources, helping you achieve resource classification management and cost analysis. Preset tags are important functions of TMS service, used to create reusable tag templates, achieving standardized tag management. By creating preset tags, commonly used tag key-value pairs can be defined, and these preset tags can be automatically applied when creating resources later, improving tag management efficiency and consistency. This best practice introduces how to use Terraform to automatically deploy preset tags, including preset tag creation and configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [Preset Tags Resource (huaweicloud_tms_tags)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/tms_tags)

### Resource/Data Source Dependencies

```text
huaweicloud_tms_tags
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Create Preset Tags

Add the following script to the TF file (such as main.tf) to create preset tags:

```hcl
variable "preset_tags" {
  description = "The preset tags to be applied to the resource"
  type        = list(object({
    key   = string
    value = string
  }))
}

# Create preset tags resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_tms_tags" "test" {
  dynamic "tags" {
    for_each = var.preset_tags

    content {
      key   = tags.value.key
      value = tags.value.value
    }
  }
}
```

**Parameter Description**:
- **tags**: Tag list, creates multiple tags through dynamic block `dynamic "tags"` based on input variable `preset_tags`
  - **key**: Tag key, assigned by referencing the `key` in the input variable
  - **value**: Tag value, assigned by referencing the `value` in the input variable

> Note: Preset tags are used to create reusable tag templates, supporting creating multiple tag key-value pairs. After creating preset tags, these tags can be automatically applied when creating resources later, achieving standardized tag management.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Preset tags configuration (Required)
preset_tags = [
  {
    key   = "foo"
    value = "bar"
  },
  {
    key   = "owner"
    value = "terraform"
  }
]
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="preset_tags=[{key=\"foo\",value=\"bar\"}]"`
2. Environment variables: `export TF_VAR_preset_tags='[{"key":"foo","value":"bar"}]'`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating preset tags
4. Run `terraform show` to view the created preset tags

## Reference Information

- [Huawei Cloud TMS Product Documentation](https://support.huaweicloud.com/tms/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Preset Tags](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/tms/preset-tags)
