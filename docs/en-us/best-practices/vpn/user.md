# Deploy User

## Application Scenario

Virtual Private Network (VPN) is a secure and reliable network connection service provided by Huawei Cloud, supporting the establishment of encrypted IPsec VPN connections between VPCs and local networks, enabling interconnection between cloud resources and local data centers. VPN user is an important function of VPN service, used to create and manage user accounts on VPN servers, achieving user authentication for point-to-site VPN connections. By creating VPN users, independent accounts and passwords can be assigned to different users, achieving user-level access control and permission management. This best practice introduces how to use Terraform to automatically deploy VPN users, including VPN user creation and configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [VPN User Resource (huaweicloud_vpn_user)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpn_user)

### Resource/Data Source Dependencies

```text
huaweicloud_vpn_user
```

> Note: VPN user needs to be associated with a VPN server. Before creating a VPN user, ensure that the VPN server has been created.

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Create VPN User

Add the following script to the TF file (such as main.tf) to create VPN user:

```hcl
# Create VPN user resource
resource "huaweicloud_vpn_user" "test" {
  vpn_server_id = var.vpn_user_server_id
  name          = var.vpn_user_name
  password      = var.vpn_user_password
}
```

**Parameter Description**:
- **vpn_server_id**: VPN server ID, assigned by referencing the input variable `vpn_user_server_id`, used to specify the VPN server to which the VPN user belongs
- **name**: VPN user name, assigned by referencing the input variable `vpn_user_name`
- **password**: VPN user password, assigned by referencing the input variable `vpn_user_password`, used for user authentication

> Note: VPN user is used for user authentication in point-to-site VPN connections. After creating a VPN user, users can use this account and password to connect to the VPN server, achieving secure remote access.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# VPN user configuration (Required)
vpn_user_server_id = "your-vpn-server-id"
vpn_user_name      = "tf_test_vpn_user"
vpn_user_password  = "your-secure-password"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="vpn_user_server_id=your-vpn-server-id" -var="vpn_user_name=tf_test_vpn_user"`
2. Environment variables: `export TF_VAR_vpn_user_server_id=your-vpn-server-id`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating VPN user
4. Run `terraform show` to view the created VPN user

## Reference Information

- [Huawei Cloud VPN Product Documentation](https://support.huaweicloud.com/intl/en-us/vpn/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For VPN User](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vpn/user)
