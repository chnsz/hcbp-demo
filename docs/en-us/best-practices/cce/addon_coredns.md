# Deploy CoreDNS Addon

## Application Scenario

Cloud Container Engine (CCE) is a high-reliability, high-performance enterprise-grade container management service that supports Kubernetes community native applications and tools. CoreDNS addon is a DNS service addon provided by CCE, used to provide DNS resolution services for Pods in the cluster. CoreDNS is the default DNS server in Kubernetes clusters, responsible for resolving service names to IP addresses, enabling service discovery functionality. By deploying the CoreDNS addon, you can provide reliable DNS services for clusters, supporting efficient communication between services. This best practice will introduce how to use Terraform to automatically deploy a CCE CoreDNS addon, including querying CCE clusters and addon templates, as well as creating CCE addons.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [CCE Clusters Query Data Source (data.huaweicloud_cce_clusters)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/cce_clusters)
- [CCE Addon Template Query Data Source (data.huaweicloud_cce_addon_template)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/cce_addon_template)

### Resources

- [CCE Addon Resource (huaweicloud_cce_addon)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_addon)

### Resource/Data Source Dependencies

```
data.huaweicloud_cce_clusters
    └── data.huaweicloud_cce_addon_template
        └── huaweicloud_cce_addon

data.huaweicloud_cce_addon_template
    └── huaweicloud_cce_addon
```

## Implementation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query CCE Cluster Information Through Data Source (Optional)

Add the following script to the TF file (such as main.tf) to inform Terraform to query CCE cluster information (if cluster ID is not specified):

```hcl
variable "cluster_id" {
  description = "The ID of the CCE cluster"
  type        = string
  default     = ""

  validation {
    condition     = var.cluster_id != "" || var.cluster_name != ""
    error_message = "One of cluster_id or cluster_name is required"
  }
}

variable "cluster_name" {
  description = "The name of the CCE cluster"
  type        = string
  default     = ""
}

# Get CCE cluster information that meets the conditions in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating addon related resources
data "huaweicloud_cce_clusters" "test" {
  count = var.cluster_id == "" ? 1 : 0

  name = var.cluster_name
}
```

**Parameter Description**:
- **count**: The number of data source queries, used to control whether to query CCE cluster information, only when `var.cluster_id` is empty, the CCE cluster information is queried
- **name**: The name of the CCE cluster, assigned by referencing input variable `cluster_name`

### 3. Query CCE Addon Template Information Through Data Source

Add the following script to the TF file to inform Terraform to query CCE addon template information:

```hcl
variable "addon_template_name" {
  description = "The name of the CCE addon template"
  type        = string
  default     = "coredns"
}

variable "addon_version" {
  description = "The version of the CCE addon template"
  type        = string
}

# Get CCE addon template information that meets the conditions in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating addon related resources
data "huaweicloud_cce_addon_template" "test" {
  cluster_id = var.cluster_id != "" ? var.cluster_id : try(data.huaweicloud_cce_clusters.test[0].clusters[0].id, null)
  name       = var.addon_template_name
  version    = var.addon_version
}
```

**Parameter Description**:
- **cluster_id**: CCE cluster ID, if the cluster ID is specified, use that value, otherwise assign by referencing the first cluster ID from the CCE clusters list query data source
- **name**: The name of the addon template, assigned by referencing input variable `addon_template_name`, default is "coredns"
- **version**: The version of the addon template, assigned by referencing input variable `addon_version`

> Note: The addon version must be compatible with the cluster version. For example, if the cluster version is v1.32, the addon version should be selected from versions compatible with v1.32.

### 4. Create CCE Addon Resource

Add the following script to the TF file to inform Terraform to create CCE addon resources:

```hcl
# Create CCE addon resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for providing DNS services to the cluster
resource "huaweicloud_cce_addon" "test" {
  cluster_id    = var.cluster_id != "" ? var.cluster_id : try(data.huaweicloud_cce_clusters.test[0].clusters[0].id, null)
  template_name = var.addon_template_name
  version       = var.addon_version

  values {
    basic_json  = jsonencode(jsondecode(data.huaweicloud_cce_addon_template.test.spec).basic)
    custom_json = jsonencode(jsondecode(data.huaweicloud_cce_addon_template.test.spec).parameters.custom)
    flavor_json = jsonencode(jsondecode(data.huaweicloud_cce_addon_template.test.spec).parameters.flavor1)
  }
}
```

**Parameter Description**:
- **cluster_id**: CCE cluster ID, if the cluster ID is specified, use that value, otherwise assign by referencing the first cluster ID from the CCE clusters list query data source
- **template_name**: The name of the addon template, assigned by referencing input variable `addon_template_name`, default is "coredns"
- **version**: The version of the addon, assigned by referencing input variable `addon_version`
- **values**: Addon configuration values block
  - **basic_json**: Basic configuration JSON, parse the basic section from the addon template's spec and encode it as a JSON string
  - **custom_json**: Custom configuration JSON, parse the parameters.custom section from the addon template's spec and encode it as a JSON string
  - **flavor_json**: Flavor configuration JSON, parse the parameters.flavor1 section from the addon template's spec and encode it as a JSON string

> Note: The values configuration of the addon includes three parts: basic_json (basic configuration), custom_json (custom configuration), and flavor_json (flavor configuration). This example directly obtains all configurations from the addon template without additional modifications. If you need to customize the configuration, you can modify the addon template configuration before creating the addon, or merge the configuration through locals variables.

### 5. Preset Input Parameters Required for Resource Deployment

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, example content is as follows:

```hcl
cluster_id    = "your_cce_cluster_id"
addon_version = "1.30.33" # When the cluster version is v1.32, the addon version should be compatible with v1.32
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content in the `tfvars` file when executing terraform commands, other naming requires adding `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values according to actual needs
   - `cluster_id`: Replace with the actual CCE cluster ID
   - `addon_version`: Select the appropriate addon version based on the cluster version (the addon version should be compatible with the cluster version)
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command line parameters: `terraform apply -var="cluster_id=my-cluster-id" -var="addon_version=1.30.33"`
2. Environment variables: `export TF_VAR_cluster_id=my-cluster-id`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating CoreDNS addon
4. Run `terraform show` to view the created CoreDNS addon

## Reference Information

- [Huawei Cloud CCE Product Documentation](https://support.huaweicloud.com/cce/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For CCE CoreDNS Addon](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cce/addon-coredns)
