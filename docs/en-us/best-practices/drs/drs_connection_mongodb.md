# Deploy MongoDB Sharding Connection

## Application Scenario

Data Replication Service (DRS) is a one-stop data replication service provided by Huawei Cloud, supporting real-time synchronization and migration between multiple database engines. When migrating or synchronizing a self-built MongoDB sharded cluster to Huawei Cloud, you need to create a source database connection in DRS first to describe the access information of the source database.

This best practice will introduce how to use Terraform to automatically deploy a DRS connection for accessing a self-built MongoDB sharded cluster. The connection contains the access information of the primary node and multiple shard nodes, and configures the SSL connection mode and driver name, providing the source connection foundation for subsequent data synchronization tasks.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [DRS Connection (huaweicloud_drs_connection)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_connection)

### Resource/Data Source Dependencies

```
huaweicloud_drs_connection
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a DRS Connection

Add the following script in the TF file (such as main.tf) to create a DRS connection:

```hcl
# Create a DRS connection resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "connection_name" {
  description = "The DRS connection name"
  type        = string
}

variable "description" {
  description = "The description of the DRS connection"
  type        = string
  default     = ""
}

variable "endpoint_ip" {
  description = "The IP address and port of the primary MongoDB database, e.g. 192.168.0.1:8080"
  type        = string
}

variable "db_user" {
  description = "The database username"
  type        = string
  default     = "mog"
}

variable "db_password" {
  description = "The password for the MongoDB database user"
  type        = string
  sensitive   = true
}

variable "db_name" {
  description = "The database name"
  type        = string
  default     = "root"
}

variable "shard1_ip" {
  description = "The IP address and port of the first MongoDB shard, e.g. 192.168.0.1:8000"
  type        = string
}

variable "shard2_ip" {
  description = "The IP address and port of the second MongoDB shard, e.g. 192.168.0.2:8000"
  type        = string
}

variable "driver_name" {
  description = "The driver name of the connection configuration"
  type        = string
  default     = "mongodb"
}

resource "huaweicloud_drs_connection" "test" {
  name        = var.connection_name
  db_type     = "mongodb"
  description = var.description

  endpoint {
    endpoint_name = "mongodb"
    ip            = var.endpoint_ip
    db_user       = var.db_user
    db_password   = var.db_password
    db_name       = var.db_name

    source_sharding {
      endpoint_name = "mongodb"
      ip            = var.shard1_ip
      db_user       = var.db_user
      db_password   = var.db_password
      db_name       = var.db_name
    }

    source_sharding {
      endpoint_name = "mongodb"
      ip            = var.shard2_ip
      db_user       = var.db_user
      db_password   = var.db_password
      db_name       = var.db_name
    }
  }

  ssl {
    ssl_link = false
  }

  config {
    driver_name = var.driver_name
  }

  lifecycle {
    ignore_changes = [
      endpoint.0.db_password,
      endpoint.0.source_sharding.0.db_password,
      endpoint.0.source_sharding.0.endpoint_name,
      endpoint.0.source_sharding.1.db_password,
      endpoint.0.source_sharding.1.endpoint_name,
    ]
  }
}
```

**Parameter Description**:
- **name**: The connection name, assigned by referencing the input variable connection_name
- **db_type**: The database type, fixed to mongodb
- **description**: The connection description, assigned by referencing the input variable description
- **endpoint.endpoint_name**: The endpoint name of the source database, fixed to mongodb
- **endpoint.ip**: The IP address and port of the primary database, assigned by referencing the input variable endpoint_ip
- **endpoint.db_user**: The database username, assigned by referencing the input variable db_user
- **endpoint.db_password**: The database password, assigned by referencing the input variable db_password
- **endpoint.db_name**: The database name, assigned by referencing the input variable db_name
- **endpoint.source_sharding.ip**: The IP address and port of the shard database, assigned by referencing the input variables shard1_ip and shard2_ip respectively
- **ssl.ssl_link**: Whether to enable the SSL connection, set to false here
- **config.driver_name**: The driver name of the connection configuration, assigned by referencing the input variable driver_name
- **lifecycle.ignore_changes**: Ignores the password and shard endpoint name fields that are not returned by the API, avoiding continuous changes caused by the API not echoing them

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
connection_name = "your_drs_mongodb_connection"
db_password     = "Test@123456"
endpoint_ip     = "192.168.0.1:8080"
shard1_ip       = "192.168.0.1:8000"
shard2_ip       = "192.168.0.2:8000"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of the `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="connection_name=your_drs_mongodb_connection"`
2. Environment variables: `export TF_VAR_connection_name=your_drs_mongodb_connection`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DRS connection
4. Run `terraform show` to view the created DRS connection

## Reference Information

- [Huawei Cloud Data Replication Service Product Documentation](https://support.huaweicloud.com/drs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DRS MongoDB Sharding Connection](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/drs/drs-connection-mongodb)
