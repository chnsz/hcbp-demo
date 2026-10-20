# Deploy Device Access Environment

## Application Scenario

IoT Device Access (IoTDA) is an IoT platform service provided by Huawei Cloud, offering secure and reliable connectivity, data collection, and command delivery capabilities for massive devices. In IoT services, you usually need to divide a resource space, define a product model, and then register devices before devices can communicate with the cloud.

This best practice will introduce how to use Terraform to automatically deploy an IoTDA device access environment, including resource space creation, product model creation, and device creation, helping you quickly complete the initial configuration of device access.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [IoTDA Resource Space (huaweicloud_iotda_space)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/iotda_space)
- [IoTDA Product (huaweicloud_iotda_product)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/iotda_product)
- [IoTDA Device (huaweicloud_iotda_device)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/iotda_device)

### Resource/Data Source Dependencies

```
huaweicloud_iotda_space
    ├── huaweicloud_iotda_product
    │   └── huaweicloud_iotda_device
    └── huaweicloud_iotda_device
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create an IoTDA Resource Space

Add the following script in the TF file (such as main.tf) to create a resource space:

```hcl
# Create an IoTDA resource space in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "space_name" {
  description = "The IoTDA resource space name"
  type        = string
}

resource "huaweicloud_iotda_space" "test" {
  name = var.space_name
}
```

**Parameter Description**:
- **name**: The resource space name, assigned by referencing the input variable space_name

### 3. Create an IoTDA Product

Add the following script in the TF file (such as main.tf) to create a product under the resource space:

```hcl
# Create an IoTDA product in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "product_name" {
  description = "The IoTDA product name"
  type        = string
}

variable "product_device_type" {
  description = "The device type of the product"
  type        = string
  default     = "Thermometer"
}

variable "product_protocol" {
  description = "The protocol used by the product"
  type        = string
  default     = "MQTT"
}

variable "product_data_type" {
  description = "The data type of the product"
  type        = string
  default     = "json"
}

variable "product_service_id" {
  description = "The service ID of the product"
  type        = string
}

variable "product_service_type" {
  description = "The service type of the product"
  type        = string
  default     = "serv_type"
}

resource "huaweicloud_iotda_product" "test" {
  name        = var.product_name
  device_type = var.product_device_type
  protocol    = var.product_protocol
  space_id    = huaweicloud_iotda_space.test.id
  data_type   = var.product_data_type

  services {
    id   = var.product_service_id
    type = var.product_service_type
  }
}
```

**Parameter Description**:
- **name**: The product name, assigned by referencing the input variable product_name
- **device_type**: The device type of the product, assigned by referencing the input variable product_device_type, defaulting to Thermometer
- **protocol**: The protocol used by the product, assigned by referencing the input variable product_protocol, defaulting to MQTT
- **space_id**: The ID of the resource space to which the product belongs, assigned by referencing the ID of the resource space created in the previous step
- **data_type**: The data format of the product, assigned by referencing the input variable product_data_type, defaulting to json
- **services.id**: The service ID of the product, assigned by referencing the input variable product_service_id
- **services.type**: The service type of the product, assigned by referencing the input variable product_service_type

### 4. Create an IoTDA Device

Add the following script in the TF file (such as main.tf) to create a device under the product:

```hcl
# Create an IoTDA device in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "device_node_id" {
  description = "The node ID of the device"
  type        = string
}

variable "device_name" {
  description = "The device name"
  type        = string
}

variable "device_secret" {
  description = "The secret of the device for authentication"
  type        = string
  sensitive   = true
}

resource "huaweicloud_iotda_device" "test" {
  node_id    = var.device_node_id
  name       = var.device_name
  space_id   = huaweicloud_iotda_space.test.id
  product_id = huaweicloud_iotda_product.test.id
  secret     = var.device_secret
}
```

**Parameter Description**:
- **node_id**: The node identifier of the device, assigned by referencing the input variable device_node_id
- **name**: The device name, assigned by referencing the input variable device_name
- **space_id**: The ID of the resource space to which the device belongs, assigned by referencing the ID of the resource space
- **product_id**: The ID of the product to which the device belongs, assigned by referencing the ID of the product created in the previous step
- **secret**: The secret used by the device for authentication, assigned by referencing the input variable device_secret

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication parameters
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource parameters
space_name         = "your_space_name"
product_name       = "your_product_name"
product_service_id = "service_1"
device_node_id     = "your_device_node_id"
device_name        = "your_device_name"
device_secret      = "1234567890"

# IoTDA service endpoint for standard/enterprise edition instances (optional)
iotda_endpoint = "your_instance_id.iotda-app.cn-north-4.myhuaweicloud.com"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="space_name=my-space"`
2. Environment variables: `export TF_VAR_space_name=my-space`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create the device access environment:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the resource space, product, and device
4. Run `terraform show` to view the created device access environment

## Reference Information

- [Huawei Cloud IoTDA Product Documentation](https://support.huaweicloud.com/iotda/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For IoTDA Device Access Environment](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/iotda/device-access)
