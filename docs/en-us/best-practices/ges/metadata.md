# Deploy GES Metadata

## Application Scenario

GES (Graph Engine Service) metadata defines the labels and properties of graph data, serving as the foundation for building graph models and importing and analyzing vertex and edge data. With metadata, you can standardize the data structure of vertices and edges in a graph, ensuring consistency in subsequent data import and query operations.

This best practice will introduce how to use Terraform to automatically deploy GES metadata, including creating an OBS bucket for storing the metadata schema file and creating the GES metadata based on that bucket.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Object Storage Bucket (huaweicloud_obs_bucket)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket)
- [Graph Engine Service Metadata (huaweicloud_ges_metadata)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ges_metadata)

### Resource/Data Source Dependencies

```
huaweicloud_obs_bucket
    └── huaweicloud_ges_metadata
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create an OBS Bucket

Add the following script in the TF file (such as main.tf) to create an OBS bucket for storing the GES metadata schema file:

```hcl
# Create an OBS bucket resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "bucket_name" {
  description = "The OBS bucket name for storing GES metadata schema files"
  type        = string
}

resource "huaweicloud_obs_bucket" "test" {
  bucket        = var.bucket_name
  acl           = "private"
  force_destroy = true
}
```

**Parameter Description**:

- **bucket**: The bucket name, assigned by referencing the input variable bucket_name. The name must be globally unique across all Huawei Cloud accounts
- **acl**: The access permission of the bucket, set to `private`, indicating that only the bucket owner can access it
- **force_destroy**: Whether to forcibly delete the bucket, set to `true`, indicating that all objects in the bucket will be deleted when the bucket is deleted

### 3. Create GES Metadata

Add the following script in the TF file (such as main.tf) to create the GES metadata:

```hcl
# Create a GES metadata resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "metadata_name" {
  description = "The GES metadata name"
  type        = string
}

variable "metadata_description" {
  description = "The description of the GES metadata"
  type        = string
}

variable "metadata_schema_file" {
  description = "The schema file name in the OBS bucket"
  type        = string
}

variable "metadata_properties" {
  description = "The properties of the GES metadata label"
  type        = list(any)
  nullable    = false
}

resource "huaweicloud_ges_metadata" "test" {
  name          = var.metadata_name
  description   = var.metadata_description
  metadata_path = "${huaweicloud_obs_bucket.test.bucket}/${var.metadata_schema_file}"

  ges_metadata {
    labels {
      name       = "user"
      properties = var.metadata_properties
    }
  }

  depends_on = [
    huaweicloud_obs_bucket.test
  ]
}
```

**Parameter Description**:

- **name**: The metadata name, assigned by referencing the input variable metadata_name
- **description**: The metadata description, assigned by referencing the input variable metadata_description
- **metadata_path**: The path of the metadata schema file in the OBS bucket, composed of the OBS bucket name and the input variable metadata_schema_file
- **ges_metadata**: The metadata definition block, where `labels` defines the label, `name` is the label name, and `properties` is the label property list, assigned by referencing the input variable metadata_properties
- **depends_on**: Explicitly declares the dependency on the OBS bucket resource, ensuring the metadata is created after the bucket is created

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# OBS bucket configuration
bucket_name = "tf-test-ges-metadata-bucket"

# GES metadata configuration
metadata_name        = "tf_test_ges_metadata"
metadata_description = "This is a demo"
metadata_schema_file = "schema_demo.xml"
metadata_properties  = [{
  dataType    = "char"
  name        = "sex"
  cardinality = "single"
}]
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="bucket_name=my-bucket"`
2. Environment variables: `export TF_VAR_bucket_name=my-bucket`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GES metadata
4. Run `terraform show` to view the created GES metadata

## Reference Information

- [Huawei Cloud Graph Engine Service Product Documentation](https://support.huaweicloud.com/ges/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GES Metadata](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ges/metadata)
