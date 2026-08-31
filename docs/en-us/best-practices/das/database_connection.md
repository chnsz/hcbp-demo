# Deploy Database Connection

## Application Scenario

Data Admin Service (DAS) is a Web service for logging in to and operating databases on Huawei Cloud. It provides a one-stop cloud database management platform for database development, O&M, and intelligent diagnosis. By creating a database instance connection, you can securely access existing database instances such as RDS in DAS and perform visualized SQL operations and O&M. Combined with database user creation and connection sharing, multiple users can collaboratively access the same connection resource. This best practice introduces how to use Terraform to automatically deploy a DAS database instance connection, including creating a database instance connection, a database user, and sharing the connection with another IAM user.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [Database Instance Connection Resource (huaweicloud_das_database_instance_connection)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_database_instance_connection)
- [Database User Resource (huaweicloud_das_database_user)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_database_user)
- [Shared Connection Resource (huaweicloud_das_shared_connection)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_shared_connection)

### Resource/Data Source Dependencies

```text
huaweicloud_das_database_instance_connection
    └── huaweicloud_das_shared_connection

huaweicloud_das_database_user
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the configuration introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create Database Instance Connection

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a database instance connection resource:

```hcl
variable "connection_instance_id" {
  description = "The ID of the RDS instance to connect"
  type        = string
}

variable "connection_engine_type" {
  description = "The engine type of the database instance"
  type        = string
}

variable "connection_network_type" {
  description = "The network type of the database instance connection"
  type        = string
}

variable "connection_username" {
  description = "The username for the database instance connection"
  type        = string
}

variable "connection_password" {
  description = "The password for the database instance connection"
  type        = string
  sensitive   = true
}

variable "connection_is_save_password" {
  description = "Whether to save the password for the database instance connection"
  type        = bool
  default     = true
  sensitive   = true
}

variable "connection_port" {
  description = "The port of the database instance connection"
  type        = number
  default     = null
}

variable "connection_database_name" {
  description = "The database name of the database instance connection"
  type        = string
  default     = null
}

variable "connection_sql_record_flag" {
  description = "Whether SQL recording is enabled for the database instance connection"
  type        = bool
  default     = null
}

variable "connection_description" {
  description = "The description of the database instance connection"
  type        = string
  default     = null
}

variable "connection_node_ids" {
  description = "The unique identifiers of the instance nodes"
  type        = list(string)
  default     = []
  nullable    = false
}

# Create a database instance connection resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_das_database_instance_connection" "test" {
  instance_id      = var.connection_instance_id
  engine_type      = var.connection_engine_type
  network_type     = var.connection_network_type
  username         = var.connection_username
  password         = var.connection_password
  is_save_password = var.connection_is_save_password

  port            = var.connection_port
  database_name   = var.connection_database_name
  sql_record_flag = var.connection_sql_record_flag
  description     = var.connection_description
  node_ids        = var.connection_node_ids
}
```

**Parameter Description**:
- **instance_id**: ID of the RDS instance to connect, assigned by referencing the input variable `connection_instance_id`
- **engine_type**: Engine type of the database instance, assigned by referencing the input variable `connection_engine_type`
- **network_type**: Network type of the database instance connection, assigned by referencing the input variable `connection_network_type`
- **username**: Username for the database instance connection, assigned by referencing the input variable `connection_username`
- **password**: Password for the database instance connection, assigned by referencing the input variable `connection_password`
- **is_save_password**: Whether to save the password for the database instance connection, assigned by referencing the input variable `connection_is_save_password`
- **port**: Port of the database instance connection, assigned by referencing the input variable `connection_port`
- **database_name**: Database name of the database instance connection, assigned by referencing the input variable `connection_database_name`
- **sql_record_flag**: Whether SQL recording is enabled for the database instance connection, assigned by referencing the input variable `connection_sql_record_flag`
- **description**: Description of the database instance connection, assigned by referencing the input variable `connection_description`
- **node_ids**: Unique identifiers of the instance nodes, assigned by referencing the input variable `connection_node_ids`

> Note: An existing connectable database instance such as RDS is required before deployment. Keep sensitive information such as database passwords secure and never commit them to version control.

### 3. Create Database User

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a database user resource:

```hcl
variable "db_user_name" {
  description = "The name of the database user"
  type        = string
}

variable "db_user_password" {
  description = "The password of the database user"
  type        = string
  sensitive   = true
}

# Create a database user resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_das_database_user" "test" {
  instance_id = var.connection_instance_id
  name        = var.db_user_name
  password    = var.db_user_password
}
```

**Parameter Description**:
- **instance_id**: ID of the RDS instance that the database user belongs to, assigned by referencing the input variable `connection_instance_id`
- **name**: Name of the database user, assigned by referencing the input variable `db_user_name`
- **password**: Password of the database user, assigned by referencing the input variable `db_user_password`

### 4. Create Shared Connection

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a shared connection resource:

```hcl
variable "shared_user_id" {
  description = "The IAM user ID to share the connection with"
  type        = string
}

variable "shared_user_name" {
  description = "The IAM user name to share the connection with"
  type        = string
}

variable "shared_expired_at" {
  description = "The expiration time of the shared connection, in RFC3339 format"
  type        = string
  default     = null
}

# Create a shared connection resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_das_shared_connection" "test" {
  connection_id = huaweicloud_das_database_instance_connection.test.id
  user_id       = var.shared_user_id
  user_name     = var.shared_user_name
  expired_at    = var.shared_expired_at
}
```

**Parameter Description**:
- **connection_id**: ID of the database instance connection to share, assigned by referencing the ID of the database instance connection resource
- **user_id**: IAM user ID to share the connection with, assigned by referencing the input variable `shared_user_id`
- **user_name**: IAM user name to share the connection with, assigned by referencing the input variable `shared_user_name`
- **expired_at**: Expiration time of the shared connection in RFC3339 format, assigned by referencing the input variable `shared_expired_at`

> Note: The shared connection depends on an existing database instance connection. Ensure the target IAM user information is correct and configure the sharing expiration time as needed.

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables for configuration values. These input parameters need to be entered manually during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through a `tfvars` file, which can avoid repeated input each time it is executed.

Create a `terraform.tfvars` file in the working directory. Example content is as follows:

```hcl
# Database instance connection configuration
connection_instance_id      = "tf_test_das_instance_id"
connection_engine_type      = "mysql"
connection_network_type     = "rds"
connection_username         = "tf_test_username"
connection_password         = "tf_test_password"
connection_is_save_password = true
connection_port             = 3306
connection_database_name    = "tf_test_database"
connection_sql_record_flag  = true
connection_description      = "tf_test_connection_description"
connection_node_ids         = []

# Database user configuration
db_user_name     = "tf_test_db_user"
db_user_password = "tf_test_db_password"

# Shared connection configuration
shared_user_id    = "tf_test_shared_user_id"
shared_user_name  = "tf_test_shared_user_name"
shared_expired_at = null
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing Terraform commands; for other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command-line parameters: `terraform apply -var="connection_instance_id=your_rds_instance_id" -var="connection_username=your_username"`
2. Environment variables: `export TF_VAR_connection_instance_id=your_rds_instance_id`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command-line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, perform the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the database connection
4. Run `terraform show` to view the created database connection

## Reference Information

- [Huawei Cloud DAS Product Documentation](https://support.huaweicloud.com/das/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DAS Database Connection Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/das/database-connection)
