# Deploy Idle Sessions Query and Kill

## Application Scenario

GaussDB is a high-performance, highly available, and highly secure enterprise-grade distributed relational database service provided by Huawei Cloud. During daily database O&M, sessions that stay idle for a long time continue to occupy connection resources and memory, which may exhaust the connection pool and degrade instance performance. Therefore, idle sessions need to be identified and cleaned up periodically.

This best practice will introduce how to use Terraform to query and kill idle sessions of a GaussDB instance. The solution first queries the available nodes and components of the instance through a data source, and then performs the idle session cleanup on the specified node based on the query result, helping O&M personnel automate session governance in an Infrastructure as Code manner.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [GaussDB Instance Node Information Query (data.huaweicloud_gaussdb_key_view_nodes_deliver)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/gaussdb_key_view_nodes_deliver)

### Resources

- [GaussDB Idle Sessions Query and Kill (huaweicloud_gaussdb_idle_sessions_query_and_kill)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_idle_sessions_query_and_kill)

### Resource/Data Source Dependencies

```
data.huaweicloud_gaussdb_key_view_nodes_deliver
    └── huaweicloud_gaussdb_idle_sessions_query_and_kill
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Query the Node Information of the GaussDB Instance

Add the following script in the TF file (such as main.tf):

```hcl
# Query the node information of the GaussDB instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_id" {
  description = "The ID of the GaussDB instance"
  type        = string
  default     = ""
}

data "huaweicloud_gaussdb_key_view_nodes_deliver" "test" {
  instance_id = var.instance_id
}
```

**Parameter Description**:
- **instance_id**: The ID of the GaussDB instance, assigned by referencing the input variable instance_id

### 3. Query and Kill Idle Sessions

Add the following script in the TF file (such as main.tf):

```hcl
# Query and kill idle sessions of the GaussDB instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_gaussdb_idle_sessions_query_and_kill" "test" {
  instance_id  = var.instance_id
  node_id      = data.huaweicloud_gaussdb_key_view_nodes_deliver.test.nodes.0.node_id
  component_id = data.huaweicloud_gaussdb_key_view_nodes_deliver.test.nodes.0.component_id
}
```

**Parameter Description**:
- **instance_id**: The ID of the GaussDB instance, assigned by referencing the input variable instance_id
- **node_id**: The ID of the node whose idle sessions are to be killed, assigned by referencing the node ID in the query result of the data source `huaweicloud_gaussdb_key_view_nodes_deliver`
- **component_id**: The ID of the component whose idle sessions are to be killed, assigned by referencing the component ID in the query result of the data source `huaweicloud_gaussdb_key_view_nodes_deliver`

> Note: Only nodes with CN or DN (primary, standby) components are supported. Deleting this resource only removes it from the Terraform state and does not undo the kill session operation.

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# GaussDB instance information
instance_id = "your_instance_id"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="instance_id=my-instance-id"`
2. Environment variables: `export TF_VAR_instance_id=my-instance-id`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start querying and killing the idle sessions of the GaussDB instance
4. Run `terraform show` to view the details of the queried and killed idle sessions

## Reference Information

- [Huawei Cloud GaussDB Product Documentation](https://support.huaweicloud.com/gaussdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GaussDB Idle Sessions Query and Kill](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/idle-sessions-query-and-kill)
