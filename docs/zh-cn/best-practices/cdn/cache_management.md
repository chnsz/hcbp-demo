# 部署缓存管理

## 应用场景

内容分发网络（Content Delivery Network，CDN）缓存管理是CDN服务提供的缓存刷新和预热功能，用于管理CDN节点上的缓存内容。通过缓存刷新，您可以强制CDN节点删除指定的缓存内容，确保用户获取到最新的资源。通过缓存预热，您可以提前将热门内容缓存到CDN节点，提高用户访问速度。通过Terraform自动化管理CDN缓存，可以确保缓存操作的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化执行CDN缓存刷新和预热操作。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [CDN缓存刷新资源（huaweicloud_cdn_cache_refresh）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cdn_cache_refresh)
- [CDN缓存预热资源（huaweicloud_cdn_cache_preheat）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cdn_cache_preheat)

### 资源/数据源依赖关系

```
huaweicloud_cdn_cache_refresh
    └── huaweicloud_cdn_cache_preheat
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建CDN缓存刷新资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CDN缓存刷新资源：

```hcl
variable "refresh_file_urls" {
  description = "The list of file URLs that need to be refreshed"
  type        = list(string)
  default     = []
  nullable    = false

  validation {
    condition     = length(var.refresh_file_urls) <= 1000
    error_message = "The refresh_file_urls list can contain up to 1000 URLs."
  }
}

variable "zh_url_encode" {
  description = "Whether to encode Chinese characters in URLs before cache refresh/preheat"
  type        = bool
  default     = false
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the resource belongs"
  type        = string
  default     = "0"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CDN缓存刷新资源
resource "huaweicloud_cdn_cache_refresh" "test" {
  count = length(var.refresh_file_urls) > 0 ? 1 : 0

  type                  = "file"
  urls                  = var.refresh_file_urls
  mode                  = "all"
  zh_url_encode         = var.zh_url_encode
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：
- **count**：资源创建条件，当refresh_file_urls列表不为空时创建资源
- **type**：刷新类型，设置为"file"表示文件刷新
- **urls**：需要刷新的文件URL列表，通过引用输入变量refresh_file_urls进行赋值，最多支持1000个URL
- **mode**：刷新模式，设置为"all"表示刷新URL及其目录下的所有内容
- **zh_url_encode**：是否对URL中的中文字符进行编码，通过引用输入变量zh_url_encode进行赋值，默认值为false
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，默认值为"0"

### 3. 创建CDN缓存预热资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CDN缓存预热资源：

```hcl
variable "preheat_urls" {
  description = "The list of URLs that need to be preheated"
  type        = list(string)
  default     = []
  nullable    = false

  validation {
    condition     = length(var.preheat_urls) <= 1000
    error_message = "The preheat_urls list can contain up to 1000 URLs."
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CDN缓存预热资源
resource "huaweicloud_cdn_cache_preheat" "test" {
  count = length(var.preheat_urls) > 0 ? 1 : 0

  urls                  = var.preheat_urls
  zh_url_encode         = var.zh_url_encode
  enterprise_project_id = var.enterprise_project_id

  depends_on = [
    huaweicloud_cdn_cache_refresh.test,
  ]
}
```

**参数说明**：
- **count**：资源创建条件，当preheat_urls列表不为空时创建资源
- **urls**：需要预热的URL列表，通过引用输入变量preheat_urls进行赋值，最多支持1000个URL
- **zh_url_encode**：是否对URL中的中文字符进行编码，通过引用输入变量zh_url_encode进行赋值，默认值为false
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，默认值为"0"
- **depends_on**：显式依赖关系，确保缓存刷新资源先于缓存预热资源创建

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# CDN缓存管理配置
refresh_file_urls = [
  "https://example.com/index.html",
]
preheat_urls = [
  "https://example.com/index.html",
]

zh_url_encode         = false
enterprise_project_id = ""
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="refresh_file_urls=[\"https://example.com/index.html\"]" -var="preheat_urls=[\"https://example.com/index.html\"]"`
2. 环境变量：`export TF_VAR_refresh_file_urls='["https://example.com/index.html"]'` 和 `export TF_VAR_preheat_urls='["https://example.com/index.html"]'`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来执行CDN缓存管理操作：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始执行缓存刷新和预热操作
4. 运行 `terraform show` 查看已执行的缓存管理操作详情

> 注意：所有URL必须属于已在CDN中配置的域名。缓存刷新操作通常在几分钟内完成。缓存预热操作可能需要更长时间，具体取决于URL数量。当URL包含中文字符时，启用中文URL编码会很有用。使用子账号时需要提供企业项目ID。

## 参考信息

- [华为云CDN产品文档](https://support.huaweicloud.com/cdn/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [缓存管理最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cdn/cache-management)
