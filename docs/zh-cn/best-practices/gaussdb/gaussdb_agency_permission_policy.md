# 部署委托权限策略

## 应用场景

GaussDB是华为云提供的高性能、高可用、高安全的企业级分布式关系型数据库服务。在使用GaussDB的过程中，部分功能（例如容灾、备份等）需要依赖委托（Agency）来获取访问其他云服务的权限，而委托本身需要绑定相应的权限策略才能正常工作。

本最佳实践将介绍如何使用Terraform自动化部署GaussDB委托权限策略，包括委托名称指定、权限策略绑定以及权限策略解绑。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [GaussDB委托权限策略（huaweicloud_gaussdb_agency_permission_policy）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_agency_permission_policy)

### 资源/数据源依赖关系

```
huaweicloud_gaussdb_agency_permission_policy
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建GaussDB委托权限策略

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建GaussDB委托权限策略资源
variable "agency_name" {
  description = "The agency name. Currently, only RDSAccessProjectResource is supported"
  type        = string
  default     = "RDSAccessProjectResource"
}

variable "bind_role_names" {
  description = "The permission policies to be bound to the agency"
  type        = set(string)
}

variable "unbind_role_names" {
  description = "The permission policies to be unbound from the agency"
  type        = set(string)
}

resource "huaweicloud_gaussdb_agency_permission_policy" "test" {
  agency_name       = var.agency_name
  bind_role_names   = var.bind_role_names
  unbind_role_names = var.unbind_role_names
}
```

**参数说明**：

- **agency_name**：委托名称，通过引用输入变量 agency_name 进行赋值，当前仅支持 `RDSAccessProjectResource`，该参数不支持更新，修改后会触发资源重建
- **bind_role_names**：需要绑定到委托的权限策略列表，通过引用输入变量 bind_role_names 进行赋值
- **unbind_role_names**：需要从委托解绑的权限策略列表，通过引用输入变量 unbind_role_names 进行赋值

> 说明：`bind_role_names` 与 `unbind_role_names` 会在同一次请求中生效；删除该资源时仅会将其从Terraform状态中移除，委托上的权限策略配置不会被实际删除。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 委托名称（可选，默认为 RDSAccessProjectResource）
agency_name = "RDSAccessProjectResource"

# 需要绑定到委托的权限策略
bind_role_names = ["DBS AgencyPolicy", "GaussDB FullAccess"]

# 需要从委托解绑的权限策略
unbind_role_names = ["GaussDB ReadOnlyAccess"]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="agency_name=RDSAccessProjectResource"`
2. 环境变量：`export TF_VAR_agency_name=RDSAccessProjectResource`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建GaussDB委托权限策略
4. 运行 `terraform show` 查看已创建的GaussDB委托权限策略

## 参考信息

- [华为云GaussDB产品文档](https://support.huaweicloud.com/gaussdb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [GaussDB委托权限策略最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/gaussdb-agency-permission-policy)
