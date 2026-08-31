# Deploy Logstash Cluster

## Application Scenario

Cloud Search Service (CSS) is a fully managed online distributed search service built on Elasticsearch and OpenSearch by Huawei Cloud. It supports efficient retrieval and analysis of structured, unstructured text, and AI vectors. CSS Logstash clusters are based on open-source Logstash and provide data collection, transformation, cleansing, and parsing capabilities, helping users build stable and reliable data ingestion and processing pipelines.

This best practice introduces how to use Terraform to automatically deploy a CSS Logstash cluster, including creating VPC, subnet, and security group, and configuring node flavor, storage, and billing mode.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [CSS Flavors Query Data Source (data.huaweicloud_css_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/css_flavors)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [CSS Logstash Cluster Resource (huaweicloud_css_logstash_cluster)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/css_logstash_cluster)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_css_logstash_cluster

data.huaweicloud_css_flavors
    └── huaweicloud_css_logstash_cluster

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_css_logstash_cluster

huaweicloud_networking_secgroup
    └── huaweicloud_css_logstash_cluster
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) document for configuration introduction.

### 2. Query Availability Zones Required for CSS Logstash Cluster Creation Through Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, whose results are used to create the CSS Logstash cluster:

```hcl
variable "availability_zone" {
  description = "The availability zone to which the CSS logstash cluster belongs"
  type        = string
  default     = ""
  nullable    = false
}

# Query all availability zones in the specified region (defaults to the region specified in the provider block when region parameter is omitted) for creating CSS Logstash cluster
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**Parameter Description**:
- **count**: The number of data sources to create, used to control whether to execute the availability zone list query data source, only created when `var.availability_zone` is empty (i.e., execute availability zone list query)

### 3. Create VPC Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC resource:

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

# Create VPC resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted) for deploying CSS Logstash cluster
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The VPC CIDR block, assigned by referencing the input variable vpc_cidr, defaults to "192.168.0.0/16"

### 4. Create VPC Subnet Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC subnet resource:

```hcl
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

# Create VPC subnet resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted) for deploying CSS Logstash cluster
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing the ID of the previously created VPC resource (huaweicloud_vpc.test)
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The subnet CIDR block, uses cidrsubnet function to derive from the VPC CIDR block when subnet_cidr is empty, otherwise assigned by referencing the input variable subnet_cidr
- **gateway_ip**: The subnet gateway IP, uses cidrhost function to derive from the subnet CIDR block when subnet_gateway_ip is empty, otherwise assigned by referencing the input variable subnet_gateway_ip

### 5. Create Security Group Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a security group resource:

```hcl
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# Create security group resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted) for deploying CSS Logstash cluster
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: The security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete default rules, set to true

### 6. Query CSS Flavors Required for CSS Logstash Cluster Creation Through Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, whose results are used to create the CSS Logstash cluster:

```hcl
variable "cluster_flavor" {
  description = "The flavor of the CSS logstash cluster"
  type        = string
  default     = ""
  nullable    = false
}

# Query all CSS flavor information in the specified region (defaults to the region specified in the provider block when region parameter is omitted) for creating CSS Logstash cluster
data "huaweicloud_css_flavors" "test" {
  count = var.cluster_flavor == "" ? 1 : 0
}
```

**Parameter Description**:
- **count**: The number of data sources to create, used to control whether to execute the CSS flavor list query data source, only created when `var.cluster_flavor` is empty (i.e., execute CSS flavor list query)

### 7. Create CSS Logstash Cluster

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CSS Logstash cluster resource:

```hcl
variable "cluster_name" {
  description = "The name of the CSS logstash cluster"
  type        = string
}

variable "cluster_engine_version" {
  description = "The engine version of the CSS logstash cluster"
  type        = string
  default     = "7.10.0"
}

variable "cluster_instance_number" {
  description = "The number of instances of the CSS logstash cluster"
  type        = number
  default     = 1
}

variable "cluster_volume_type" {
  description = "The volume type of the CSS logstash cluster"
  type        = string
  default     = "ULTRAHIGH"
}

variable "cluster_volume_size" {
  description = "The volume size of the CSS logstash cluster"
  type        = number
  default     = 40
}

variable "charging_mode" {
  description = "The charging mode of the CSS logstash cluster"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The period unit of the CSS logstash cluster"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the CSS logstash cluster"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "The auto renew of the CSS logstash cluster"
  type        = string
  default     = "false"
}

# Create CSS Logstash cluster resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_css_logstash_cluster" "test" {
  name              = var.cluster_name
  engine_version    = var.cluster_engine_version
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id

  node_config {
    flavor          = var.cluster_flavor == "" ? try(data.huaweicloud_css_flavors.test[0].flavors[0].name, null) : var.cluster_flavor
    instance_number = var.cluster_instance_number

    volume {
      volume_type = var.cluster_volume_type
      size        = var.cluster_volume_size
    }
  }

  charging_mode = var.charging_mode
  period_unit   = var.period_unit
  period        = var.period
  auto_renew    = var.auto_renew
}
```

**Parameter Description**:
- **name**: The CSS Logstash cluster name, assigned by referencing the input variable cluster_name
- **engine_version**: The CSS Logstash cluster engine version, assigned by referencing the input variable cluster_engine_version, defaults to "7.10.0"
- **availability_zone**: The availability zone of the CSS Logstash cluster, assigned based on the availability zone list query data source (data.huaweicloud_availability_zones) when availability_zone is empty, otherwise assigned by referencing the input variable availability_zone
- **vpc_id**: The ID of the VPC to which the CSS Logstash cluster belongs, referencing the ID of the previously created VPC resource (huaweicloud_vpc.test)
- **subnet_id**: The ID of the subnet to which the CSS Logstash cluster belongs, referencing the ID of the previously created VPC subnet resource (huaweicloud_vpc_subnet.test)
- **security_group_id**: The ID of the security group associated with the CSS Logstash cluster, referencing the ID of the previously created security group resource (huaweicloud_networking_secgroup.test)
- **node_config**: The CSS Logstash cluster node configuration block
  - **flavor**: The CSS Logstash cluster node flavor, assigned based on the CSS flavor list query data source (data.huaweicloud_css_flavors) when cluster_flavor is empty, otherwise assigned by referencing the input variable cluster_flavor
  - **instance_number**: The number of CSS Logstash cluster nodes, assigned by referencing the input variable cluster_instance_number, defaults to 1
  - **volume**: The node storage configuration block
    - **volume_type**: The storage type, assigned by referencing the input variable cluster_volume_type, defaults to "ULTRAHIGH"
    - **size**: The storage capacity, assigned by referencing the input variable cluster_volume_size, defaults to 40
- **charging_mode**: The charging mode, assigned by referencing the input variable charging_mode, defaults to "postPaid"
- **period_unit**: The subscription period unit, assigned by referencing the input variable period_unit, defaults to null
- **period**: The subscription period, assigned by referencing the input variable period, defaults to null
- **auto_renew**: Whether to auto-renew, assigned by referencing the input variable auto_renew, defaults to "false"

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Network configuration
vpc_name            = "tf_test_logstash_cluster"
subnet_name         = "tf_test_logstash_cluster"
security_group_name = "tf_test_logstash_cluster"

# CSS Logstash cluster configuration
cluster_name = "tf_test_logstash_cluster"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="cluster_name=tf_test_logstash_cluster" -var="vpc_name=tf_test_logstash_cluster"`
2. Environment variables: `export TF_VAR_cluster_name=tf_test_logstash_cluster`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the CSS Logstash cluster
4. Run `terraform show` to view the created CSS Logstash cluster

## Reference Information

- [Huawei Cloud CSS Product Documentation](https://support.huaweicloud.com/css/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For CSS Logstash Cluster](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/css/logstash-cluster)
