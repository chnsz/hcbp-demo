# Deploy Environment with Associated EIP

## Application Scenario

ServiceStage is a one-stop application hosting platform provided by Huawei Cloud. An environment is the infrastructure unit that hosts applications, bringing computing and network resources into a unified application deployment and operations system. In real-world scenarios, applications in an environment often need to access the internet or be accessed from the internet, so an Elastic IP (EIP) must be associated with the environment to provide public network access.

This best practice will introduce how to use Terraform to automatically create a VPC, an EIP, and a ServiceStage environment, and associate the EIP with the environment through the environment associate resource, thereby providing public network access for the environment.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Elastic IP (huaweicloud_vpc_eip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [ServiceStage Environment (huaweicloud_servicestagev3_environment)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/servicestagev3_environment)
- [ServiceStage Environment Associate (huaweicloud_servicestagev3_environment_associate)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/servicestagev3_environment_associate)

### Resource/Data Source Dependencies

```
huaweicloud_vpc
    └── huaweicloud_servicestagev3_environment
            └── huaweicloud_servicestagev3_environment_associate

huaweicloud_vpc_eip
    └── huaweicloud_servicestagev3_environment_associate
```

## Operation Steps

### 1. Script Preparation

Prepare a TF file (such as main.tf) in the specified workspace for writing the current best practice script, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a VPC

Add the following script to the TF file (such as main.tf) to create a VPC:

```hcl
# Create a VPC resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = "192.168.0.0/16"
}
```

**Parameter Description**:
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The CIDR block of the VPC, fixed to `192.168.0.0/16` here

### 3. Create an EIP

Add the following script to the TF file (such as main.tf) to create an EIP:

```hcl
# Create an EIP resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "eip_bandwidth_name" {
  description = "The name of the dedicated bandwidth of the EIP"
  type        = string
}

resource "huaweicloud_vpc_eip" "test" {
  publicip {
    type = "5_bgp"
  }

  bandwidth {
    name        = var.eip_bandwidth_name
    size        = 5
    share_type  = "PER"
    charge_mode = "traffic"
  }
}
```

**Parameter Description**:
- **publicip.type**: The public IP type, set to `5_bgp` (full dynamic BGP) here
- **bandwidth.name**: The dedicated bandwidth name, assigned by referencing the input variable eip_bandwidth_name
- **bandwidth.size**: The bandwidth size in Mbit/s, set to `5` here
- **bandwidth.share_type**: The bandwidth type, where `PER` indicates dedicated bandwidth
- **bandwidth.charge_mode**: The billing mode, where `traffic` indicates pay-per-traffic

### 4. Create a ServiceStage Environment

Add the following script to the TF file (such as main.tf) to create a ServiceStage environment:

```hcl
# Create a ServiceStage environment resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "environment_name" {
  description = "The name of the environment"
  type        = string
}

variable "environment_description" {
  description = "The description of the environment"
  type        = string
  default     = ""
}

resource "huaweicloud_servicestagev3_environment" "test" {
  name        = var.environment_name
  description = var.environment_description
  vpc_id      = huaweicloud_vpc.test.id
}
```

**Parameter Description**:
- **name**: The environment name, assigned by referencing the input variable environment_name; it must contain 2 to 64 characters, only letters, digits, hyphens (-), and underscores (_) are allowed, and it must start with a letter and end with a letter or a digit
- **description**: The environment description, assigned by referencing the input variable environment_description; it must not exceed 128 characters
- **vpc_id**: The ID of the VPC to which the environment belongs, referencing the ID of the VPC created in the previous step; this parameter cannot be updated, and changing it will recreate the environment

### 5. Associate the EIP with the Environment

Add the following script to the TF file (such as main.tf) to associate the EIP with the environment:

```hcl
# Create a ServiceStage environment associate resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_servicestagev3_environment_associate" "test" {
  environment_id = huaweicloud_servicestagev3_environment.test.id

  resources {
    id   = huaweicloud_vpc_eip.test.id
    type = "eip"
  }
}
```

**Parameter Description**:
- **environment_id**: The ID of the environment to which resources are associated, referencing the ID of the environment created in the previous step; this parameter cannot be updated
- **resources.id**: The ID of the resource to be associated, referencing the ID of the EIP created earlier
- **resources.type**: The type of the resource to be associated, set to `eip` here

> Note: The creation, update, and deletion of this associate resource all use the PUT method to manage associated resources in full-set mode. Do not associate other resources to the same environment in the console at the same time, otherwise they may be overwritten or dissociated by Terraform. Destroying this associate resource only dissociates the resource from the environment, and the EIP itself is still managed by its own resource definition.

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through a `tfvars` file, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
vpc_name                = "tf_test_ss_vpc"
eip_bandwidth_name      = "tf_test_ss_eip_bandwidth"
environment_name        = "tf_test_ss_environment"
environment_description = "Created by Terraform for ServiceStage best practice example"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; for other names, you need to add `.auto` before the tfvars name, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the ServiceStage environment with the associated EIP
4. Run `terraform show` to view the created ServiceStage environment with the associated EIP

## Reference Information

- [Huawei Cloud ServiceStage Product Documentation](https://support.huaweicloud.com/servicestage/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For ServiceStage Environment with Associated EIP](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/servicestage/environment-with-associated-eip)
