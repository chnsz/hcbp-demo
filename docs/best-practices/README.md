# 华为云最佳实践文档中心

欢迎来到华为云最佳实践文档中心。本文档集合包含了各种华为云服务的最佳实践示例，帮助您更好地使用华为云解决实际业务问题。

## 文档导航

### [API网关（APIG）最佳实践](apig/index.md)

API网关（API Gateway）是为企业和开发者提供的高性能、高可用、高安全的云原生网关服务。

### [弹性伸缩（AS）最佳实践](as/index.md)

弹性伸缩（Auto Scaling）是一种根据用户的业务需求和策略，自动调整计算资源的服务，帮助您实现资源的动态调配和成本优化。

### [云容器引擎（CCE）最佳实践](cce/postpaid_cluster.md)

云容器引擎（Cloud Container Engine, CCE）是一个高可靠高性能的企业级容器管理服务，支持Kubernetes社区原生应用和工具，提供容器化应用的全生命周期管理能力。

### [弹性云服务器（ECS）最佳实践](ecs/index.md)

弹性云服务器（Elastic Cloud Server, ECS）是由CPU、内存、操作系统、云硬盘组成的基础的计算组件，为您的应用提供可靠、安全、灵活、高效的计算环境。

### [函数工作流（FunctionGraph）最佳实践](fgs/index.md)

函数工作流（FunctionGraph）是一项基于事件驱动的无服务器计算服务，支持多种编程语言和触发方式，让您无需管理服务器即可快速构建应用。

### [Web应用防火墙（WAF）最佳实践](waf/index.md)

Web应用防火墙（WAF）为Web应用提供一站式安全防护，支持多种攻击检测与拦截，满足企业级安全与合规需求。

### [云桌面（Workspace）最佳实践](workspace/index.md)

云桌面（Workspace）是一种基于云计算的桌面虚拟化服务，为企业用户提供安全、便捷的云上办公解决方案，支持多种操作系统和终端接入方式。

## 文档特点

- **实践导向**：每个最佳实践都基于实际业务场景
- **完整示例**：提供完整的代码示例和配置说明
- **详细指导**：包含步骤详解和注意事项说明
- **原理解析**：深入浅出地解释技术原理

## 文档结构

每个最佳实践文档通常包含以下部分：

1. **概述**
   - 应用场景
   - 方案优势
   - 涉及产品

2. **资源设计**
   - 资源清单
   - 架构说明
   - 依赖关系

3. **实现步骤**
   - 准备工作
   - 详细步骤
   - 验证测试

4. **最佳实践**
   - 安全建议
   - 性能优化
   - 成本控制

## 使用指南

### 在线浏览

您可以通过以下方式浏览文档：

1. **GitBook方式**
   ```bash
   # 安装GitBook CLI
   npm install -g gitbook-cli
   
   # 在docs目录下启动服务
   cd docs
   gitbook serve
   ```

2. **GitHub方式**
   - 直接在GitHub上浏览Markdown文件
   - 使用GitHub的文件导航功能

### 离线阅读

1. **下载PDF版本**
   ```bash
   # 在docs目录下生成PDF
   gitbook pdf ./ ./华为云最佳实践手册.pdf
   ```

2. **克隆仓库到本地**
   ```bash
   git clone https://github.com/your-org/hcbp-demo.git
   ```

## 文档更新

我们会持续更新和完善最佳实践文档：

- **定期更新**：根据产品更新和用户反馈
- **版本控制**：使用Git管理文档版本
- **更新日志**：记录重要变更

## 反馈与建议

我们非常重视您的反馈和建议：

1. **问题反馈**
   - 在GitHub上提交Issue
   - 描述您遇到的问题
   - 提供复现步骤

2. **内容建议**
   - 提出新的最佳实践主题
   - 完善现有文档内容
   - 分享您的实践经验

3. **贡献方式**
   - Fork仓库
   - 提交Pull Request
   - 参与文档讨论

## 相关资源

- [华为云官方文档中心](https://support.huaweicloud.com/)
- [Terraform最佳实践（GitHub）](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples)

## 免责声明

- 本文档仅供参考，不构成任何明示或暗示的保证
- 请在实际部署前充分测试和验证
- 遵循华为云服务的使用条款和条件
