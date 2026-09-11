# Deploy DIS Stream

## Application Scenario

Data Ingestion Service (DIS) is a real-time data ingestion service provided by Huawei Cloud, used to collect and transmit massive amounts of data to the cloud in real time, supporting the ingestion of various data sources and data types. A DIS stream is the basic unit of data ingestion, responsible for carrying data writes and reads, and can meet different throughput and reliability requirements through parameters such as the number of partitions and the data retention period.

This best practice will introduce how to use Terraform to automatically deploy a DIS stream, including basic stream configuration, auto scaling partition configuration, data format and compression configuration, and tag management.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [DIS Stream (huaweicloud_dis_stream)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dis_stream)

### Resource/Data Source Dependencies

```
huaweicloud_dis_stream
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create a DIS Stream

Add the following script to the TF file (such as main.tf) to create a DIS stream:

```hcl
# Create a DIS stream resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "stream_name" {
  description = "The name of the DIS stream"
  type        = string
}

variable "stream_partition_count" {
  description = "The number of partitions for the DIS stream"
  type        = number
}

variable "stream_type" {
  description = "The type of the DIS stream"
  type        = string
  default     = null
}

variable "stream_retention_period" {
  description = "The data retention period in hours"
  type        = number
  default     = 24
}

variable "stream_auto_scale_min_partition_count" {
  description = "The minimum number of partitions for auto scaling"
  type        = number
  default     = null
}

variable "stream_auto_scale_max_partition_count" {
  description = "The maximum number of partitions for auto scaling"
  type        = number
  default     = null
}

variable "stream_compression_format" {
  description = "The compression format of the data"
  type        = string
  default     = null
}

variable "stream_data_type" {
  description = "The type of the data"
  type        = string
  default     = null
}

variable "stream_csv_delimiter" {
  description = "The delimiter for CSV data"
  type        = string
  default     = null
}

variable "stream_data_schema" {
  description = "The schema of the data"
  type        = string
  default     = null
}

variable "stream_tags" {
  description = "The key/value pairs to associate with the DIS stream"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_dis_stream" "test" {
  stream_name      = var.stream_name
  partition_count  = var.stream_partition_count
  stream_type      = var.stream_type
  retention_period = var.stream_retention_period

  auto_scale_min_partition_count = var.stream_auto_scale_min_partition_count
  auto_scale_max_partition_count = var.stream_auto_scale_max_partition_count

  compression_format = var.stream_compression_format
  data_type          = var.stream_data_type
  csv_delimiter      = var.stream_csv_delimiter
  data_schema        = var.stream_data_schema

  tags = var.stream_tags
}
```

**Parameter Description**:
- **stream_name**: Assigned by referencing the input variable stream_name, used to specify the name of the DIS stream
- **partition_count**: Assigned by referencing the input variable stream_partition_count, used to specify the number of partitions for the DIS stream
- **stream_type**: Assigned by referencing the input variable stream_type, used to specify the type of the DIS stream. Possible values are **COMMON** (normal stream) and **ADVANCED** (advanced stream)
- **retention_period**: Assigned by referencing the input variable stream_retention_period, used to specify the data retention period in hours. The value ranges from 24 to 72
- **auto_scale_min_partition_count**: Assigned by referencing the input variable stream_auto_scale_min_partition_count, used to specify the minimum number of partitions for auto scaling
- **auto_scale_max_partition_count**: Assigned by referencing the input variable stream_auto_scale_max_partition_count, used to specify the maximum number of partitions for auto scaling
- **compression_format**: Assigned by referencing the input variable stream_compression_format, used to specify the compression format of the data. Possible values are **zip**, **gzip**, **snappy**, **lz4**, and **zstd**
- **data_type**: Assigned by referencing the input variable stream_data_type, used to specify the type of the data. Possible values are **CSV**, **JSON**, and **BLOB**
- **csv_delimiter**: Assigned by referencing the input variable stream_csv_delimiter, used to specify the delimiter for CSV data
- **data_schema**: Assigned by referencing the input variable stream_data_schema, used to specify the schema of the data in JSON format
- **tags**: Assigned by referencing the input variable stream_tags, used to specify the key/value pairs to associate with the DIS stream

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# DIS stream configuration
stream_name                           = "tf_test_dis_stream"
stream_partition_count                = 2
stream_auto_scale_min_partition_count = 2
stream_auto_scale_max_partition_count = 4
stream_type                           = "COMMON"
stream_compression_format             = "zip"
stream_data_type                      = "CSV"
stream_csv_delimiter                  = ";"
stream_data_schema                    = "{\"type\":\"record\",\"name\":\"RecordName\",\"fields\":[{\"type\":\"string\",\"name\":\"name\"}]}"
stream_tags = {
  foo = "bar"
  key = "value"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="stream_name=my-stream"`
2. Environment variables: `export TF_VAR_stream_name=my-stream`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DIS stream
4. Run `terraform show` to view the created DIS stream

## Reference Information

- [Huawei Cloud Data Ingestion Service Product Documentation](https://support.huaweicloud.com/dis/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DIS Stream](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dis/stream)
