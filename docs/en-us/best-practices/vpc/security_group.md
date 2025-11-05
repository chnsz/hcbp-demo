# Deploy Security Group

## Application Scenario

Security Group is a virtual firewall in Huawei Cloud VPC used to control network access. By configuring security group rules, you can precisely control network access permissions for cloud servers, databases, and other resources. Security groups support both inbound and outbound rule configurations, effectively protecting the security of cloud resources. This best practice will introduce how to use Terraform to automatically deploy security groups and their rule configurations.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule Resource (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)

### Resource/Data Source Dependencies

```
huaweicloud_networking_secgroup
    └── huaweicloud_networking_secgroup_rule
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create Security Group Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a security group resource:

```hcl
variable "security_group_name" {
  description = "The name of the security group."
  type        = string
}

# Create security group resource
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete default rules, set to true to delete default rules

### 3. Create Security Group Rule Resources

Add the following script to the TF file to instruct Terraform to batch create security group rules:

```hcl
variable "security_group_rule_configurations" {
  description = "The list of security group rule configurations. Each item is a map with keys: direction, ethertype, protocol, ports, remote_ip_prefix."
  type = list(object({
    direction        = optional(string, "ingress")
    ethertype        = optional(string, "IPv4")
    protocol         = optional(string, null)
    ports            = optional(string, null)
    remote_ip_prefix = optional(string, "0.0.0.0/0")
  }))
  nullable = false
}

# Create security group rule resources
resource "huaweicloud_networking_secgroup_rule" "test" {
  count = length(var.security_group_rule_configurations)

  direction         = lookup(var.security_group_rule_configurations[count.index], "direction", "ingress")
  ethertype         = lookup(var.security_group_rule_configurations[count.index], "ethertype", "IPv4")
  protocol          = lookup(var.security_group_rule_configurations[count.index], "protocol", null)
  ports             = lookup(var.security_group_rule_configurations[count.index], "ports", null)
  remote_ip_prefix  = lookup(var.security_group_rule_configurations[count.index], "remote_ip_prefix", "0.0.0.0/0")
  security_group_id = huaweicloud_networking_secgroup.test.id
}
```

**Parameter Description**:
- **direction**: Rule direction, assigned by referencing the input variable security_group_rule_configurations, default value is "ingress"
- **ethertype**: Ethernet type, assigned by referencing the input variable security_group_rule_configurations, default value is "IPv4"
- **protocol**: Protocol type, assigned by referencing the input variable security_group_rule_configurations, default value is null
- **ports**: Port range, assigned by referencing the input variable security_group_rule_configurations, default value is null
- **remote_ip_prefix**: Remote IP address range, assigned by referencing the input variable security_group_rule_configurations, default value is "0.0.0.0/0"
- **security_group_id**: Security group ID, referencing the ID of the previously created security group resource

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Security group configuration
security_group_name = "tf_test_security_group"
security_group_rule_configurations = [
  # Allow all IPv4 ingress traffic of the ICMP protocol
  {
    direction        = "ingress"
    ethertype        = "IPv4"
    protocol         = "icmp"
    ports            = null
    remote_ip_prefix = "0.0.0.0/0"
  },
  # Allow some ports for IPv4 ingress traffic of the TCP protocol
  {
    direction        = "ingress"
    ethertype        = "IPv4"
    protocol         = "tcp"
    ports            = "22-23,443,3389,30100-30130"
    remote_ip_prefix = "0.0.0.0/0"
  },
  # Allow all IPv4 egress traffic
  {
    direction        = "egress"
    ethertype        = "IPv4"
    remote_ip_prefix = "0.0.0.0/0"
  },
  # Allow all IPv6 egress traffic
  {
    direction        = "egress"
    ethertype        = "IPv6"
    remote_ip_prefix = "::/0"
  }
]
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="security_group_name=my-security-group"`
2. Environment variables: `export TF_VAR_security_group_name=my-security-group`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating security groups and rules
4. Run `terraform show` to view the created security group and rule details

## Reference Information

- [Huawei Cloud VPC Product Documentation](https://support.huaweicloud.com/vpc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For VPC Security Group](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vpc/security-group)
