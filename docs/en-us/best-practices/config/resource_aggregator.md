# Deploy Resource Aggregator

## Application Scenario

Configuration Audit (Config) is a one-stop compliance management service provided by Huawei Cloud, helping users continuously monitor and evaluate the configuration compliance of cloud resources. Config service provides pre-built compliance rule packages and custom rules, supporting multiple compliance frameworks and standards, helping enterprises establish a comprehensive compliance management system.

Resource aggregator is an important function of Config service, used to aggregate cloud resource information across accounts or organizations, achieving unified compliance management. Through resource aggregators, enterprises can centrally manage resources across multiple accounts or organizations, uniformly execute compliance check policies, and improve the efficiency and consistency of compliance management. Resource aggregators support both account-level and organization-level types, providing enterprises with flexible resource configuration aggregation solutions. This best practice will introduce how to use Terraform to automatically deploy Config resource aggregators, including aggregator creation and account configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [RMS Resource Aggregator Resource (huaweicloud_rms_resource_aggregator)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rms_resource_aggregator)

### Resource/Data Source Dependencies

```
huaweicloud_rms_resource_aggregator.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create Resource Aggregator

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a resource aggregator resource:

```hcl
variable "aggregator_name" {
  description = "Resource aggregator name"
  type        = string
}

variable "aggregator_type" {
  description = "Resource aggregator type, can be ACCOUNT or ORGANIZATION"
  type        = string
}

variable "account_ids" {
  description = "Source account ID list"
  type        = set(string)
  default     = []
}

# Create a resource aggregator resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_rms_resource_aggregator" "test" {
  name        = var.aggregator_name
  type        = var.aggregator_type
  account_ids = var.account_ids
}
```

**Parameter Description**:
- **name**: Resource aggregator name, assigned by referencing the input variable aggregator_name
- **type**: Resource aggregator type, assigned by referencing the input variable aggregator_type, supports ACCOUNT or ORGANIZATION
- **account_ids**: Source account ID list, assigned by referencing the input variable account_ids

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Resource aggregator configuration
aggregator_name = "tf_test_aggregator"
aggregator_type = "ACCOUNT"
account_ids     = ["account-id-1", "account-id-2", "account-id-3"]
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="aggregator_name=my-aggregator" -var="aggregator_type=ACCOUNT"`
2. Environment variables: `export TF_VAR_aggregator_name=my-aggregator`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the resource aggregator
4. Run `terraform show` to view the details of the created resource aggregator

## Reference Information

- [Huawei Cloud Config Product Documentation](https://support.huaweicloud.com/rms/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Config Resource Aggregator](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rms/resource-aggregator)
