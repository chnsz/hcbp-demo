# Deploy Cloud Domain

## Application Scenario

Huawei Cloud Web Application Firewall (WAF) Cloud Mode is a Web security protection service based on shared resources that can provide Web attack protection for specified domains. By configuring WAF cloud mode domains, you can provide protection capabilities for your websites against various common Web attacks such as SQL injection, XSS cross-site scripting, web Trojan uploads, command injection, malicious crawlers, and CC attacks. WAF cloud mode domains support flexible origin server configuration, SSL certificate management, custom error pages, timeout settings, and traffic marking, meeting the security protection needs of different business scenarios. This best practice introduces how to use Terraform to automatically deploy a WAF cloud mode domain, including creating WAF cloud instances and domain configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [WAF Cloud Instance Resource (huaweicloud_waf_cloud_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/waf_cloud_instance)
- [WAF Domain Resource (huaweicloud_waf_domain)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/waf_domain)

### Resource/Data Source Dependencies

```text
huaweicloud_waf_cloud_instance
    └── huaweicloud_waf_domain
```

> Note: WAF domain depends on WAF cloud instance. Before creating a WAF domain, you need to create a WAF cloud instance first. WAF cloud instance provides basic security protection capabilities, and WAF domain is a specific protection domain configured on the basis of the cloud instance.

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Create WAF Cloud Instance

Add the following script to the TF file (such as main.tf) to create a WAF cloud instance:

```hcl
resource "huaweicloud_waf_cloud_instance" "test" {
  resource_spec_code = var.cloud_instance_resource_spec_code

  dynamic "bandwidth_expack_product" {
    for_each = var.cloud_instance_bandwidth_expack_product

    content {
      resource_size = bandwidth_expack_product.value["resource_size"]
    }
  }

  dynamic "domain_expack_product" {
    for_each = var.cloud_instance_domain_expack_product

    content {
      resource_size = domain_expack_product.value["resource_size"]
    }
  }

  dynamic "rule_expack_product" {
    for_each = var.cloud_instance_rule_expack_product

    content {
      resource_size = rule_expack_product.value["resource_size"]
    }
  }

  charging_mode         = var.cloud_instance_charging_mode
  period_unit           = var.cloud_instance_period_unit
  period                = var.cloud_instance_period
  auto_renew            = var.cloud_instance_auto_renew
  enterprise_project_id = var.enterprise_project_id
}
```

**Parameter Description**:
- **resource_spec_code**: Resource specification code, assigned by referencing the input variable `cloud_instance_resource_spec_code`, such as "detection" (detection mode) or "premium" (premium edition)
- **bandwidth_expack_product**: Bandwidth expansion package configuration, creates bandwidth expansion packages through dynamic block `dynamic "bandwidth_expack_product"` based on input variable `cloud_instance_bandwidth_expack_product`
  - **resource_size**: Resource size, assigned by referencing the `resource_size` in the input variable
- **domain_expack_product**: Domain expansion package configuration, creates domain expansion packages through dynamic block `dynamic "domain_expack_product"` based on input variable `cloud_instance_domain_expack_product`
  - **resource_size**: Resource size, assigned by referencing the `resource_size` in the input variable
- **rule_expack_product**: Rule expansion package configuration, creates rule expansion packages through dynamic block `dynamic "rule_expack_product"` based on input variable `cloud_instance_rule_expack_product`
  - **resource_size**: Resource size, assigned by referencing the `resource_size` in the input variable
- **charging_mode**: Billing mode, assigned by referencing the input variable `cloud_instance_charging_mode`, such as "prePaid" (prepaid) or "postPaid" (pay-per-use)
- **period_unit**: Subscription period unit, assigned by referencing the input variable `cloud_instance_period_unit`, such as "month" or "year"
- **period**: Subscription period, assigned by referencing the input variable `cloud_instance_period`
- **auto_renew**: Whether to enable auto-renewal, assigned by referencing the input variable `cloud_instance_auto_renew`, such as "true" or "false"
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable `enterprise_project_id`, default is "0" indicating the default enterprise project

### 3. Create WAF Domain

Add the following script to the TF file (such as main.tf) to create a WAF domain:

```hcl
resource "huaweicloud_waf_domain" "test" {
  domain                = var.cloud_domain
  certificate_id        = var.cloud_certificate_id
  certificate_name      = var.cloud_certificate_name
  proxy                 = var.cloud_proxy
  enterprise_project_id = var.enterprise_project_id
  description           = var.cloud_description
  website_name          = var.cloud_website_name
  protect_status        = var.cloud_protect_status
  forward_header_map    = var.cloud_forward_header_map

  dynamic "custom_page" {
    for_each = var.cloud_custom_page

    content {
      http_return_code = custom_page.value["http_return_code"]
      block_page_type  = custom_page.value["block_page_type"]
      page_content     = custom_page.value["page_content"]
    }
  }

  dynamic "timeout_settings" {
    for_each = var.cloud_timeout_settings

    content {
      connection_timeout = timeout_settings.value["connection_timeout"]
      read_timeout       = timeout_settings.value["read_timeout"]
      write_timeout      = timeout_settings.value["write_timeout"]
    }
  }

  dynamic "traffic_mark" {
    for_each = var.cloud_traffic_mark

    content {
      ip_tags     = traffic_mark.value["ip_tags"]
      session_tag = traffic_mark.value["session_tag"]
      user_tag    = traffic_mark.value["user_tag"]
    }
  }

  dynamic "server" {
    for_each = var.cloud_server

    content {
      client_protocol = server.value["client_protocol"]
      server_protocol = server.value["server_protocol"]
      address         = server.value["address"]
      port            = server.value["port"]
      type            = server.value["type"]
      weight          = server.value["weight"]
    }
  }

  depends_on = [
    huaweicloud_waf_cloud_instance.test
  ]
}
```

**Parameter Description**:
- **domain**: Domain name to be protected, assigned by referencing the input variable `cloud_domain`
- **certificate_id**: SSL certificate ID, assigned by referencing the input variable `cloud_certificate_id`, used for HTTPS access
- **certificate_name**: SSL certificate name, assigned by referencing the input variable `cloud_certificate_name`
- **proxy**: Whether to enable proxy, assigned by referencing the input variable `cloud_proxy`, true means enable proxy, false means disable proxy
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable `enterprise_project_id`, default is "0" indicating the default enterprise project
- **description**: Domain description, assigned by referencing the input variable `cloud_description`
- **website_name**: Website name, assigned by referencing the input variable `cloud_website_name`
- **protect_status**: Protection status, assigned by referencing the input variable `cloud_protect_status`, 0 means disable protection, 1 means enable protection
- **forward_header_map**: Field forwarding configuration, assigned by referencing the input variable `cloud_forward_header_map`, used for custom request header forwarding
- **custom_page**: Custom error page configuration, creates custom error pages through dynamic block `dynamic "custom_page"` based on input variable `cloud_custom_page`
  - **http_return_code**: HTTP return code, assigned by referencing the `http_return_code` in the input variable
  - **block_page_type**: Block page type, assigned by referencing the `block_page_type` in the input variable
  - **page_content**: Page content, assigned by referencing the `page_content` in the input variable
- **timeout_settings**: Timeout settings configuration, creates timeout settings through dynamic block `dynamic "timeout_settings"` based on input variable `cloud_timeout_settings`
  - **connection_timeout**: Connection timeout, assigned by referencing the `connection_timeout` in the input variable
  - **read_timeout**: Read timeout, assigned by referencing the `read_timeout` in the input variable
  - **write_timeout**: Write timeout, assigned by referencing the `write_timeout` in the input variable
- **traffic_mark**: Traffic marking configuration, creates traffic marking through dynamic block `dynamic "traffic_mark"` based on input variable `cloud_traffic_mark`
  - **ip_tags**: IP tag list, assigned by referencing the `ip_tags` in the input variable
  - **session_tag**: Session tag, assigned by referencing the `session_tag` in the input variable
  - **user_tag**: User tag, assigned by referencing the `user_tag` in the input variable
- **server**: Origin server configuration list, creates origin server configurations through dynamic block `dynamic "server"` based on input variable `cloud_server`
  - **client_protocol**: Client protocol, assigned by referencing the `client_protocol` in the input variable, such as "HTTP" or "HTTPS"
  - **server_protocol**: Server protocol, assigned by referencing the `server_protocol` in the input variable, such as "HTTP" or "HTTPS"
  - **address**: Origin server address, assigned by referencing the `address` in the input variable, can be an IP address or domain name
  - **port**: Origin server port, assigned by referencing the `port` in the input variable
  - **type**: Origin server type, assigned by referencing the `type` in the input variable, such as "ipv4" or "ipv6"
  - **weight**: Origin server weight, assigned by referencing the `weight` in the input variable, used for load balancing
- **depends_on**: Explicit dependency relationship, ensures that WAF cloud instance is created before WAF domain

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# WAF cloud instance configuration (Required)
cloud_instance_resource_spec_code = "detection"
cloud_instance_charging_mode      = "prePaid"
cloud_instance_period_unit        = "month"
cloud_instance_period             = 1

# WAF domain configuration (Required)
cloud_domain = "demo-example-test.huawei.com"

# Origin server configuration (Required)
cloud_server = [
  {
    client_protocol = "HTTP"
    server_protocol = "HTTP"
    address         = "119.8.0.17"
    port            = 8080
    type            = "ipv4"
    weight          = 1
  }
]
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="cloud_domain=demo-example-test.huawei.com"`
2. Environment variables: `export TF_VAR_cloud_domain=demo-example-test.huawei.com`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating WAF cloud mode domain and related resources
4. Run `terraform show` to view the created WAF cloud mode domain

## Reference Information

- [Huawei Cloud WAF Product Documentation](https://support.huaweicloud.com/intl/en-us/waf/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Cloud Domain](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/waf/cloud-domain)
