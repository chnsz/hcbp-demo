# Deploy API with Custom Authentication

## Application Scenario

Huawei Cloud API Gateway (APIG) supports multiple authentication methods, including IAM authentication, APP authentication, etc. At the same time, API Gateway also supports users to use their own authentication methods (hereinafter referred to as custom authentication) to better兼容 existing business capabilities. Custom authentication can meet complex authentication requirements, such as integrating third-party authentication systems, implementing custom permission control logic, etc. This best practice will introduce how to use Terraform to automatically deploy APIs with custom authentication and how to use FunctionGraph functions to implement frontend authentication for APIs.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule Resource (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [API Gateway Instance Resource (huaweicloud_apig_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/apig_instance)
- [API Gateway Group Resource (huaweicloud_apig_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/apig_group)
- [API Definition Resource (huaweicloud_apig_api)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/apig_api)
- [Custom Authorizer Resource (huaweicloud_apig_custom_authorizer)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/apig_custom_authorizer)
- [Function Resource (huaweicloud_fgs_function)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/fgs_function)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_apig_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_apig_instance

huaweicloud_networking_secgroup
    └── huaweicloud_networking_secgroup_rule
        └── huaweicloud_apig_instance

huaweicloud_apig_instance
    └── huaweicloud_apig_group
        └── huaweicloud_apig_api
            └── huaweicloud_apig_custom_authorizer
                └── huaweicloud_fgs_function
```

## Implementation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones Required for API Gateway Instance Resource Creation Through Data Source

Add the following script to the TF file (such as main.tf) to inform Terraform to perform a data source query, the query results are used to create API Gateway instances:

```hcl
# Get all availability zone information in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating API Gateway instances
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
This data source does not require additional parameters and queries all available availability zone information in the current region by default.

### 3. Create VPC Resource

Add the following script to the TF file (such as main.tf) to inform Terraform to create VPC resources:

```hcl
variable "vpc_name" {
  description = "VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
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
  description = "Subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "Subnet CIDR block"
  type        = string
}

variable "subnet_gateway" {
  description = "Subnet gateway address"
  type        = string
}

# Create VPC subnet resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for deploying API Gateway instances
resource "huaweicloud_vpc_subnet" "test" {
  name       = var.subnet_name
  cidr       = var.subnet_cidr
  gateway_ip = var.subnet_gateway
  vpc_id     = huaweicloud_vpc.test.id
}
```

**Parameter Description**:
- **name**: Subnet name, assigned by referencing input variable subnet_name
- **cidr**: Subnet network segment, assigned by referencing input variable subnet_cidr
- **gateway_ip**: Gateway IP address, assigned by referencing input variable subnet_gateway
- **vpc_id**: ID of the VPC to which the subnet belongs, referencing the ID of the VPC resource created earlier

### 5. Create Security Group Resource

Add the following script to the TF file to inform Terraform to create security group resources:

```hcl
variable "secgroup_name" {
  description = "Security group name"
  type        = string
}

# Create security group resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for deploying API Gateway instances
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.secgroup_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Security group name, assigned by referencing input variable secgroup_name
- **delete_default_rules**: Whether to delete default rules, set to true to delete default rules

### 6. Create Security Group Rule Resource

Add the following script to the TF file to inform Terraform to create security group rule resources:

```hcl
# Create security group rule resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for configuring API Gateway instance access control
resource "huaweicloud_networking_secgroup_rule" "allow_web" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype        = "IPv4"
  protocol         = "tcp"
  port_range_min   = 80
  port_range_max   = 80
  remote_ip_prefix = "0.0.0.0/0"
  description      = "Allow HTTP access"
}

resource "huaweicloud_networking_secgroup_rule" "allow_https" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype        = "IPv4"
  protocol         = "tcp"
  port_range_min   = 443
  port_range_max   = 443
  remote_ip_prefix = "0.0.0.0/0"
  description      = "Allow HTTPS access"
}
```

**Parameter Description**:
- **security_group_id**: Security group ID, referencing the ID of the security group resource created earlier
- **direction**: Rule direction, set to "ingress" for inbound rules
- **ethertype**: Network protocol version, set to "IPv4"
- **protocol**: Protocol type, set to "tcp"
- **port_range_min/port_range_max**: Port range, set to 80 and 443 respectively
- **remote_ip_prefix**: Allowed access IP range, set to "0.0.0.0/0" to allow all IP access
- **description**: Rule description

### 7. Create API Gateway Instance Resource

Add the following script to the TF file to inform Terraform to create API Gateway instance resources:

```hcl
variable "apig_name" {
  description = "API Gateway instance name"
  type        = string
}

# Create API Gateway instance resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing)
resource "huaweicloud_apig_instance" "test" {
  name                  = var.apig_name
  edition              = "BASIC"
  vpc_id               = huaweicloud_vpc.test.id
  subnet_id            = huaweicloud_vpc_subnet.test.id
  security_group_id    = huaweicloud_networking_secgroup.test.id
  availability_zones   = [data.huaweicloud_availability_zones.test.names[0]]
  description          = "API Gateway instance"
  enterprise_project_id = "0"
}
```

**Parameter Description**:
- **name**: API Gateway instance name, assigned by referencing input variable apig_name
- **edition**: Instance specification, set to "BASIC"
- **vpc_id**: VPC ID, referencing the ID of the VPC resource created earlier
- **subnet_id**: Subnet ID, referencing the ID of the subnet resource created earlier
- **security_group_id**: Security group ID, referencing the ID of the security group resource created earlier
- **availability_zones**: Availability zone list, using the first availability zone from the availability zone list query data source
- **description**: Instance description
- **enterprise_project_id**: Enterprise project ID, set to "0"

### 8. Create API Gateway Group Resource

Add the following script to the TF file to inform Terraform to create API Gateway group resources:

```hcl
variable "group_name" {
  description = "API group name"
  type        = string
}

# Create API Gateway group resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing)
resource "huaweicloud_apig_group" "test" {
  name        = var.group_name
  description = "API group"
  instance_id = huaweicloud_apig_instance.test.id
}
```

**Parameter Description**:
- **name**: API group name, assigned by referencing input variable group_name
- **description**: Group description
- **instance_id**: API Gateway instance ID, referencing the ID of the API Gateway instance resource created earlier

### 9. Create Function Resource

Add the following script to the TF file to inform Terraform to create function resources:

```hcl
variable "function_name" {
  description = "Function name"
  type        = string
}

# Create function resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing)
resource "huaweicloud_fgs_function" "test" {
  name        = var.function_name
  app         = "default"
  handler     = "index.handler"
  memory_size = 128
  timeout     = 30
  runtime     = "Python3.6"
  code_type   = "inline"
  
  func_code = <<EOF
import json

def handler(event, context):
    # Get authentication header information
    token = event['headers'].get('Authorization', '')
    
    if not token:
        return {
            'statusCode': 401,
            'body': json.dumps({
                'message': 'Unauthorized'
            })
        }
    
    # Implement your authentication logic here
    # For example: verify token, check user permissions, etc.
    
    return {
        'statusCode': 200,
        'body': json.dumps({
            'principalId': 'user123',
            'context': {
                'userId': 'user123',
                'userRole': 'admin'
            }
        })
    }
EOF
}
```

**Parameter Description**:
- **name**: Function name, assigned by referencing input variable function_name
- **app**: Application to which the function belongs, set to "default"
- **handler**: Function entry point, set to "index.handler"
- **memory_size**: Function runtime memory size, set to 128MB
- **timeout**: Function timeout, set to 30 seconds
- **runtime**: Runtime environment, set to "Python3.6"
- **code_type**: Code type, set to "inline"
- **func_code**: Function code, containing custom authentication logic

### 10. Create Custom Authorizer Resource

Add the following script to the TF file to inform Terraform to create custom authorizer resources:

```hcl
variable "authorizer_name" {
  description = "Custom authorizer name"
  type        = string
}

# Create custom authorizer resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing)
resource "huaweicloud_apig_custom_authorizer" "test" {
  instance_id  = huaweicloud_apig_instance.test.id
  name         = var.authorizer_name
  type         = "FRONTEND"
  function_urn = huaweicloud_fgs_function.test.urn
  identities   = ["Authorization"]
}
```

**Parameter Description**:
- **instance_id**: API Gateway instance ID, referencing the ID of the API Gateway instance resource created earlier
- **name**: Custom authorizer name, assigned by referencing input variable authorizer_name
- **type**: Authorizer type, set to "FRONTEND"
- **function_urn**: Function URN, referencing the URN of the function resource created earlier
- **identities**: Authentication parameter list, set to ["Authorization"]

### 11. Create API Definition Resource

Add the following script to the TF file to inform Terraform to create API definition resources:

```hcl
variable "api_name" {
  description = "API name"
  type        = string
}

# Create API definition resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing)
resource "huaweicloud_apig_api" "test" {
  instance_id       = huaweicloud_apig_instance.test.id
  group_id         = huaweicloud_apig_group.test.id
  name             = var.api_name
  type             = "Public"
  request_protocol = "HTTPS"
  request_method   = "GET"
  request_path     = "/test"
  security_authentication = "CUSTOM"
  backend_type     = "HTTP"
  backend_path     = "/test"
  backend_method   = "GET"
  backend_address  = "https://example.com"
}
```

**Parameter Description**:
- **instance_id**: API Gateway instance ID, referencing the ID of the API Gateway instance resource created earlier
- **group_id**: API group ID, referencing the ID of the API Gateway group resource created earlier
- **name**: API name, assigned by referencing input variable api_name
- **type**: API type, set to "Public"
- **request_protocol**: Request protocol, set to "HTTPS"
- **request_method**: Request method, set to "GET"
- **request_path**: Request path, set to "/test"
- **security_authentication**: Security authentication method, set to "CUSTOM"
- **backend_type**: Backend type, set to "HTTP"
- **backend_path**: Backend path, set to "/test"
- **backend_method**: Backend method, set to "GET"
- **backend_address**: Backend address, set to "https://example.com"

### 12. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC configuration
vpc_name = "tf_test_vpc"
vpc_cidr = "192.168.0.0/16"

# Subnet configuration
subnet_name = "tf_test_subnet"
subnet_cidr = "192.168.1.0/24"
subnet_gateway = "192.168.1.1"

# Security group configuration
secgroup_name = "tf_test_secgroup"

# API Gateway configuration
apig_name = "tf_test_apig"
group_name = "tf_test_group"
api_name = "tf_test_api"

# Function configuration
function_name = "tf_test_function"

# Custom authorizer configuration
authorizer_name = "tf_test_authorizer"
```

**Usage Method**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content in this `tfvars` file when executing terraform commands, other names need to add `.auto` definition before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc" -var="subnet_name=my-subnet"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values in the following priority: command line parameters > variable files > environment variables > default values.

### 13. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating API Gateway custom authentication
4. Run `terraform show` to view the details of the created API Gateway custom authentication

## Reference Information

- [Huawei Cloud API Gateway Product Documentation](https://support.huaweicloud.com/apig/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [APIG Custom Authentication Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/apig)
