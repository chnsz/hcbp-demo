# 部署华为云资源前的准备工作

在开始使用Terraform之前，您需要完成以下准备步骤。

## 安装Terraform

Terraform的安装方式因操作系统而异，我们提供了详细的安装指南文档：[如何安装Terraform](how_to_install_terraform.md)
请根据文档的指引进行操作。

## 准备部署脚本的工作目录

创建一个新的工作目录并准备Terraform配置文件（扩展名为.tf，如main.tf）：

```bash
mkdir -p terraform-demo
cd terraform-demo
touch main.tf
```

在`main.tf`中，您将定义用于部署华为云资源所需的HCL脚本。

## 配置华为云认证信息

### 方式一：环境变量配置（推荐）

您可以通过设置以下环境变量来配置华为云认证信息：

```bash
# 必选配置
export HW_ACCESS_KEY="your_ak"     # AK，于控制台"我的凭证-访问密钥"处生成
export HW_SECRET_KEY="your_sk"     # SK，于控制台"我的凭证-访问密钥"处生成
export HW_REGION_NAME="cn-north-4" # 区域代码，如：cn-north-4

# 可选配置
export HW_ENTERPRISE_PROJECT_ID="your_eps_id" # 企业项目ID（如果需要）
export HW_DOMAIN_NAME="your_domain_name"      # 账号名（如果需要跨账号操作）
```

### 方式二：Provider配置块

在Terraform配置文件中直接配置认证信息：

```hcl
terraform {
  required_providers {
    huaweicloud = {
      source  = "huaweicloud/huaweicloud"
      version = ">= 1.xx.x" # 加载不低于1.xx.x版本的最新provider，改配置对本地provider仍有效
    }
  }
}

provider "huaweicloud" {
  access_key = "your_ak"    # 必填，AK，于控制台"我的凭证-访问密钥"处生成
  secret_key = "your_sk"    # 必填，SK，于控制台"我的凭证-访问密钥"处生成
  region     = "cn-north-4" # 必填，区域代码，如：cn-north-4

  # 以下为可选配置
  enterprise_project_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" # 企业项目ID，如果账号开通了企业管理功能，则该ID默认为"0"
  domain_name           = "your_domain_name"                     # 账号名
}
```

## 参考信息

- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
