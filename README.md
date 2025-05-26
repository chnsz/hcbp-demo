# 华为云最佳实践示例

本项目包含了一系列华为云服务的最佳实践示例，旨在帮助用户更好地使用华为云服务，解决实际业务场景中的问题。

## 最佳实践列表

+ [API网关 (APIG)](docs/apig/index.md)
+ [函数工作流 (FunctionGraph)](docs/fgs/index.md)
+ [Web应用防火墙 (WAF)](docs/waf/index.md)
+ [云桌面 (Workspace)](docs/workspace/index.md)

## 使用说明

### 环境要求

- Terraform 0.12.x 或更高版本
- 华为云账号和访问密钥（AK/SK）
- 相关服务的权限和配额

### 快速开始

1. **配置华为云认证信息**

   ```bash
   export HW_ACCESS_KEY="your-ak"
   export HW_SECRET_KEY="your-sk"
   export HW_REGION_NAME="cn-north-4"
   ```

2. **克隆代码仓库**

   ```bash
   git clone https://github.com/your-org/hcbp-demo.git
   cd hcbp-demo
   ```

3. **查看文档**

   本项目使用GitBook组织文档，您可以：
   - 直接在GitHub上浏览Markdown文档
   - 使用GitBook本地服务器查看文档
   ```bash
   npm install -g gitbook-cli
   gitbook serve
   ```

4. **运行示例**

   每个最佳实践都包含完整的操作步骤，请参考具体文档进行操作。

## 文档约定

- 所有文档均采用Markdown格式编写
- 示例代码采用Terraform HCL语言
- 配置说明包含中英文注释
- 资源命名遵循华为云最佳实践规范

## 贡献指南

我们欢迎您为这个项目做出贡献：

1. Fork本仓库
2. 创建您的特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交您的更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 开启一个Pull Request

## 注意事项

1. **安全性**
   - 不要在代码中硬编码敏感信息
   - 使用环境变量或配置文件管理密钥
   - 遵循最小权限原则配置IAM策略

2. **成本控制**
   - 注意及时清理不再使用的资源
   - 合理设置自动缩放策略
   - 关注资源的计费模式

3. **最佳实践**
   - 参考华为云官方文档
   - 遵循基础设施即代码（IaC）原则
   - 做好资源标签管理

## 相关资源

- [华为云官方文档](https://support.huaweicloud.com/)
- [Terraform华为云Provider](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [API网关服务文档](https://support.huaweicloud.com/apig/index.html)
- [函数工作流服务文档](https://support.huaweicloud.com/functiongraph/index.html)
- [云桌面服务文档](https://support.huaweicloud.com/workspace/index.html)

## 许可证

本项目采用 [Apache 2.0 许可证](LICENSE)。

## 联系我们

如果您有任何问题或建议，请：
- 提交Issue
- 发送邮件至 [support@example.com](mailto:support@example.com)
- 访问我们的[官方网站](https://example.com) 
