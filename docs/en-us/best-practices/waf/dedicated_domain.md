# Deploy Professional Edition Domain

## Application Scenario

Huawei Cloud Web Application Firewall (WAF) Professional Edition Domain is a Web security protection service based on dedicated resources that can provide Web attack protection for specified domains.
By configuring WAF Professional Edition Domain, you can provide your website with protection capabilities against various common Web attacks such as SQL injection, XSS cross-site scripting, web trojan upload, command injection, and more.
This best practice will introduce how to use Terraform to automatically deploy a WAF Professional Edition Domain, including creating WAF Professional Edition instances, WAF policies, and domain configurations.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zone List Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Compute Flavor List Query Data Source (data.huaweicloud_compute_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [WAF Professional Edition Instance Resource (huaweicloud_waf_dedicated_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/waf_dedicated_instance)
- [WAF Policy Resource (huaweicloud_waf_policy)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/waf_policy)
- [WAF Professional Edition Domain Resource (huaweicloud_waf_dedicated_domain)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/waf_dedicated_domain)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    ├── huaweicloud_vpc_subnet
    ├── data.huaweicloud_compute_flavors
    └── huaweicloud_waf_dedicated_instance

data.huaweicloud_compute_flavors
    └── huaweicloud_waf_dedicated_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_waf_dedicated_instance
        └── huaweicloud_waf_dedicated_domain

huaweicloud_networking_secgroup
    └── huaweicloud_waf_dedicated_instance

huaweicloud_waf_dedicated_instance
    └── huaweicloud_waf_policy
        └── huaweicloud_waf_dedicated_domain
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Prerequisite Resource Preparation

This best practice requires creating prerequisite resources such as VPC, subnets, security groups, and WAF Professional Edition instances first. Please follow the following steps in the "Deploy WAF Professional Edition Instance" best practice for preparation:

- **Step 2**: Create VPC resource
- **Step 3**: Query availability zones required for WAF instance resource creation through data sources
- **Step 4**: Create VPC subnet resource
- **Step 5**: Query compute flavors required for WAF instance resource creation through data sources
- **Step 6**: Create security group resource
- **Step 7**: Create WAF Professional Edition instance resource

After completing the above steps, continue with the subsequent steps of this best practice.

### 3. Create WAF Policy Resource

Add the following script to the TF file to instruct Terraform to create a WAF policy resource:

```hcl
variable "policy_name" {
  description = "The WAF policy name"
  type        = string
}

variable "policy_level" {
  description = "The WAF policy level"
  type        = number
  default     = 1
}

# Create a WAF policy resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to configure protection rules for WAF Professional Edition Domain
resource "huaweicloud_waf_policy" "test" {
  name  = var.policy_name
  level = var.policy_level

  depends_on = [
    huaweicloud_waf_dedicated_instance.test
  ]
}
```

**Parameter Description**:
- **name**: WAF policy name, assigned by referencing the input variable policy_name
- **level**: WAF policy protection level, assigned by referencing the input variable policy_level, default value is 1
- **depends_on**: Resource dependency relationship, ensuring WAF Professional Edition instance has been created

### 4. Create WAF Professional Edition Domain Resource

Add the following script to the TF file to instruct Terraform to create a WAF Professional Edition Domain resource:

```hcl
variable "dedicated_mode_domain_name" {
  description = "The WAF dedicated mode domain name"
  type        = string
}

variable "dedicated_domain_client_protocol" {
  description = "The client protocol of the WAF dedicated domain"
  type        = string
  default     = "HTTP"
}

variable "dedicated_domain_server_protocol" {
  description = "The server protocol of the WAF dedicated domain"
  type        = string
  default     = "HTTP"
}

variable "dedicated_domain_address" {
  description = "The address of the WAF dedicated domain"
  type        = string
  default     = "192.168.0.14"
}

variable "dedicated_domain_port" {
  description = "The port of the WAF dedicated domain"
  type        = number
  default     = 8080
}

variable "dedicated_domain_type" {
  description = "The type of the WAF dedicated domain"
  type        = string
  default     = "ipv4"
}

# Create a WAF Professional Edition Domain resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_waf_dedicated_domain" "test" {
  domain      = var.dedicated_mode_domain_name
  policy_id   = huaweicloud_waf_policy.test.id
  keep_policy = true

  server {
    client_protocol = var.dedicated_domain_client_protocol
    server_protocol = var.dedicated_domain_server_protocol
    address         = var.dedicated_domain_address == "" ? cidrhost(huaweicloud_vpc_subnet.test.cidr, 4) : var.dedicated_domain_address
    port            = var.dedicated_domain_port
    type            = var.dedicated_domain_type
    vpc_id          = huaweicloud_vpc.test.id
  }
}
```

**Parameter Description**:
- **domain**: WAF Professional Edition Domain name, assigned by referencing the input variable dedicated_mode_domain_name
- **policy_id**: WAF policy ID, referencing the ID of the previously created WAF policy resource
- **keep_policy**: Whether to keep the policy, set to true to keep the policy
- **server**: Server configuration block
  - **client_protocol**: Client protocol, assigned by referencing the input variable dedicated_domain_client_protocol, default value is "HTTP"
  - **server_protocol**: Server protocol, assigned by referencing the input variable dedicated_domain_server_protocol, default value is "HTTP"
  - **address**: Server address, if dedicated_domain_address is empty, uses cidrhost function to get the 4th IP address from the subnet segment, otherwise uses the specified dedicated_domain_address
  - **port**: Server port, assigned by referencing the input variable dedicated_domain_port, default value is 8080
  - **type**: Server type, assigned by referencing the input variable dedicated_domain_type, default value is "ipv4"
  - **vpc_id**: VPC ID, referencing the ID of the previously created VPC resource

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
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

# WAF Professional Edition instance configuration
dedicated_instance_name               = "tf_test_waf_instance"
dedicated_instance_specification_code = "waf.instance.professional"
dedicated_instance_performance_type   = "normal"
dedicated_instance_cpu_core_count     = 4
dedicated_instance_memory_size        = 8

# Security group configuration
security_group_name = "tf_test_security_group"

# WAF policy configuration
policy_name = "tf_test_waf_policy"
policy_level = 1

# WAF Professional Edition Domain configuration
dedicated_mode_domain_name = "www.example.com"
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc" -var="subnet_name=my-subnet"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating WAF Professional Edition Domain
4. Run `terraform show` to view the created WAF Professional Edition Domain details

## Reference Information

- [Huawei Cloud WAF Product Documentation](https://support.huaweicloud.com/waf/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [WAF Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/waf)
