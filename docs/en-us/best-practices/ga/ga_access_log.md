# Deploy GA Access Log

## Application Scenario

Global Accelerator (GA) is a global network acceleration service provided by Huawei Cloud. It accesses user traffic from globally distributed points of presence and forwards the traffic to origin servers over the Huawei Cloud backbone network, effectively reducing cross-region access latency and improving access experience. Access logs record the access request information of listeners, which is an important basis for troubleshooting access exceptions, analyzing traffic characteristics, and meeting security audit requirements.

This best practice will introduce how to use Terraform to automatically deploy a GA access log, including creating a GA accelerator, a listener, an LTS log group and log stream, and delivering the listener access logs to LTS for unified storage and analysis.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Global Accelerator (huaweicloud_ga_accelerator)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ga_accelerator)
- [Global Accelerator Listener (huaweicloud_ga_listener)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ga_listener)
- [LTS Log Group (huaweicloud_lts_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [LTS Log Stream (huaweicloud_lts_stream)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [Global Accelerator Access Log (huaweicloud_ga_access_log)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ga_access_log)

### Resource/Data Source Dependencies

```
huaweicloud_ga_accelerator
    └── huaweicloud_ga_listener
            └── huaweicloud_ga_access_log

huaweicloud_lts_group
    ├── huaweicloud_lts_stream
    │       └── huaweicloud_ga_access_log
    └── huaweicloud_ga_access_log
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Global Accelerator

Add the following script to the TF file (such as main.tf) to create a global accelerator:

```hcl
# Create a global accelerator resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "accelerator_name" {
  description = "The name of the GA accelerator"
  type        = string
}

variable "accelerator_description" {
  description = "The description of the GA accelerator"
  type        = string
  default     = ""
}

variable "ip_area" {
  description = "The area of the IP address. Valid values: CM, CT, CU, EU, AP, AF, ME, GE"
  type        = string
  default     = "CM"
}

variable "tags" {
  description = "The tags of the GA accelerator and listener"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_ga_accelerator" "test" {
  name        = var.accelerator_name
  description = var.accelerator_description

  ip_sets {
    ip_type = "IPV4"
    area    = var.ip_area
  }

  ip_sets {
    ip_type = "IPV6"
    area    = var.ip_area
  }

  tags = var.tags
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable accelerator_name, used to specify the name of the global accelerator
- **description**: Assigned by referencing the input variable accelerator_description, used to specify the description of the global accelerator
- **ip_sets.ip_type**: The IP address type. This best practice configures both IPV4 and IPV6 IP sets
- **ip_sets.area**: Assigned by referencing the input variable ip_area, used to specify the area of the IP address
- **tags**: Assigned by referencing the input variable tags, used to add tags to the global accelerator

### 3. Create a Global Accelerator Listener

Add the following script to the TF file (such as main.tf) to create a global accelerator listener:

```hcl
# Create a global accelerator listener resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "listener_name" {
  description = "The name of the GA listener"
  type        = string
}

variable "listener_protocol" {
  description = "The protocol of the GA listener. Valid values: TCP, UDP"
  type        = string
  default     = "TCP"
}

variable "listener_description" {
  description = "The description of the GA listener"
  type        = string
  default     = ""
}

variable "port_from" {
  description = "The start port of the listener port range"
  type        = number
  default     = 4000
}

variable "port_to" {
  description = "The end port of the listener port range"
  type        = number
  default     = 4200
}

resource "huaweicloud_ga_listener" "test" {
  accelerator_id = huaweicloud_ga_accelerator.test.id
  name           = var.listener_name
  protocol       = var.listener_protocol
  description    = var.listener_description

  port_ranges {
    from_port = var.port_from
    to_port   = var.port_to
  }

  tags = var.tags
}
```

**Parameter Description**:
- **accelerator_id**: Assigned by referencing the ID of the global accelerator resource huaweicloud_ga_accelerator.test, used to associate the listener with its global accelerator
- **name**: Assigned by referencing the input variable listener_name, used to specify the name of the listener
- **protocol**: Assigned by referencing the input variable listener_protocol, used to specify the protocol of the listener
- **description**: Assigned by referencing the input variable listener_description, used to specify the description of the listener
- **port_ranges.from_port**: Assigned by referencing the input variable port_from, used to specify the start port of the listener port range
- **port_ranges.to_port**: Assigned by referencing the input variable port_to, used to specify the end port of the listener port range
- **tags**: Assigned by referencing the input variable tags, used to add tags to the listener

### 4. Create an LTS Log Group

Add the following script to the TF file (such as main.tf) to create an LTS log group:

```hcl
# Create an LTS log group resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "lts_group_name" {
  description = "The name of the LTS log group"
  type        = string
}

variable "lts_ttl_in_days" {
  description = "The TTL in days for the LTS log group"
  type        = number
  default     = 30
}

resource "huaweicloud_lts_group" "test" {
  group_name  = var.lts_group_name
  ttl_in_days = var.lts_ttl_in_days
}
```

**Parameter Description**:
- **group_name**: Assigned by referencing the input variable lts_group_name, used to specify the name of the log group
- **ttl_in_days**: Assigned by referencing the input variable lts_ttl_in_days, used to specify the retention period of logs in days

### 5. Create an LTS Log Stream

Add the following script to the TF file (such as main.tf) to create an LTS log stream:

```hcl
# Create an LTS log stream resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "lts_stream_name" {
  description = "The name of the LTS log stream"
  type        = string
}

resource "huaweicloud_lts_stream" "test" {
  group_id    = huaweicloud_lts_group.test.id
  stream_name = var.lts_stream_name
}
```

**Parameter Description**:
- **group_id**: Assigned by referencing the ID of the LTS log group resource huaweicloud_lts_group.test, used to associate the log stream with its log group
- **stream_name**: Assigned by referencing the input variable lts_stream_name, used to specify the name of the log stream

### 6. Create a GA Access Log

Add the following script to the TF file (such as main.tf) to create a GA access log:

```hcl
# Create a GA access log resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_ga_access_log" "test" {
  resource_type = "LISTENER"
  resource_id   = huaweicloud_ga_listener.test.id
  log_group_id  = huaweicloud_lts_group.test.id
  log_stream_id = huaweicloud_lts_stream.test.id
}
```

**Parameter Description**:
- **resource_type**: The type of the resource associated with the log. Currently, only `LISTENER` is supported
- **resource_id**: Assigned by referencing the ID of the global accelerator listener resource huaweicloud_ga_listener.test, used to specify the listener whose access logs are collected
- **log_group_id**: Assigned by referencing the ID of the LTS log group resource huaweicloud_lts_group.test, used to specify the target log group for access log delivery
- **log_stream_id**: Assigned by referencing the ID of the LTS log stream resource huaweicloud_lts_stream.test, used to specify the target log stream for access log delivery

### 7. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through the `tfvars` file, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "<YOUR_ACCESS_KEY>"
secret_key  = "<YOUR_SECRET_KEY>"

# GA accelerator and listener configuration
accelerator_name = "ga-accelerator-test"
listener_name    = "ga-listener-test"

# LTS configuration
lts_group_name  = "ga-lts-group"
lts_stream_name = "ga-lts-stream"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="accelerator_name=my-accelerator"`
2. Environment variables: `export TF_VAR_accelerator_name=my-accelerator`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 8. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GA access log
4. Run `terraform show` to view the created GA access log

## Reference Information

- [Huawei Cloud Global Accelerator Product Documentation](https://support.huaweicloud.com/ga/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GA Access Log](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ga/ga-access-log)
