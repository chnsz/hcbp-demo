# 部署组织单元

## 应用场景

华为云组织（Organizations）服务是华为云提供的多账户管理服务，支持企业通过组织架构统一管理多个华为云账户，实现资源的集中管理和权限的统一控制。组织单元是Organizations服务中的重要资源，用于在组织下创建层级化的组织结构，实现账户和资源的分类管理。通过创建组织单元，可以将账户按照业务、部门或项目进行分组，实现资源的隔离、成本的分摊和权限的精细化管理，满足企业级多账户管理的需求。本最佳实践将介绍如何使用Terraform自动化部署Organizations组织单元，包括组织单元的创建和配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [Organizations组织单元资源（huaweicloud_organizations_organizational_unit）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/organizations_organizational_unit)

### 数据源

- [Organizations组织数据源（huaweicloud_organizations_organization）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/organizations_organization)

### 资源/数据源依赖关系

```text
data.huaweicloud_organizations_organization
    └── huaweicloud_organizations_organizational_unit
```

> 注意：组织单元依赖组织数据源获取根组织ID，通过parent_id参数指定父级组织或组织单元。组织单元可以嵌套创建，形成层级化的组织结构。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询组织信息

在TF文件（如main.tf）中添加以下脚本以查询组织信息：

```hcl
data "huaweicloud_organizations_organization" "test" {}
```

**参数说明**：
- 该数据源无需输入参数，会自动查询当前账户所属的组织信息，包括根组织ID等。

### 3. 创建组织单元

在TF文件（如main.tf）中添加以下脚本以创建组织单元：

```hcl
resource "huaweicloud_organizations_organizational_unit" "test" {
  name      = var.organizational_unit_name
  parent_id = data.huaweicloud_organizations_organization.test.root_id
  tags      = var.tags
}
```

**参数说明**：
- **name**：组织单元名称，通过引用输入变量 `organizational_unit_name` 进行赋值，用于标识组织单元
- **parent_id**：父级组织或组织单元ID，通过引用数据源 `data.huaweicloud_organizations_organization.test.root_id` 获取根组织ID，用于指定组织单元所属的父级组织或组织单元
- **tags**：组织单元标签，通过引用输入变量 `tags` 进行赋值，用于对组织单元进行分类和管理

### 4. 预设资源部署所需的入参（可选）

本实践中，资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 组织单元配置（必填）
organizational_unit_name = "tf_test_organizational_unit"

tags = {
  key   = "value"
  owner = "terraform"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="organizational_unit_name=tf_test_organizational_unit"`
2. 环境变量：`export TF_VAR_organizational_unit_name=tf_test_organizational_unit`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建组织单元
4. 运行 `terraform show` 查看已创建的组织单元

## 参考信息

- [华为云Organizations产品文档](https://support.huaweicloud.com/organizations/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [组织单元最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/organizations/organization-unit)
