# Deploy VIP and Associate Instances

## Application Scenario

Virtual IP (VIP) is a network technology in Huawei Cloud VPC used to achieve high availability, which can bind a virtual IP address to multiple ECS instances, implementing load balancing and failover. Through VIP association, high availability deployment of business can be achieved. When an ECS instance fails, traffic can automatically switch to other healthy instances. This best practice will introduce how to use Terraform to automatically deploy VIP associations, including creating ECS instances, VIP resources, and their association configurations.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zone List Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS Flavor List Query Data Source (data.huaweicloud_compute_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [IMS Image List Query Data Source (data.huaweicloud_images_images)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule Resource (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [ECS Instance Resource (huaweicloud_compute_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [VIP Resource (huaweicloud_networking_vip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_vip)
- [VIP Association Resource (huaweicloud_networking_vip_associate)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_vip_associate)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    ├── data.huaweicloud_compute_flavors
    │   └── data.huaweicloud_images_images
    │       └── huaweicloud_compute_instance
    └── huaweicloud_compute_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_compute_instance
        └── huaweicloud_networking_vip
            └── huaweicloud_networking_vip_associate

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_compute_instance

huaweicloud_compute_instance
    └── huaweicloud_networking_vip_associate
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Prerequisite Resource Preparation

This best practice requires creating prerequisite resources such as VPC, subnets, security groups, and ECS instances first. Please follow the following steps in the "Deploy Basic Elastic Cloud Server" best practice for preparation:

- **Step 2**: Query availability zones required for ECS instance resource creation through data sources
- **Step 3**: Query flavors required for ECS instance resource creation through data sources
- **Step 4**: Query images required for ECS instance resource creation through data sources
- **Step 5**: Create VPC resource
- **Step 6**: Create VPC subnet resource
- **Step 7**: Create security group resource
- **Step 8**: Create ECS instance

After completing the above steps, continue with the subsequent steps of this best practice.

### 3. Create VIP Resource

Add the following script to the TF file to instruct Terraform to create a VIP resource:

```hcl
# Create VIP resource
resource "huaweicloud_networking_vip" "test" {
  network_id = huaweicloud_vpc_subnet.test.id
}
```

**Parameter Description**:
- **network_id**: Network ID that the VIP belongs to, referencing the ID of the previously created subnet resource

### 4. Create VIP Association Resource

Add the following script to the TF file to instruct Terraform to create a VIP association resource:

```hcl
# Create VIP association resource
resource "huaweicloud_networking_vip_associate" "test" {
  vip_id   = huaweicloud_networking_vip.test.id
  port_ids = try([
    huaweicloud_compute_instance.test.network[0].port
  ], [])
}
```

**Parameter Description**:
- **vip_id**: VIP ID, referencing the ID of the previously created VIP resource
- **port_ids**: List of port IDs to associate, using try function to get the network port ID of the ECS instance, using empty list if retrieval fails

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

# Security group configuration
security_group_name = "tf_test_security_group"

# ECS instance configuration
instance_name          = "tf_test_instance"
administrator_password = "YourPasswordInput!"
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
3. After confirming the resource plan is correct, run `terraform apply` to start creating VIP associations
4. Run `terraform show` to view the created VIP association details

## Reference Information

- [Huawei Cloud VPC Product Documentation](https://support.huaweicloud.com/vpc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [VPC VIP Association Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vpc)
