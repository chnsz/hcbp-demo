# Deploy Basic Auto Scaling Configuration

## Application Scenario

Huawei Cloud Auto Scaling service is a service that automatically adjusts computing resources, capable of automatically adjusting the number of elastic computing instances based on business needs and policies. By configuring AS configuration, you can define templates for elastic scaling instances, including image, specification, security group, and other configurations, providing a foundation for subsequent scaling group creation. This best practice will introduce how to use Terraform to automatically deploy basic AS configuration, including the creation of security groups, key pairs, and AS configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Image Query Data Source (data.huaweicloud_images_image)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_image)
- [Compute Flavors Query Data Source (data.huaweicloud_compute_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)

### Resources

- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Key Pair Resource (huaweicloud_kps_keypair)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kps_keypair)
- [AS Configuration Resource (huaweicloud_as_configuration)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_configuration)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_compute_flavors
        └── huaweicloud_as_configuration

data.huaweicloud_images_image
    └── huaweicloud_as_configuration

huaweicloud_networking_secgroup
    └── huaweicloud_as_configuration

huaweicloud_kps_keypair
    └── huaweicloud_as_configuration
```

## Implementation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones Required for AS Configuration Resource Creation Through Data Source

Add the following script to the TF file (such as main.tf) to inform Terraform to perform a data source query, the query results are used to create AS configuration:

```hcl
# Get all availability zone information in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating AS configuration
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
This data source does not require additional parameters and queries all available availability zone information in the current region by default.

### 3. Query Images Required for AS Configuration Resource Creation Through Data Source

Add the following script to the TF file to inform Terraform to query images that meet the conditions:

```hcl
# Get all image information that meets specific conditions in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating AS configuration
data "huaweicloud_images_image" "test" {
  name        = "Ubuntu 18.04 server 64bit"
  visibility  = "public"
  most_recent = true
}
```

**Parameter Description**:
- **name**: Image name, set to "Ubuntu 18.04 server 64bit"
- **visibility**: Image visibility, set to "public" for public images
- **most_recent**: Whether to use the latest version of the image, set to true to use the latest version

### 4. Query Instance Flavors Required for AS Configuration Resource Creation Through Data Source

Add the following script to the TF file to inform Terraform to query instance flavors that meet the conditions:

```hcl
# Get all instance flavor information that meets specific conditions in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating AS configuration
data "huaweicloud_compute_flavors" "test" {
  availability_zone = data.huaweicloud_availability_zones.test.names[0]
  performance_type  = "normal"
  cpu_core_count    = 2
  memory_size       = 4
}
```

**Parameter Description**:
- **availability_zone**: Availability zone where the instance flavor is located, using the first availability zone from the availability zone list query data source
- **performance_type**: Performance type, set to "normal" for standard type
- **cpu_core_count**: CPU core count, set to 2 cores
- **memory_size**: Memory size (GB), set to 4GB

### 5. Create Security Group Resource

Add the following script to the TF file to inform Terraform to create security group resources:

```hcl
# Create security group resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for deploying AS configuration
resource "huaweicloud_networking_secgroup" "test" {
  name                 = "test-secgroup-demo"
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Security group name, set to "test-secgroup-demo"
- **delete_default_rules**: Whether to delete default rules, set to true to delete default rules

### 6. Create Key Pair Resource

Add the following script to the TF file to inform Terraform to create key pair resources:

```hcl
variable "public_key" {
  description = "The public key for the keypair"
  type        = string
}

# Create key pair resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for deploying AS configuration
resource "huaweicloud_kps_keypair" "acc_key" {
  name       = "test-keypair-demo"
  public_key = var.public_key
}
```

**Parameter Description**:
- **name**: Key pair name, set to "test-keypair-demo"
- **public_key**: Public key content, assigned by referencing input variable public_key

### 7. Create AS Configuration Resource

Add the following script to the TF file to inform Terraform to create AS configuration resources:

```hcl
# Create AS configuration resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing)
resource "huaweicloud_as_configuration" "acc_as_config" {
  scaling_configuration_name = "test-as-configuration-demo"
  instance_config {
    image              = data.huaweicloud_images_image.test.id
    flavor             = data.huaweicloud_compute_flavors.test.ids[0]
    key_name           = huaweicloud_kps_keypair.acc_key.id
    security_group_ids = [huaweicloud_networking_secgroup.test.id]

    metadata = {
      some_key = "some_value"
    }
    user_data = <<EOT
#!/bin/sh
echo "Hello World! The time is now $(date -R)!" | tee /root/output.txt
EOT

    disk {
      size        = 40
      volume_type = "SSD"
      disk_type   = "SYS"
    }

    public_ip {
      eip {
        ip_type = "5_bgp"
        bandwidth {
          size          = 10
          share_type    = "PER"
          charging_mode = "traffic"
        }
      }
    }
  }
}
```

**Parameter Description**:
- **scaling_configuration_name**: AS configuration name, set to "test-as-configuration-demo"
- **instance_config**: Instance configuration block
  - **image**: Image ID, using the ID from the image query data source
  - **flavor**: Instance flavor, using the first flavor ID from the instance flavor query data source
  - **key_name**: Key pair ID, referencing the ID of the key pair resource created earlier
  - **security_group_ids**: Security group ID list, referencing the ID of the security group resource created earlier
  - **metadata**: Metadata configuration, setting key-value pairs
  - **user_data**: Instance startup script, used for initializing instances
  - **disk**: Disk configuration block
    - **size**: Disk size (GB), set to 40GB
    - **volume_type**: Disk type, set to "SSD"
    - **disk_type**: Disk purpose, set to "SYS" for system disk
  - **public_ip**: Public IP configuration block
    - **eip**: Elastic public IP configuration
      - **ip_type**: Public IP type, set to "5_bgp"
      - **bandwidth**: Bandwidth configuration block
        - **size**: Bandwidth size, set to 10Mbps
        - **share_type**: Bandwidth type, set to "PER" for dedicated
        - **charging_mode**: Billing mode, set to "traffic" for traffic-based billing

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Key pair configuration
public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC..."
```

**Usage Method**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content in this `tfvars` file when executing terraform commands, other names need to add `.auto` definition before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="public_key=your-public-key"`
2. Environment variables: `export TF_VAR_public_key=your-public-key`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values in the following priority: command line parameters > variable files > environment variables > default values.

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating AS configuration
4. Run `terraform show` to view the details of the created AS configuration

## Reference Information

- [Huawei Cloud Auto Scaling Product Documentation](https://support.huaweicloud.com/as/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [AS Basic Configuration Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/as)
