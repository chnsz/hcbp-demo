# Deploy Cache Management

## Application Scenario

Content Delivery Network (CDN) cache management is a cache refresh and preheat function provided by the CDN service, used to manage cached content on CDN nodes. Through cache refresh, you can force CDN nodes to delete specified cached content, ensuring users get the latest resources. Through cache preheat, you can pre-cache popular content to CDN nodes, improving user access speed. Automating CDN cache management through Terraform can ensure standardized and consistent cache operations, improving operational efficiency. This best practice will introduce how to use Terraform to automatically execute CDN cache refresh and preheat operations.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [CDN Cache Refresh Resource (huaweicloud_cdn_cache_refresh)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cdn_cache_refresh)
- [CDN Cache Preheat Resource (huaweicloud_cdn_cache_preheat)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cdn_cache_preheat)

### Resource/Data Source Dependencies

```
huaweicloud_cdn_cache_refresh
    └── huaweicloud_cdn_cache_preheat
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create CDN Cache Refresh Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CDN cache refresh resource:

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

# Create CDN cache refresh resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cdn_cache_refresh" "test" {
  count = length(var.refresh_file_urls) > 0 ? 1 : 0

  type                  = "file"
  urls                  = var.refresh_file_urls
  mode                  = "all"
  zh_url_encode         = var.zh_url_encode
  enterprise_project_id = var.enterprise_project_id
}
```

**Parameter Description**:
- **count**: Resource creation condition, create resource when refresh_file_urls list is not empty
- **type**: Refresh type, set to "file" for file refresh
- **urls**: List of file URLs that need to be refreshed, assigned by referencing the input variable refresh_file_urls, supports up to 1000 URLs
- **mode**: Refresh mode, set to "all" to refresh the URL and all content under its directory
- **zh_url_encode**: Whether to encode Chinese characters in URLs, assigned by referencing the input variable zh_url_encode, default value is false
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable enterprise_project_id, default value is "0"

### 3. Create CDN Cache Preheat Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CDN cache preheat resource:

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

# Create CDN cache preheat resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
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

**Parameter Description**:
- **count**: Resource creation condition, create resource when preheat_urls list is not empty
- **urls**: List of URLs that need to be preheated, assigned by referencing the input variable preheat_urls, supports up to 1000 URLs
- **zh_url_encode**: Whether to encode Chinese characters in URLs, assigned by referencing the input variable zh_url_encode, default value is false
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable enterprise_project_id, default value is "0"
- **depends_on**: Explicit dependency relationship, ensuring the cache refresh resource is created before the cache preheat resource

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# CDN Cache Management Configuration
refresh_file_urls = [
  "https://example.com/index.html",
]
preheat_urls = [
  "https://example.com/index.html",
]

zh_url_encode         = false
enterprise_project_id = ""
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="refresh_file_urls=[\"https://example.com/index.html\"]" -var="preheat_urls=[\"https://example.com/index.html\"]"`
2. Environment variables: `export TF_VAR_refresh_file_urls='["https://example.com/index.html"]'` and `export TF_VAR_preheat_urls='["https://example.com/index.html"]'`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to execute CDN cache management operations:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start executing cache refresh and preheat operations
4. Run `terraform show` to view the details of the executed cache management operations

> Note: All URLs must belong to domains that are already configured in CDN. Cache refresh operations typically complete within a few minutes. Cache preheat operations may take longer depending on the number of URLs. Chinese URL encoding is useful when URLs contain Chinese characters. Enterprise project ID is required when using sub-account.

## Reference Information

- [Huawei Cloud CDN Product Documentation](https://support.huaweicloud.com/cdn/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Cache Management](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cdn/cache-management)
