# Deploy Hosted Connect

## Application Scenario

Direct Connect (DC) is a high-performance, low-latency, secure, and reliable dedicated line access service provided by Huawei Cloud, offering enterprises dedicated network connections from local data centers to Huawei Cloud. Direct Connect service supports multiple access methods, including physical dedicated lines and virtual dedicated lines, meeting network connection requirements for different scales and scenarios.

Hosted connect is a type of connection in Direct Connect service that allows creating virtual connections on existing operational connections, enabling multi-tenant sharing of physical dedicated line resources. Through hosted connects, you can reduce dedicated line costs, improve resource utilization, and meet network connection requirements for small and medium enterprises and multi-tenant scenarios. This best practice will introduce how to use Terraform to automatically deploy DC hosted connects, including connection creation, bandwidth configuration, and VLAN allocation.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

None

### Resources

- [DC Hosted Connect Resource (huaweicloud_dc_hosted_connect)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dc_hosted_connect)

### Resource/Data Source Dependencies

```
huaweicloud_dc_hosted_connect.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create DC Hosted Connect

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a DC hosted connect resource:

```hcl
variable "hosted_connect_name" {
  description = "Hosted connect name"
  type        = string
}

variable "hosted_connect_description" {
  description = "Hosted connect description"
  type        = string
  default     = "Created by Terraform"
}

variable "bandwidth" {
  description = "Hosted connect bandwidth size (Mbit/s)"
  type        = number
}

variable "hosting_id" {
  description = "Operational connection ID on which the hosted connect is based"
  type        = string
}

variable "vlan" {
  description = "VLAN assigned to the hosted connect"
  type        = number
}

variable "resource_tenant_id" {
  description = "Tenant ID to which the hosted connect belongs"
  type        = string
}

variable "peer_location" {
  description = "Location of the local facility at the other end of the connection"
  type        = string
  default     = ""
}

# Create a DC hosted connect resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_dc_hosted_connect" "test" {
  name               = var.hosted_connect_name
  description        = var.hosted_connect_description
  bandwidth          = var.bandwidth
  hosting_id         = var.hosting_id
  vlan               = var.vlan
  resource_tenant_id = var.resource_tenant_id
  peer_location      = var.peer_location
}
```

**Parameter Description**:
- **name**: Hosted connect name, assigned by referencing the input variable hosted_connect_name
- **description**: Hosted connect description, assigned by referencing the input variable hosted_connect_description
- **bandwidth**: Bandwidth size, assigned by referencing the input variable bandwidth, unit is Mbit/s
- **hosting_id**: Operational connection ID, assigned by referencing the input variable hosting_id, specifies the operational connection on which the hosted connect is based
- **vlan**: VLAN ID, assigned by referencing the input variable vlan, used for network isolation
- **resource_tenant_id**: Tenant ID, assigned by referencing the input variable resource_tenant_id, specifies the tenant to which the hosted connect belongs
- **peer_location**: Peer location, assigned by referencing the input variable peer_location, describes the location of the local facility at the other end of the connection

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Hosted connect configuration
hosted_connect_name        = "tf_test_hosted_connect"
hosted_connect_description = "Created by Terraform"
bandwidth                  = 100
hosting_id                 = "tf_test_hosting_id"
vlan                       = 100
resource_tenant_id         = "tf_test_resource_tenant_id"
peer_location              = "Beijing Data Center"
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="hosted_connect_name=my-connect" -var="bandwidth=100"`
2. Environment variables: `export TF_VAR_hosted_connect_name=my-connect`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the DC hosted connect
4. Run `terraform show` to view the details of the created DC hosted connect

## Reference Information

- [Huawei Cloud DC Product Documentation](https://support.huaweicloud.com/dc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DC Hosted Connect](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dc/hosted_connect)
