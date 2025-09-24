# 常见问题

## 如何下载并部署本地华为云Provider

在本地执行机网络能正常访问 registry.io 网络的情况下，我们可以通过 terraform init 命令将华为云 provider下载到工作目录下。但对于部分网络受管控的用户来说，使用该命令会消耗较长时间甚至失败，而通过本地部署华为云 Provider 并加载可大大加快初始化的速度。

通过以下步骤可快速在本地搭建一个可用的 Provider 环境：

1. 通过其他途径于 [华为云Provider仓库的发布列表](https://github.com/huaweicloud/terraform-provider-huaweicloud/releases) 中下载预期版本的压缩包，并通过文件传输等方式（如本地网络下载允许则跳过此步骤）将版本压缩包存至本机。

2. 准备本地 Provider 环境所属的工作目录。

- **Linux**: ~/.terraform.d/plugins/<local-registry>/<organization>/huaweicloud/<version>/<os_arch>
- **Windows**: %APPDATA%\terraform.d\plugins\<local-registry>\<organization>\huaweicloud\<version>\<os_arch>
- **MacOS**: ~/.terraform.d/plugins/<local-registry>/<organization>/huaweicloud/<version>/<os_arch>

  注：
  - **local-registry**: 本地注册目录的名称，可自定义。
  - **organization**: 组织名称，可自定义。
  - **version**: 版本号，格式为`a.b.c`。
  - **os_arch**: 执行机架构类型，如windows_amd64（注：现已不支持32位操作系统）。对于MacOS，Apple Silicon的架构类型为darwin_arm64，Intel为darwin_amd64

3. 解压缩至步骤2准备的工作目录，可仅提取其中的terraform-provider-huaweicloud名称（前缀）的可执行文件。

   注：不要将压缩包放至在步骤2准备的工作目录中。

4. Linux及MacOS用户请检查可执行文件的权限是否包含当前操作用户所在组的读权限和可执行权限。

5. 使用本地 Provider 所适配的 Terraform 代码块加载 Provider。

   ```hcl
   terraform {
     required_version = ">= 1.9.0"

     required_providers {
       huaweicloud = {
         source = "local-registry/huaweicloud/huaweicloud"
       }
     }
   }
   ```

   如出现以下错误提示，请根据步骤2-5重新检查本地配置是否正确

   ```bash
   Error: Could not retrieve the list of available versions for provider local-registry/huaweicloud/huaweicloud: could not connect to local-registry: Failed to request discovery document: Get "https://local-registry/.well-know/terraform.json": dial tcp: lookup local-registry: no such host
   ```

## 如何令Terraform在执行过程中生成详细日志

Terraform 具备在执行过程中打印详细执行请求、返回、调试记录等详细日志信息的能力，需要在执行环境中配置以下两个环境变量，配置完成后的后续命令执行即可自动在配置目录下生成日志文件。

| 变量名 | 变量值 |
| ----- | ------ |
| TF_LOG | DEBUG |
| TF_LOG_PATH | ./terraform.log |

Linux及MacOS用户可直接在命令行或bash.rc脚本中通过export命令进行配置：

```bash
$ export TF_LOG=DEBUG
$ export TF_LOG_PATH=./terraform.log
```

Windows用户可在 **此电脑 - 属性 - 高级系统设置 - 环境变量** 中（不用版本的操作系统入口会有不同，这里以Win11举例）配置用户环境变量或系统环境变量或通过 PowerShell 命令行键入以下命令进行配置。

```bash
> $env:TF_LOG="DEBUG"
> $env:TF_LOG_PATH=".\terraform.log"
```

## 如何配置terraform命令自动补全

目前自动补全功能仅支持bash和zsh。通过执行以下命令可以令新会话的terraform命令支持自动补全。

```bash
$ terraform -install=autocomplete
```
