# Create proxy_cache Plugin

## Application Scenario

The proxy_cache plugin of API Gateway is a plugin used to cache API response data, which can significantly improve API response speed and reduce the load on backend services. The proxy_cache plugin supports configuring cache keys, cache policies, TTL (Time To Live), and other parameters to help you achieve flexible API response cache management. By properly configuring cache policies, you can reduce the pressure of repeated requests on backend services and improve the overall system performance and availability. This best practice will introduce how to use Terraform to automatically create the proxy_cache plugin for API Gateway.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [API Gateway Instance Resource (huaweicloud_apig_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/apig_instance)
- [API Gateway Plugin Resource (huaweicloud_apig_plugin)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/apig_plugin)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_apig_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_apig_instance

huaweicloud_networking_secgroup
    └── huaweicloud_apig_instance

huaweicloud_apig_instance
    └── huaweicloud_apig_plugin
```

## Implementation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones Required for API Gateway Instance Resource Creation Through Data Source

Add the following script to the TF file (such as main.tf) to inform Terraform to perform a data source query, the query results are used to create API Gateway instances:

```hcl
variable "availability_zones" {
  description = "The availability zones to which the instance belongs"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "availability_zones_count" {
  description = "The number of availability zones to which the instance belongs"
  type        = number
  default     = 1
}

# Get all availability zone information in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating API Gateway instances
data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) == 0 ? 1 : 0
}
```

**Parameter Description**:
- **count**: The number of data source creations, used to control whether to execute the availability zone list query data source. The data source is only created (i.e., the availability zone list query is executed) when `var.availability_zones` is empty.

### 3. Create VPC Resource

Add the following script to the TF file (such as main.tf) to inform Terraform to create VPC resources:

```hcl
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

# Create VPC resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for deploying API Gateway instances
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: VPC name, assigned by referencing input variable vpc_name
- **cidr**: VPC CIDR network segment, assigned by referencing input variable vpc_cidr

### 4. Create VPC Subnet Resource

Add the following script to the TF file to inform Terraform to create VPC subnet resources:

```hcl
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
}

# Create VPC subnet resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for deploying API Gateway instances
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing the ID of the VPC resource created earlier
- **name**: Subnet name, assigned by referencing input variable subnet_name
- **cidr**: Subnet network segment. When subnet_cidr is empty, it automatically calculates the subnet network segment from the VPC's CIDR; otherwise, it uses the value of input variable subnet_cidr
- **gateway_ip**: Gateway IP address. When subnet_gateway_ip is empty, it automatically gets the gateway IP from the calculated subnet network segment; otherwise, it uses the value of input variable subnet_gateway_ip

### 5. Create Security Group Resource

Add the following script to the TF file to inform Terraform to create security group resources:

```hcl
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# Create security group resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for deploying API Gateway instances
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Security group name, assigned by referencing input variable security_group_name
- **delete_default_rules**: Whether to delete default rules, set to true to delete default rules

### 6. Create API Gateway Instance Resource

Add the following script to the TF file to inform Terraform to create API Gateway instance resources:

```hcl
variable "instance_name" {
  description = "The name of the APIG instance"
  type        = string
}

variable "instance_edition" {
  description = "The edition of the APIG instance"
  type        = string
  default     = "BASIC"
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = null
}

# Create API Gateway instance resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing)
resource "huaweicloud_apig_instance" "test" {
  name                  = var.instance_name
  edition               = var.instance_edition
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  availability_zones    = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, var.availability_zones_count), null) : var.availability_zones
  enterprise_project_id = var.enterprise_project_id
}
```

**Parameter Description**:
- **name**: API Gateway instance name, assigned by referencing input variable instance_name
- **edition**: Instance specification, assigned by referencing input variable instance_edition, default value is "BASIC"
- **vpc_id**: VPC ID, referencing the ID of the VPC resource created earlier
- **subnet_id**: Subnet ID, referencing the ID of the subnet resource created earlier
- **security_group_id**: Security group ID, referencing the ID of the security group resource created earlier
- **availability_zones**: Availability zone list. When availability_zones is empty, it uses the result of the availability zone list query data source; otherwise, it uses the value of input variable availability_zones
- **enterprise_project_id**: Enterprise project ID, assigned by referencing input variable enterprise_project_id

### 7. Create API Gateway Plugin Resource

Add the following script to the TF file to inform Terraform to create API Gateway plugin resources:

```hcl
variable "plugin_name" {
  description = "The name of the APIG plugin"
  type        = string
}

variable "plugin_description" {
  description = "The description of the APIG plugin"
  type        = string
  default     = null
}

# Create API Gateway plugin resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing)
resource "huaweicloud_apig_plugin" "test" {
  instance_id = huaweicloud_apig_instance.test.id
  name        = var.plugin_name
  type        = "proxy_cache"
  description = var.plugin_description

  content = jsonencode({
    cache_key = {
      system_params = []
      parameters = [
        "custom_param"
      ],
      headers = []
    },
    cache_http_status_and_ttl = [
      {
        http_status = [
          202,
          203
        ],
        ttl = 5
      }
    ],
    client_cache_control = {
      mode  = "off",
      datas = []
    },
    cacheable_headers = [
      "X-Custom-Header"
    ]
  })
}
```

**Parameter Description**:
- **instance_id**: API Gateway instance ID, referencing the ID of the API Gateway instance resource created earlier
- **name**: Plugin name, assigned by referencing input variable plugin_name
- **type**: Plugin type, set to "proxy_cache" to create a proxy cache plugin
- **description**: Plugin description, assigned by referencing input variable plugin_description
- **content**: Plugin configuration content, encoded in JSON format, containing the following configuration items:
  - **cache_key**: Cache key configuration, including system parameters, custom parameters, and request headers
  - **cache_http_status_and_ttl**: Cache HTTP status code and TTL configuration, specifying which HTTP status code responses need to be cached and the cache time
  - **client_cache_control**: Client cache control configuration, setting client cache mode
  - **cacheable_headers**: List of cacheable request headers

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `terraform.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# VPC configuration
vpc_name              = "tf_test_apig_instance_vpc"

# Subnet configuration
subnet_name           = "tf_test_apig_instance_subnet"

# Security group configuration
security_group_name   = "tf_test_apig_instance_security_group"

# API Gateway instance configuration
instance_name         = "tf_test_apig_instance"
enterprise_project_id = "0"

# API Gateway plugin configuration
plugin_name           = "tf_test_apig_plugin"
plugin_description    = "Created by Terraform script"
```

**Usage Method**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content in this `tfvars` file when executing terraform commands, other names need to add `.auto` definition before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc" -var="subnet_name=my-subnet"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the proxy_cache plugin
4. Run `terraform show` to view the details of the created proxy_cache plugin

## Reference Information

- [Huawei Cloud API Gateway Product Documentation](https://support.huaweicloud.com/apig/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [APIG proxy_cache Plugin Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/apig/proxy-cache-plugin)
