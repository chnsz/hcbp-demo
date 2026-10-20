# 部署设备接入环境

## 应用场景

设备接入服务（IoT Device Access，IoTDA）是华为云提供的物联网平台服务，为海量设备提供安全可靠的连接、数据采集与命令下发能力。在物联网业务中，通常需要先划分资源空间，再定义产品模型，最后注册设备，才能完成设备与云端的对接。

本最佳实践将介绍如何使用Terraform自动化部署IoTDA设备接入环境，包括资源空间创建、产品模型创建和设备创建，帮助您快速完成设备接入的初始化配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [IoTDA资源空间（huaweicloud_iotda_space）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/iotda_space)
- [IoTDA产品（huaweicloud_iotda_product）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/iotda_product)
- [IoTDA设备（huaweicloud_iotda_device）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/iotda_device)

### 资源/数据源依赖关系

```
huaweicloud_iotda_space
    ├── huaweicloud_iotda_product
    │   └── huaweicloud_iotda_device
    └── huaweicloud_iotda_device
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建IoTDA资源空间

在TF文件（如main.tf）中添加以下脚本以创建资源空间：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建IoTDA资源空间
variable "space_name" {
  description = "The IoTDA resource space name"
  type        = string
}

resource "huaweicloud_iotda_space" "test" {
  name = var.space_name
}
```

**参数说明**：
- **name**：资源空间名称，通过引用输入变量 space_name 进行赋值

### 3. 创建IoTDA产品

在TF文件（如main.tf）中添加以下脚本以在资源空间下创建产品：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建IoTDA产品
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

**参数说明**：
- **name**：产品名称，通过引用输入变量 product_name 进行赋值
- **device_type**：产品所属的设备类型，通过引用输入变量 product_device_type 进行赋值，默认值为 Thermometer
- **protocol**：产品使用的协议，通过引用输入变量 product_protocol 进行赋值，默认值为 MQTT
- **space_id**：产品所属的资源空间ID，引用上一步创建的资源空间的ID进行赋值
- **data_type**：产品的数据格式，通过引用输入变量 product_data_type 进行赋值，默认值为 json
- **services.id**：产品服务ID，通过引用输入变量 product_service_id 进行赋值
- **services.type**：产品服务类型，通过引用输入变量 product_service_type 进行赋值

### 4. 创建IoTDA设备

在TF文件（如main.tf）中添加以下脚本以在产品下创建设备：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建IoTDA设备
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

**参数说明**：
- **node_id**：设备的节点标识，通过引用输入变量 device_node_id 进行赋值
- **name**：设备名称，通过引用输入变量 device_name 进行赋值
- **space_id**：设备所属的资源空间ID，引用资源空间的ID进行赋值
- **product_id**：设备所属的产品ID，引用上一步创建的产品的ID进行赋值
- **secret**：设备用于认证的密钥，通过引用输入变量 device_secret 进行赋值

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证参数
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源参数
space_name         = "your_space_name"
product_name       = "your_product_name"
product_service_id = "service_1"
device_node_id     = "your_device_node_id"
device_name        = "your_device_name"
device_secret      = "1234567890"

# 标准版/企业版实例的IoTDA服务端点（可选）
iotda_endpoint = "your_instance_id.iotda-app.cn-north-4.myhuaweicloud.com"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="space_name=my-space"`
2. 环境变量：`export TF_VAR_space_name=my-space`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建设备接入环境：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建资源空间、产品和设备
4. 运行 `terraform show` 查看已创建的设备接入环境

## 参考信息

- [华为云IoTDA产品文档](https://support.huaweicloud.com/iotda/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [IoTDA设备接入环境最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/iotda/device-access)
