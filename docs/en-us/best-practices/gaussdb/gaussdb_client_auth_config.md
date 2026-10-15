# Deploy Client Access Authentication Configuration

## Application Scenario

GaussDB is a high-performance, highly available, and highly secure enterprise-grade distributed relational database service provided by Huawei Cloud, supporting fine-grained control over database access sources through client access authentication (HBA, Host-Based Authentication) configuration. The client access authentication configuration defines the client address ranges allowed to connect to the database, the matched databases and users, and the authentication method used, making it an important means of ensuring database access security.

This best practice will introduce how to use Terraform to automatically deploy a GaussDB client access authentication configuration, including specifying the GaussDB instance, configuring the client connection type, the matched database and user, the IP address range, and the authentication method.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [GaussDB Client Access Authentication Configuration (huaweicloud_gaussdb_client_auth_config)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_client_auth_config)

### Resource/Data Source Dependencies

```
huaweicloud_gaussdb_client_auth_config
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) for configuration details.

### 2. Create a GaussDB Client Access Authentication Configuration

Add the following script to the TF file (such as main.tf) to create a GaussDB client access authentication configuration:

```hcl
# Create a GaussDB client access authentication configuration resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_id" {
  description = "The ID of the GaussDB instance"
  type        = string
  default     = ""
}

variable "config_type" {
  description = "The client connection type. Valid values are host, hostssl, and hostnossl"
  type        = string
  default     = "host"
}

variable "config_database" {
  description = "The database name that the record matches. The value can be all or an existing database name"
  type        = string
  default     = "all"
}

variable "config_user" {
  description = "The database user that the record matches. The value can be all or an existing username"
  type        = string
  default     = "root"
}

variable "config_address" {
  description = "The IP address range that the record matches. The value must be in CIDR format"
  type        = string
  default     = "10.10.0.0/16"
}

variable "config_method" {
  description = "The authentication method. Valid values include md5, sha256, sm3, reject, cert, etc"
  type        = string
  default     = "md5"
}

resource "huaweicloud_gaussdb_client_auth_config" "test" {
  instance_id = var.instance_id
  type        = var.config_type
  database    = var.config_database
  user        = var.config_user
  address     = var.config_address
  method      = var.config_method
}
```

**Parameter Description**:

- **instance_id**: The GaussDB instance ID, assigned by referencing the input variable instance_id
- **type**: The client connection type, assigned by referencing the input variable config_type. Valid values are `host` (allows both SSL and non-SSL connections), `hostssl` (only allows SSL connections), and `hostnossl` (only allows non-SSL connections)
- **database**: The database name that the record matches, assigned by referencing the input variable config_database. The value can be `all` or an existing database name
- **user**: The database user that the record matches, assigned by referencing the input variable config_user. The value can be `all` or an existing username
- **address**: The IP address range that the record matches, assigned by referencing the input variable config_address. The value must be in CIDR format (e.g., `10.10.0.0/16`)
- **method**: The authentication method, assigned by referencing the input variable config_method. Valid values include `md5`, `sha256`, `sm3`, `reject`, `cert`, etc.

> Note: `instance_id`, `type`, `database`, `user`, and `address` cannot be modified after creation; changing them will create a new resource. Only the `method` field can be updated in-place after creation.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Fill in according to the script variables; use placeholders for sensitive information
instance_id     = "your_instance_id"
config_type     = "host"
config_database = "all"
config_user     = "root"
config_address  = "10.10.0.0/16"
config_method   = "md5"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="instance_id=your_instance_id"`
2. Environment variables: `export TF_VAR_instance_id=your_instance_id`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GaussDB client access authentication configuration
4. Run `terraform show` to view the created GaussDB client access authentication configuration

## Reference Information

- [Huawei Cloud GaussDB Product Documentation](https://support.huaweicloud.com/gaussdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GaussDB Client Access Authentication Configuration](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/gaussdb-client-auth-config)
