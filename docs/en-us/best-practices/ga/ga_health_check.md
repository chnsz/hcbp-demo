# Deploy GA Health Check

## Application Scenario

Global Accelerator (GA) is a global network acceleration service provided by Huawei Cloud. By bringing user traffic into the Huawei Cloud backbone network from the nearest point of presence and forwarding it to origin servers, GA provides nearby access and global acceleration capabilities. In acceleration scenarios, endpoints in an endpoint group may become unavailable due to network jitter or service exceptions. A health check periodically probes the availability of endpoints, automatically isolates unhealthy endpoints, and ensures the stability of the acceleration path.

This best practice will introduce how to use Terraform to automatically deploy a GA health check, including accelerator creation, listener configuration, endpoint group creation, and health check configuration, helping you quickly build a global acceleration service with health probing capabilities.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Global Accelerator (huaweicloud_ga_accelerator)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ga_accelerator)
- [Global Accelerator Listener (huaweicloud_ga_listener)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ga_listener)
- [Global Accelerator Endpoint Group (huaweicloud_ga_endpoint_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ga_endpoint_group)
- [Global Accelerator Health Check (huaweicloud_ga_health_check)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ga_health_check)

### Resource/Data Source Dependencies

```
huaweicloud_ga_accelerator
    └── huaweicloud_ga_listener
            └── huaweicloud_ga_endpoint_group
                    └── huaweicloud_ga_health_check
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create a Global Accelerator

Add the following script in the TF file (such as main.tf):

```hcl
# Create a global accelerator resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "accelerator_name" {
  description = "The name of the GA accelerator"
  type        = string
}

variable "accelerator_description" {
  description = "The description of the GA accelerator"
  type        = string
  default     = "GA accelerator for health check"
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
- **name**: The name of the global accelerator, assigned by referencing the input variable accelerator_name
- **description**: The description of the global accelerator, assigned by referencing the input variable accelerator_description
- **ip_sets.ip_type**: The IP address type, configured as IPV4 and IPV6 respectively to implement dual-stack access
- **ip_sets.area**: The area of the IP address, assigned by referencing the input variable ip_area
- **tags**: The tags of the global accelerator, assigned by referencing the input variable tags

### 3. Create a Global Accelerator Listener

Add the following script in the TF file (such as main.tf):

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
  default     = "GA listener for health check"
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
- **accelerator_id**: The ID of the global accelerator to which the listener belongs, assigned by referencing the ID of the global accelerator resource created in the previous step
- **name**: The name of the listener, assigned by referencing the input variable listener_name
- **protocol**: The protocol of the listener, assigned by referencing the input variable listener_protocol
- **description**: The description of the listener, assigned by referencing the input variable listener_description
- **port_ranges.from_port**: The start port of the listener port range, assigned by referencing the input variable port_from
- **port_ranges.to_port**: The end port of the listener port range, assigned by referencing the input variable port_to
- **tags**: The tags of the listener, assigned by referencing the input variable tags

### 4. Create a Global Accelerator Endpoint Group

Add the following script in the TF file (such as main.tf):

```hcl
# Create a global accelerator endpoint group resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "endpoint_group_name" {
  description = "The name of the GA endpoint group"
  type        = string
}

variable "endpoint_group_description" {
  description = "The description of the GA endpoint group"
  type        = string
  default     = "GA endpoint group for health check"
}

variable "backend_region" {
  description = "The region where the backend resources are located"
  type        = string
  default     = "cn-south-1"
}

resource "huaweicloud_ga_endpoint_group" "test" {
  name        = var.endpoint_group_name
  description = var.endpoint_group_description
  region_id   = var.backend_region

  listeners {
    id = huaweicloud_ga_listener.test.id
  }
}
```

**Parameter Description**:
- **name**: The name of the endpoint group, assigned by referencing the input variable endpoint_group_name
- **description**: The description of the endpoint group, assigned by referencing the input variable endpoint_group_description
- **region_id**: The backend region of the endpoint group, assigned by referencing the input variable backend_region
- **listeners.id**: The ID of the associated listener, assigned by referencing the ID of the listener resource created in the previous step

### 5. Create a Global Accelerator Health Check

Add the following script in the TF file (such as main.tf):

```hcl
# Create a global accelerator health check resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "health_check_enabled" {
  description = "Whether to enable health check"
  type        = bool
  default     = true
}

variable "health_check_interval" {
  description = "The health check interval in seconds. Range: 1-60"
  type        = number
  default     = 10
}

variable "health_check_max_retries" {
  description = "The maximum number of retries. Range: 1-10"
  type        = number
  default     = 5
}

variable "health_check_port" {
  description = "The port used for health check. Range: 1-65535"
  type        = number
  default     = 8001
}

variable "health_check_timeout" {
  description = "The timeout duration of health check in seconds. Range: 1-60"
  type        = number
  default     = 10
}

resource "huaweicloud_ga_health_check" "test" {
  endpoint_group_id = huaweicloud_ga_endpoint_group.test.id
  enabled           = var.health_check_enabled
  protocol          = "TCP"
  interval          = var.health_check_interval
  max_retries       = var.health_check_max_retries
  port              = var.health_check_port
  timeout           = var.health_check_timeout
}
```

**Parameter Description**:
- **endpoint_group_id**: The ID of the endpoint group to which the health check belongs, assigned by referencing the ID of the endpoint group resource created in the previous step
- **enabled**: Whether to enable the health check, assigned by referencing the input variable health_check_enabled
- **protocol**: The protocol of the health check, fixed to TCP
- **interval**: The health check interval in seconds, assigned by referencing the input variable health_check_interval
- **max_retries**: The maximum number of retries, assigned by referencing the input variable health_check_max_retries
- **port**: The port used for health check, assigned by referencing the input variable health_check_port
- **timeout**: The timeout duration of health check in seconds, assigned by referencing the input variable health_check_timeout

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through the `tfvars` file, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Global accelerator parameters
accelerator_name    = "ga-accelerator-test"
listener_name       = "ga-listener-test"
endpoint_group_name = "ga-endpoint-group-test"
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

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GA health check
4. Run `terraform show` to view the created GA health check

## Reference Information

- [Huawei Cloud Global Accelerator Product Documentation](https://support.huaweicloud.com/ga/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GA Health Check](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ga/ga-health-check)
