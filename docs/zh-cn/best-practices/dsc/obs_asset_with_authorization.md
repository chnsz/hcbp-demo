# 部署OBS资产授权与资产添加

## 应用场景

数据安全中心（Data Security Center，DSC）是华为云提供的一站式数据安全治理服务，支持对对象存储服务（OBS）中的数据进行敏感数据识别与分类分级。在使用DSC对OBS桶进行敏感数据扫描之前，需要先为DSC开启对应资产类型的授权，并将目标OBS桶添加为DSC资产。

本最佳实践将介绍如何使用Terraform自动化完成DSC的OBS资产授权与OBS资产添加，包括创建OBS桶、开启DSC的OBS资产授权以及将OBS桶添加为DSC资产。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [OBS桶（huaweicloud_obs_bucket）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket)
- [DSC资产授权（huaweicloud_dsc_asset_authorization）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dsc_asset_authorization)
- [DSC OBS资产（huaweicloud_dsc_asset_obs）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dsc_asset_obs)

### 资源/数据源依赖关系

```
huaweicloud_obs_bucket
    └── huaweicloud_dsc_asset_obs

huaweicloud_dsc_asset_authorization
    └── huaweicloud_dsc_asset_obs
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建OBS桶

在TF文件（如main.tf）中添加以下脚本以创建OBS桶：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建OBS桶资源
variable "bucket_name" {
  description = "The name of the OBS bucket to be added as a DSC asset"
  type        = string
}

resource "huaweicloud_obs_bucket" "test" {
  bucket        = var.bucket_name
  acl           = "private"
  force_destroy = true
}
```

**参数说明**：

- **bucket**：OBS桶名称，通过引用输入变量 bucket_name 进行赋值
- **acl**：OBS桶的访问控制策略，设置为 private 表示私有读写
- **force_destroy**：设置为 true 表示删除桶时会强制删除桶内所有对象

### 3. 开启DSC的OBS资产授权

在TF文件（如main.tf）中添加以下脚本以开启DSC的OBS资产授权：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下开启DSC的OBS资产授权
resource "huaweicloud_dsc_asset_authorization" "test" {
  type                 = "OBS"
  authorization_status = true
}
```

**参数说明**：

- **type**：资产类型，设置为 OBS 表示对OBS资产进行授权
- **authorization_status**：授权状态，设置为 true 表示开启授权

### 4. 添加OBS资产到DSC

在TF文件（如main.tf）中添加以下脚本以将OBS桶添加为DSC资产：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下添加OBS资产到DSC
variable "asset_name" {
  description = "The name of the DSC OBS asset"
  type        = string
}

resource "huaweicloud_dsc_asset_obs" "test" {
  name          = var.asset_name
  bucket_name   = huaweicloud_obs_bucket.test.bucket
  bucket_policy = "private"

  depends_on = [huaweicloud_dsc_asset_authorization.test]
}
```

**参数说明**：

- **name**：DSC OBS资产名称，通过引用输入变量 asset_name 进行赋值，需在已添加的OBS资产中保持唯一
- **bucket_name**：OBS桶名称，通过引用OBS桶资源的 bucket 属性进行赋值
- **bucket_policy**：OBS桶策略，需与实际OBS桶的ACL保持一致，设置为 private 表示私有
- **depends_on**：显式声明依赖关系，确保在资产授权开启后再添加OBS资产

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
bucket_name = "tf-test-dsc-obs-bucket"
asset_name  = "tf-test-dsc-obs-asset"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="bucket_name=my-bucket"`
2. 环境变量：`export TF_VAR_bucket_name=my-bucket`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建OBS资产授权与OBS资产
4. 运行 `terraform show` 查看已创建的OBS资产授权与OBS资产

## 参考信息

- [华为云数据安全中心产品文档](https://support.huaweicloud.com/dsc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DSC OBS资产授权与资产添加最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dsc/obs-asset-with-authorization)
