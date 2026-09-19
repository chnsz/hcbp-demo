# Deploy OBS Asset with Authorization

## Application Scenario

Data Security Center (DSC) is a one-stop data security governance service provided by Huawei Cloud, supporting sensitive data identification and classification for data stored in Object Storage Service (OBS). Before using DSC to scan OBS buckets for sensitive data, you need to enable the authorization for the corresponding asset type and add the target OBS bucket as a DSC asset.

This best practice will introduce how to use Terraform to automatically complete the DSC OBS asset authorization and OBS asset addition, including creating an OBS bucket, enabling the DSC OBS asset authorization, and adding the OBS bucket as a DSC asset.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [OBS Bucket (huaweicloud_obs_bucket)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket)
- [DSC Asset Authorization (huaweicloud_dsc_asset_authorization)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dsc_asset_authorization)
- [DSC OBS Asset (huaweicloud_dsc_asset_obs)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dsc_asset_obs)

### Resource/Data Source Dependencies

```
huaweicloud_obs_bucket
    └── huaweicloud_dsc_asset_obs

huaweicloud_dsc_asset_authorization
    └── huaweicloud_dsc_asset_obs
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create an OBS Bucket

Add the following script in the TF file (such as main.tf) to create an OBS bucket:

```hcl
# Create an OBS bucket resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "bucket_name" {
  description = "The name of the OBS bucket to be added as a DSC asset"
  type        = string
}

resource "huaweicloud_obs_bucket" "test" {
  bucket        = var.bucket_name
  acl           = "private"
  force_destroy = true
}
```

**Parameter Description**:

- **bucket**: The OBS bucket name, assigned by referencing the input variable bucket_name
- **acl**: The access control policy of the OBS bucket, set to private for private read and write
- **force_destroy**: Set to true to force delete all objects in the bucket when deleting the bucket

### 3. Enable the DSC OBS Asset Authorization

Add the following script in the TF file (such as main.tf) to enable the DSC OBS asset authorization:

```hcl
# Enable the DSC OBS asset authorization in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_dsc_asset_authorization" "test" {
  type                 = "OBS"
  authorization_status = true
}
```

**Parameter Description**:

- **type**: The asset type, set to OBS to authorize OBS assets
- **authorization_status**: The authorization status, set to true to enable the authorization

### 4. Add the OBS Asset to DSC

Add the following script in the TF file (such as main.tf) to add the OBS bucket as a DSC asset:

```hcl
# Add the OBS asset to DSC in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "asset_name" {
  description = "The name of the DSC OBS asset"
  type        = string
}

resource "huaweicloud_dsc_asset_obs" "test" {
  name          = var.asset_name
  bucket_name   = huaweicloud_obs_bucket.test.bucket
  bucket_policy = "private"

  depends_on = [huaweicloud_dsc_asset_authorization.test]
}
```

**Parameter Description**:

- **name**: The DSC OBS asset name, assigned by referencing the input variable asset_name, and it must be unique among the added OBS assets
- **bucket_name**: The OBS bucket name, assigned by referencing the bucket attribute of the OBS bucket resource
- **bucket_policy**: The OBS bucket policy, which must be consistent with the actual ACL of the OBS bucket, set to private
- **depends_on**: Explicitly declares the dependency to ensure the OBS asset is added after the asset authorization is enabled

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
bucket_name = "tf-test-dsc-obs-bucket"
asset_name  = "tf-test-dsc-obs-asset"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="bucket_name=my-bucket"`
2. Environment variables: `export TF_VAR_bucket_name=my-bucket`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the OBS asset authorization and OBS asset
4. Run `terraform show` to view the created OBS asset authorization and OBS asset

## Reference Information

- [Huawei Cloud Data Security Center Product Documentation](https://support.huaweicloud.com/dsc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DSC OBS Asset with Authorization](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dsc/obs-asset-with-authorization)
