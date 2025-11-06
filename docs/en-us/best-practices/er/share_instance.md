# Deploy Shared Instance

## Application Scenario

Enterprise Router (ER) is a high-performance, highly available enterprise-grade router service provided by Huawei Cloud, supporting enterprise-level network functions such as multi-VPC interconnection, dedicated line access, and VPN connections. ER service provides flexible routing policies and rich network connectivity capabilities, meeting complex enterprise network architecture requirements.

ER shared instances are an important feature of the ER service, allowing one account (owner) to share ER instances with other accounts (acceptors), implementing cross-account network resource sharing. Through RAM (Resource Access Manager) service, owners can precisely control sharing permissions, and acceptors can securely use shared ER instances. This configuration is suitable for multi-account environments, partner network sharing, cost optimization, and other scenarios. This best practice will introduce how to use Terraform to automatically deploy ER shared instances, including ER instance creation, RAM resource sharing, cross-account VPC connections, and attachment acceptance.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [ER Availability Zones Query Data Source (data.huaweicloud_er_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/er_availability_zones)
- [RAM Resource Permissions Query Data Source (data.huaweicloud_ram_resource_permissions)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/ram_resource_permissions)
- [RAM Resource Share Invitations Query Data Source (data.huaweicloud_ram_resource_share_invitations)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/ram_resource_share_invitations)

### Resources

- [ER Instance Resource (huaweicloud_er_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/er_instance)
- [RAM Resource Share Resource (huaweicloud_ram_resource_share)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ram_resource_share)
- [RAM Resource Share Accepter Resource (huaweicloud_ram_resource_share_accepter)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ram_resource_share_accepter)
- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [ER VPC Connection Resource (huaweicloud_er_vpc_attachment)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/er_vpc_attachment)
- [ER Attachment Accepter Resource (huaweicloud_er_attachment_accepter)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/er_attachment_accepter)

### Resource/Data Source Dependencies

```
data.huaweicloud_er_availability_zones.test
    └── huaweicloud_er_instance.test

huaweicloud_er_instance.test
    ├── data.huaweicloud_ram_resource_permissions.test
    │   └── huaweicloud_ram_resource_share.test
    └── huaweicloud_er_attachment_accepter.test

huaweicloud_ram_resource_share.test
    └── data.huaweicloud_ram_resource_share_invitations.test
        └── huaweicloud_ram_resource_share_accepter.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_er_vpc_attachment.test

huaweicloud_er_vpc_attachment.test
    └── huaweicloud_er_attachment_accepter.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Configure Multi-Account Provider

Add the following script to the TF file (e.g., main.tf) to configure multi-account Provider:

```hcl
# Owner account Provider configuration
provider "huaweicloud" {
  alias      = "owner"
  region     = var.region_name
  access_key = var.access_key
  secret_key = var.secret_key
}

# Acceptor account Provider configuration
provider "huaweicloud" {
  alias      = "principal"
  region     = var.region_name
  access_key = var.principal_access_key
  secret_key = var.principal_secret_key
}
```

**Parameter Description**:
- **owner**: Owner account Provider, used to create and share ER instances
- **principal**: Acceptor account Provider, used to accept shared ER instances

### 3. Query ER Availability Zone Information Through Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the results of which are used to create ER instances:

```hcl
# Get all ER availability zone information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create ER instances
data "huaweicloud_er_availability_zones" "test" {
  provider = huaweicloud.owner
}
```

**Parameter Description**:
- **provider**: Specify using owner account Provider for query

### 4. Create ER Instance

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an ER instance resource:

```hcl
variable "instance_name" {
  description = "ER instance name"
  type        = string
}

variable "instance_asn" {
  description = "ER instance ASN number"
  type        = number
  default     = 64512
}

variable "instance_description" {
  description = "ER instance description"
  type        = string
  default     = "The ER instance to share with other accounts"
}

variable "instance_enable_default_propagation" {
  description = "Whether to enable default propagation"
  type        = bool
  default     = true
}

variable "instance_enable_default_association" {
  description = "Whether to enable default association"
  type        = bool
  default     = true
}

variable "instance_auto_accept_shared_attachments" {
  description = "Whether to automatically accept shared attachments"
  type        = bool
  default     = false
}

# Create an ER instance resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_er_instance" "test" {
  provider = huaweicloud.owner

  availability_zones = slice(data.huaweicloud_er_availability_zones.test.names, 0, 1)

  name        = var.instance_name
  asn         = var.instance_asn
  description = var.instance_description

  enable_default_propagation     = var.instance_enable_default_propagation
  enable_default_association     = var.instance_enable_default_association
  auto_accept_shared_attachments = var.instance_auto_accept_shared_attachments
}
```

**Parameter Description**:
- **provider**: Specify using owner account Provider to create resources
- **availability_zones**: Availability zone list, using the first result from ER availability zone list query data source
- **name**: Instance name, assigned by referencing the input variable instance_name
- **asn**: ASN number, assigned by referencing the input variable instance_asn
- **description**: Instance description, assigned by referencing the input variable instance_description
- **enable_default_propagation**: Enable default propagation, assigned by referencing the input variable instance_enable_default_propagation
- **enable_default_association**: Enable default association, assigned by referencing the input variable instance_enable_default_association
- **auto_accept_shared_attachments**: Auto accept shared attachments, assigned by referencing the input variable instance_auto_accept_shared_attachments

### 5. Query RAM Resource Permissions

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the results of which are used to create RAM resource sharing:

```hcl
# Get ER instance RAM resource permission information, used to create resource sharing
data "huaweicloud_ram_resource_permissions" "test" {
  provider = huaweicloud.owner

  resource_type = "er:instances"

  depends_on = [huaweicloud_er_instance.test]
}
```

**Parameter Description**:
- **provider**: Specify using owner account Provider for query
- **resource_type**: Resource type, set to "er:instances" for ER instance resources
- **depends_on**: Dependency relationship, ensuring ER instance is created before querying permissions

### 6. Create RAM Resource Share

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a RAM resource share resource:

```hcl
variable "resource_share_name" {
  description = "Resource share name"
  type        = string
  default     = "resource-share-er"
}

variable "principal_account_id" {
  description = "Acceptor account ID"
  type        = string
}

variable "owner_account_id" {
  description = "Owner account ID"
  type        = string
}

# Create a RAM resource share resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_ram_resource_share" "test" {
  provider = huaweicloud.owner

  name          = var.resource_share_name
  principals    = [var.principal_account_id]
  resource_urns = ["er:${var.region_name}:${var.owner_account_id}:instances:${huaweicloud_er_instance.test.id}"]

  permission_ids = data.huaweicloud_ram_resource_permissions.test.permissions[*].id
}
```

**Parameter Description**:
- **provider**: Specify using owner account Provider to create resources
- **name**: Resource share name, assigned by referencing the input variable resource_share_name
- **principals**: Acceptor account ID list, assigned by referencing the input variable principal_account_id
- **resource_urns**: Resource URN list, containing complete resource identifier for ER instance
- **permission_ids**: Permission ID list, using permission IDs from RAM resource permissions query data source

### 7. Query RAM Resource Share Invitations

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the results of which are used to accept resource sharing:

```hcl
# Get pending RAM resource share invitation information, used to accept resource sharing
data "huaweicloud_ram_resource_share_invitations" "test" {
  provider = huaweicloud.principal

  status = "pending"

  depends_on = [huaweicloud_ram_resource_share.test]
}
```

**Parameter Description**:
- **provider**: Specify using acceptor account Provider for query
- **status**: Invitation status, set to "pending" for pending invitations
- **depends_on**: Dependency relationship, ensuring resource sharing is created before querying invitations

### 8. Accept RAM Resource Share

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a RAM resource share accepter resource:

```hcl
# Create a RAM resource share accepter resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_ram_resource_share_accepter" "test" {
  provider = huaweicloud.principal

  resource_share_invitation_id = try([for v in data.huaweicloud_ram_resource_share_invitations.test.resource_share_invitations : v.id if v.resource_share_id == huaweicloud_ram_resource_share.test.id][0], "")
  action                       = "accept"

  # After accepting the invitation, querying data.huaweicloud_ram_resource_share_invitations again will be empty
  # This resource is a one-time resource, add ignore_changes to prevent resource changes during terraform plan execution
  lifecycle {
    ignore_changes = [
      resource_share_invitation_id,
    ]
  }
}
```

**Parameter Description**:
- **provider**: Specify using acceptor account Provider to create resources
- **resource_share_invitation_id**: Resource share invitation ID, matching corresponding invitation ID from query results
- **action**: Operation type, set to "accept" to accept invitation
- **lifecycle.ignore_changes**: Lifecycle ignore changes, preventing issues caused by resource ID changes

### 9. Create Acceptor VPC

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an acceptor VPC resource:

```hcl
variable "principal_vpc_name" {
  description = "Acceptor VPC name"
  type        = string
}

variable "principal_vpc_cidr" {
  description = "Acceptor VPC CIDR block"
  type        = string
  default     = "192.168.0.0/16"
}

# Create an acceptor VPC resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_vpc" "test" {
  provider = huaweicloud.principal

  name = var.principal_vpc_name
  cidr = var.principal_vpc_cidr
}
```

**Parameter Description**:
- **provider**: Specify using acceptor account Provider to create resources
- **name**: VPC name, assigned by referencing the input variable principal_vpc_name
- **cidr**: VPC CIDR block, assigned by referencing the input variable principal_vpc_cidr

### 10. Create Acceptor VPC Subnet

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an acceptor VPC subnet resource:

```hcl
variable "principal_subnet_name" {
  description = "Acceptor VPC subnet name"
  type        = string
}

variable "principal_subnet_cidr" {
  description = "Acceptor VPC subnet CIDR block"
  type        = string
  default     = ""
  nullable    = false
}

variable "principal_subnet_gateway_ip" {
  description = "Acceptor VPC subnet gateway IP"
  type        = string
  default     = ""
  nullable    = false
}

# Create an acceptor VPC subnet resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_vpc_subnet" "test" {
  provider = huaweicloud.principal

  vpc_id     = huaweicloud_vpc.test.id
  name       = var.principal_subnet_name
  cidr       = var.principal_subnet_cidr != "" ? var.principal_subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.principal_subnet_gateway_ip != "" ? var.principal_subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**Parameter Description**:
- **provider**: Specify using acceptor account Provider to create resources
- **vpc_id**: VPC ID, assigned by referencing the acceptor VPC resource (huaweicloud_vpc.test) ID
- **name**: Subnet name, assigned by referencing the input variable principal_subnet_name
- **cidr**: Subnet CIDR block, prioritizes using input variable, calculates using cidrsubnet function if empty
- **gateway_ip**: Gateway IP, prioritizes using input variable, calculates using cidrhost function if empty

### 11. Create ER VPC Connection

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an ER VPC connection resource:

```hcl
variable "attachment_name" {
  description = "ER VPC connection name"
  type        = string
}

# Create an ER VPC connection resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_er_vpc_attachment" "test" {
  provider = huaweicloud.principal

  instance_id = huaweicloud_er_instance.test.id
  vpc_id      = huaweicloud_vpc.test.id
  subnet_id   = huaweicloud_vpc_subnet.test.id
  name        = var.attachment_name

  depends_on = [huaweicloud_ram_resource_share_accepter.test]
}
```

**Parameter Description**:
- **provider**: Specify using acceptor account Provider to create resources
- **instance_id**: ER instance ID, assigned by referencing the ER instance resource (huaweicloud_er_instance.test) ID
- **vpc_id**: VPC ID, assigned by referencing the acceptor VPC resource (huaweicloud_vpc.test) ID
- **subnet_id**: Subnet ID, assigned by referencing the acceptor VPC subnet resource (huaweicloud_vpc_subnet.test) ID
- **name**: Connection name, assigned by referencing the input variable attachment_name
- **depends_on**: Dependency relationship, ensuring RAM resource share acceptance is completed before creating VPC connection

### 12. Accept ER Connection

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an ER attachment accepter resource:

```hcl
# Create an ER attachment accepter resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_er_attachment_accepter" "test" {
  provider = huaweicloud.owner

  instance_id   = huaweicloud_er_instance.test.id
  attachment_id = huaweicloud_er_vpc_attachment.test.id
  action        = "accept"
}
```

**Parameter Description**:
- **provider**: Specify using owner account Provider to create resources
- **instance_id**: ER instance ID, assigned by referencing the ER instance resource (huaweicloud_er_instance.test) ID
- **attachment_id**: Attachment ID, assigned by referencing the ER VPC connection resource (huaweicloud_er_vpc_attachment.test) ID
- **action**: Operation type, set to "accept" to accept connection

### 13. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Acceptor account authentication information
principal_access_key = "your_principal_access_key"
principal_secret_key = "your_principal_secret_key"

# ER instance configuration
instance_name = "tf_test_er_instance"

# RAM resource sharing configuration
resource_share_name  = "tf_test_resource_share"
principal_account_id = "your_principal_account_id"
owner_account_id     = "your_owner_account_id"

# Acceptor VPC configuration
principal_vpc_name    = "tf_test_er_instance_vpc"
principal_subnet_name = "tf_test_er_instance_subnet"

# ER VPC connection configuration
attachment_name = "tf_test_er_attachment"
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="instance_name=my-er" -var="principal_account_id=123456"`
2. Environment variables: `export TF_VAR_instance_name=my-er`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 14. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating ER shared instances
4. Run `terraform show` to view the created ER shared instances

## Reference Information

- [Huawei Cloud ER Product Documentation](https://support.huaweicloud.com/er/index.html)
- [Huawei Cloud RAM Product Documentation](https://support.huaweicloud.com/ram/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For ER Shared Instance](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/er/share-instance)
