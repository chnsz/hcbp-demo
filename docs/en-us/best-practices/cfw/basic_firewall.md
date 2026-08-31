# Deploy Basic Firewall

## Application Scenario

Cloud Firewall (CFW) is a new-generation cloud-native firewall that provides protection for internet boundaries and VPC boundaries on the cloud, including real-time intrusion detection and prevention, global unified access control, full traffic analysis visualization, and log auditing and traceability analysis. By purchasing a CFW firewall instance and configuring EIP auto-protection or manually binding existing EIPs, you can quickly enable internet boundary protection for public cloud assets.

This best practice introduces how to use Terraform to automatically deploy a basic CFW firewall, including purchasing a firewall instance, enabling EIP auto-protection, and optionally manually binding existing EIPs for protection.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [CFW Firewall Resource (huaweicloud_cfw_firewall)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cfw_firewall)
- [CFW EIP Auto Protection Resource (huaweicloud_cfw_eip_auto_protection)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cfw_eip_auto_protection)
- [CFW EIP Protection Resource (huaweicloud_cfw_eip_protection)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cfw_eip_protection)

### Resource/Data Source Dependencies

```
huaweicloud_cfw_firewall
    ├── huaweicloud_cfw_eip_auto_protection
    └── huaweicloud_cfw_eip_protection
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) document for configuration introduction.

### 2. Create CFW Firewall

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CFW firewall resource:

```hcl
variable "firewall_name" {
  description = "The CFW firewall name"
  type        = string
}

variable "firewall_flavor" {
  description = "The flavor version of the firewall"
  type        = string
  default     = "Professional"
}

variable "firewall_charging_mode" {
  description = "The charging mode of the firewall"
  type        = string
  default     = "postPaid"
}

variable "firewall_tags" {
  description = "The key/value pairs to associate with the resources"
  type        = map(string)
  default = {
    key = "value"
    foo = "bar"
  }
}

# Create CFW firewall resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cfw_firewall" "test" {
  name = var.firewall_name

  flavor {
    version = var.firewall_flavor
  }

  charging_mode = var.firewall_charging_mode
  tags          = var.firewall_tags
}
```

**Parameter Description**:
- **name**: The CFW firewall name, assigned by referencing the input variable firewall_name
- **flavor**: The firewall flavor configuration block
  - **version**: The firewall flavor version, assigned by referencing the input variable firewall_flavor, defaults to "Professional"
- **charging_mode**: The charging mode of the firewall, assigned by referencing the input variable firewall_charging_mode, defaults to "postPaid"
- **tags**: The key/value pairs associated with the firewall, assigned by referencing the input variable firewall_tags

### 3. Create EIP Auto Protection

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CFW EIP auto protection resource:

```hcl
variable "eip_auto_protection_status" {
  description = "Whether to enable auto-protection for EIPs. 1: enable, 0: disable"
  type        = number
  default     = 1
}

# Create CFW EIP auto protection resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cfw_eip_auto_protection" "test" {
  fw_instance_id = huaweicloud_cfw_firewall.test.id
  object_id      = try(huaweicloud_cfw_firewall.test.protect_objects[0].object_id, "")
  status         = var.eip_auto_protection_status
}
```

**Parameter Description**:
- **fw_instance_id**: The firewall instance ID, referencing the ID of the previously created CFW firewall resource (huaweicloud_cfw_firewall.test)
- **object_id**: The protected object ID, assigned based on the protect_objects returned by the previously created CFW firewall resource (huaweicloud_cfw_firewall.test)
- **status**: Whether to enable EIP auto-protection, assigned by referencing the input variable eip_auto_protection_status, 1 indicates enable, 0 indicates disable, defaults to 1

### 4. Create EIP Protection

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CFW EIP protection resource for manually binding existing EIPs for protection:

```hcl
variable "eip_protection_enabled" {
  description = "Whether to enable manual EIP protection for specific existing EIPs"
  type        = bool
  default     = false
}

variable "eip_protection_eip_ids" {
  description = "The list of existing EIPs to protect, each with id and public_ipv4"
  type = list(object({
    id          = string
    public_ipv4 = string
  }))
  default  = []
  nullable = false
}

# Create CFW EIP protection resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cfw_eip_protection" "test" {
  count = var.eip_protection_enabled && length(var.eip_protection_eip_ids) > 0 ? 1 : 0

  object_id = try(huaweicloud_cfw_firewall.test.protect_objects[0].object_id, "")

  dynamic "protected_eip" {
    for_each = var.eip_protection_eip_ids

    content {
      id          = protected_eip.value.id
      public_ipv4 = protected_eip.value.public_ipv4
    }
  }
}
```

**Parameter Description**:
- **count**: The number of EIP protection resources to create, only created when `var.eip_protection_enabled` is true and `var.eip_protection_eip_ids` is not empty
- **object_id**: The protected object ID, assigned based on the protect_objects returned by the previously created CFW firewall resource (huaweicloud_cfw_firewall.test)
- **protected_eip**: The protected EIP configuration, created through the dynamic block `dynamic "protected_eip"` based on the input variable eip_protection_eip_ids
  - **id**: The ID of the existing EIP, assigned by referencing the id in the input variable
  - **public_ipv4**: The public IPv4 address of the existing EIP, assigned by referencing the public_ipv4 in the input variable

> Note: When using only EIP auto-protection, keep `eip_protection_enabled` as false; to manually bind specific EIPs, set it to true and fill in the actual EIP information in `eip_protection_eip_ids`.

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# CFW firewall configuration
firewall_name          = "tf_test_cfw_firewall"
firewall_flavor        = "Professional"
firewall_charging_mode = "postPaid"
firewall_tags = {
  environment = "test"
  managed_by  = "terraform"
}

# EIP auto-protection configuration
eip_auto_protection_status = 1

# Manual EIP protection configuration
eip_protection_enabled = false
eip_protection_eip_ids = []
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="firewall_name=tf_test_cfw_firewall" -var="firewall_flavor=Professional"`
2. Environment variables: `export TF_VAR_firewall_name=tf_test_cfw_firewall`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the basic CFW firewall
4. Run `terraform show` to view the created basic CFW firewall

## Reference Information

- [Huawei Cloud CFW Product Documentation](https://support.huaweicloud.com/cfw/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For CFW Basic Firewall](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cfw/basic-firewall)
