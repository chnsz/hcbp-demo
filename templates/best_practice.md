# 最佳实践标题（内容根据最佳实践的标题进行生成，如：部署一台弹性云服务器；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

## 应用场景（固定标题；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

弹性云服务器（Elastic Cloud Server，ECS）是由CPU、内存、操作系统、云硬盘组成的基础的计算组件。
弹性云服务器创建成功后，您就可以像使用自己的本地PC或物理服务器一样，在云上使用弹性云服务器。华为云提供了多种类型的弹性云服务器，可满足不同的使用场景。
在创建之前，您需要根据实际的应用场景确认弹性云服务器的规格类型，镜像类型，磁盘种类等参数，并选择合适的网络参数和安全组规则。
本最佳实践将介绍如何使用Terraform自动化部署一个基础的ECS实例。

## 相关资源/数据源（固定标题；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

本最佳实践涉及以下主要资源和数据源：（固定表达；本括号及括号内的内容不在自动生成的最终结果中）

### 数据源（固定标题；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）
（数据源的排序与对应参考脚本保持一致；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

- [数据源A的释义（data.huaweicloud_xxx_xxxa）](数据源对应的provider文档跳转链接)
- [数据源B的释义（data.huaweicloud_xxx_xxxb）](数据源对应的provider文档跳转链接)
...（如果有其他数据源，按照相同的格式进行生成；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

（例如：- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

### 资源

- [资源A的释义（huaweicloud_xxx_xxxa）](资源对应的provider文档跳转链接)
- [资源B的释义（huaweicloud_xxx_xxxb）](资源对应的provider文档跳转链接)
...（如果有其他资源，按照相同的格式进行生成；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

（例如：- [ECS实例资源（data.huaweicloud_compute_instance](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

### 资源/数据源依赖关系

```
data.huaweicloud_xxx_xxxa
    └── huaweicloud_xxx_xxxa

data.huaweicloud_xxx_xxxb
    └── huaweicloud_xxx_xxxb

huaweicloud_xxx_xxxa
    └── huaweicloud_xxx_xxxb
        ├── huaweicloud_xxx_xxxc
        └── huaweicloud_xxx_xxxd
```

（资源/数据源依赖关系块中必须包含全部的数据源和资源，格式严格按照上述示例作为标准，注意排序；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

## 操作步骤（固定标题；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

### 1. 脚本准备（固定标题；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../docs/introductions/prepare_before_deploy.md)一文中的介绍。

（本节为固定描述，因为所有的最佳实践一定有这一步；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

### 2. 通过数据源查询ECS实例资源创建所需的可用区（data.huaweicloud_availability_zones）

（从第二小节开始按照最佳实践脚本的定义顺序逐一进行说明；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建ECS实例（包括用于筛选规格）：

（此处贴上对应HCL代码，如下HCL代码块所示，数据源说明部分的代码需要对数据源进行注释说明。
如： # 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的xxx信息，用于......
其中数据源的注释介绍有两种格式：
  a. 数据源中声明了region，则该功能介绍例如该格式：获取特定region下所有的可用区信息，用于创建ECS实例。
  b. 数据源中未声明region，则该功能介绍例如该格式：获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建ECS实例。
注：如果资源中声明了文档，其需要置于参数的顶部，即Meta参数的下方的首个参数声明
本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）
```hcl
variable "availability_zone" {
  description = "ECS实例所属的可用区信息"
  type        = string
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的xxx信息，用于创建ECS实例资源
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0

  ......（参数定义，如果有的话；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）
}
```
（根据最佳实践脚本内容将对应数据源的脚本代码填入该HCL代码块并补充对应输入变量的定义；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

**参数说明**：（固定标题；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）
- **count**：数据源的创建数，用于控制是否执行可用区列表查询数据源，仅当 `var.availability_zone` 为空时创建数据源（即执行可用区列表查询）
......（其他参数定义；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

......
（所有的数据源都依照该示例进行生成，如果有特殊约束，则在参数说明的后方进行补充说明；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

### n. 创建ECS实例（huaweicloud_compute_instance）

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ECS实例资源：

（此处贴上对应HCL代码，如下HCL代码块所示，代码中如果引用了数据源需要对数据源进行注释说明。
如： # 在指定region（默认继承当前provider块中所指定的region）下创建xxx资源，......
其中数据源的注释介绍有两种格式：
  a. 资源中声明了region，则该功能介绍例如该格式：在指定region下创建ECS实例资源。
  b. 资源中未声明region，则该功能介绍例如该格式：在指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建云桌面实例。
注：如果资源中声明了文档，其需要置于参数的顶部，即Meta参数的下方的首个参数声明
本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）
```hcl
variable "instance_name" {
  description = "The name of the ECS instance"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ECS实例资源。
data "huaweicloud_compute_instance" "test" {
  name              = var.instance_name
  availability_zone = data.huaweicloud_availability_zones.test.names[0]
  flavor_id         = try(data.huaweicloud_compute_flavors.test.flavors[0].id, "")
  image_id          = try(data.huaweicloud_images_images.test.images[0].id, "")

  ......（参数定义，如果有的话；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

  lifecycle {
    ignore_changes = [
      admin_pass,
    ]
  }
}
```
（根据最佳实践脚本内容将对应数据源的脚本代码填入该HCL代码块并补充对应输入变量的定义；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

**参数说明**：（固定标题；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）
- **name**：ECS实例的名称，通过引用输入变量instance_name进行赋值
- **availability_zone**：ECS实例所在的可用区，根据可用区列表查询数据源（data.huaweicloud_availability_zones）的返回结果进行赋值
- **flavor_id**：ECS实例所使用规格的ID，根据ECS规格列表查询数据源（data.huaweicloud_compute_flavors）的返回结果进行赋值
- **image_id**：ECS实例所使用镜像的ID，根据IMS镜像列表查询数据源（data.huaweicloud_images_images）的返回结果进行赋值
......（其他参数定义；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

......
（所有的资源都依照该示例进行生成，如果有特殊约束，则在参数说明的后方进行补充说明，如：
> 要远程登录（22/3389）、公网ping（ICMP需单独配置）、以及网站服务（80/443）的云服务器场景。
  该规则示例中使用了示例IP段（192.168.1.0/24），实际部署时请替换为您企业办公网络的具体IP段。
  建议遵循最小权限原则，只开放必要的端口和IP范围，以提高安全性。
本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

### x. 初始化并应用Terraform配置

（最后一个小节，标题为固定表达；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建xxx（替换为被创建对象，也就是本实践的主要介绍对象；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）
4. 运行 `terraform show` 查看已创建的xxx（替换为被创建对象，也就是本实践的主要介绍对象；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

## 参考信息（固定标题；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

（Terraform华为云Provider文档为固定参考信息，不作修改，其他参考部分根据各自最佳实践进行修改；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

- [华为云ECS产品文档](https://support.huaweicloud.com/ecs/index.html)
（在华为云Supports文档中找到对应云产品的index页，附上对应链接；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
（固定参考信息，所有最佳实践参考均需附上该跳转链接；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）
- [ECS最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/workspace/desktop/basic)
（生成该最佳实践的脚本（main.tf）所在目录，附上对应目录的跳转链接；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）

（生成的最佳实践文档（.md）的行末需要有一个空行；本括号中的内容仅供自动生成参考，本括号及括号内的内容不在自动生成的最终结果中）
