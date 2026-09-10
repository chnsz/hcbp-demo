# 部署Redis诊断任务

## 应用场景

分布式缓存服务（DCS）Redis实例在长期运行过程中，可能出现大Key、慢查询、连接数异常、内存碎片率偏高等潜在问题，影响缓存服务的性能与稳定性。通过诊断任务，可以对指定时间范围内的Redis实例运行状况进行智能分析，输出异常项与失败项统计以及各节点的诊断报告，帮助运维人员快速定位并处理隐患。

本最佳实践将介绍如何使用Terraform自动化部署DCS Redis诊断任务，针对已存在的Redis实例在指定时间范围内发起诊断，并获取诊断结果。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [DCS Redis诊断任务（huaweicloud_dcs_diagnosis_task）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_diagnosis_task)

### 资源/数据源依赖关系

```
huaweicloud_dcs_diagnosis_task
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建DCS Redis诊断任务

在TF文件（如main.tf）中添加以下脚本以创建DCS Redis诊断任务：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DCS Redis诊断任务资源
variable "instance_id" {
  description = "The ID of the DCS instance to diagnose"
  type        = string
}

variable "begin_time" {
  description = "The start time of the diagnosis task, in RFC3339 format"
  type        = string
}

variable "end_time" {
  description = "The end time of the diagnosis task, in RFC3339 format"
  type        = string
}

resource "huaweicloud_dcs_diagnosis_task" "test" {
  instance_id = var.instance_id
  begin_time  = var.begin_time
  end_time    = var.end_time
}
```

**参数说明**：

- **instance_id**：待诊断的DCS实例ID，通过引用输入变量 instance_id 进行赋值
- **begin_time**：诊断任务的开始时间，RFC3339格式，通过引用输入变量 begin_time 进行赋值
- **end_time**：诊断任务的结束时间，RFC3339格式，通过引用输入变量 end_time 进行赋值

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源信息
instance_id = "your_dcs_instance_id"
begin_time  = "2024-01-01T00:00:00Z"
end_time    = "2024-01-02T00:00:00Z"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="instance_id=your_dcs_instance_id"`
2. 环境变量：`export TF_VAR_instance_id=your_dcs_instance_id`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DCS Redis诊断任务
4. 运行 `terraform show` 查看已创建的DCS Redis诊断任务

## 参考信息

- [华为云DCS产品文档](https://support.huaweicloud.com/dcs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DCS Redis诊断任务最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-diagnosis-task)
