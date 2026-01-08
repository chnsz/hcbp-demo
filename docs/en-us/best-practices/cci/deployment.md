# Deploy Stateless Workload

## Application Scenario

Cloud Container Instance (CCI) stateless workload is a container application deployment method provided by the CCI service, used to deploy and manage stateless container applications. By creating stateless workloads, you can achieve automatic scaling, rolling updates, and fault recovery for containers, improving application availability and reliability. Automating stateless workload creation through Terraform can ensure standardized and consistent container application deployment, improving operational efficiency. This best practice will introduce how to use Terraform to automatically create a CCI stateless workload, including namespace creation.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [CCI Namespace Resource (huaweicloud_cciv2_namespace)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cciv2_namespace)
- [CCI Stateless Workload Resource (huaweicloud_cciv2_deployment)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cciv2_deployment)

### Resource/Data Source Dependencies

```
huaweicloud_cciv2_namespace
    └── huaweicloud_cciv2_deployment
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create CCI Namespace Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CCI namespace resource:

```hcl
variable "namespace_name" {
  description = "The name of CCI namespace"
  type        = string
}

# Create CCI namespace resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cciv2_namespace" "test" {
  name = var.namespace_name
}
```

**Parameter Description**:
- **name**: The namespace name, assigned by referencing the input variable namespace_name

### 3. Create CCI Stateless Workload Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CCI stateless workload resource:

```hcl
variable "deployment_name" {
  description = "The name of CCI deployment"
  type        = string
}

variable "instance_type" {
  description = "The instance type of CCI pod"
  type        = string
  default     = "general-computing"
}

variable "container_name" {
  description = "The name of container"
  type        = string
  default     = "c1"
}

variable "container_image" {
  description = "The image of container"
  type        = string
  default     = "alpine:latest"
}

variable "cpu_limit" {
  description = "The CPU limit of container"
  type        = string
  default     = "1"
}

variable "memory_limit" {
  description = "The memory limit of container"
  type        = string
  default     = "2G"
}

variable "image_pull_secret_name" {
  description = "The name of image pull secret"
  type        = string
  default     = "imagepull-secret"
}

# Create CCI stateless workload resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cciv2_deployment" "test" {
  namespace = huaweicloud_cciv2_namespace.test.name
  name      = var.deployment_name

  selector {
    match_labels = {
      app = "template1"
    }
  }

  template {
    metadata {
      labels = {
        app = "template1"
      }

      annotations = {
        "resource.cci.io/instance-type" = var.instance_type
      }
    }

    spec {
      containers {
        name  = var.container_name
        image = var.container_image

        resources {
          limits = {
            cpu    = var.cpu_limit
            memory = var.memory_limit
          }

          requests = {
            cpu    = var.cpu_limit
            memory = var.memory_limit
          }
        }
      }

      image_pull_secrets {
        name = var.image_pull_secret_name
      }
    }
  }

  lifecycle {
    ignore_changes = [
      template.0.metadata.0.annotations,
    ]
  }
}
```

**Parameter Description**:
- **namespace**: The namespace name, referencing the name of the previously created CCI namespace resource (huaweicloud_cciv2_namespace.test)
- **name**: The stateless workload name, assigned by referencing the input variable deployment_name
- **selector.match_labels**: The selector match labels, used to match pods, set to app = "template1"
- **template.metadata.labels**: The pod template labels, must match the selector labels, set to app = "template1"
- **template.metadata.annotations**: The pod template annotations, specify the instance type through the resource.cci.io/instance-type annotation, assigned by referencing the input variable instance_type, default value is "general-computing"
- **template.spec.containers.name**: The container name, assigned by referencing the input variable container_name, default value is "c1"
- **template.spec.containers.image**: The container image, assigned by referencing the input variable container_image, default value is "alpine:latest"
- **template.spec.containers.resources.limits.cpu**: The CPU limit, assigned by referencing the input variable cpu_limit, default value is "1"
- **template.spec.containers.resources.limits.memory**: The memory limit, assigned by referencing the input variable memory_limit, default value is "2G"
- **template.spec.containers.resources.requests.cpu**: The CPU request, assigned by referencing the input variable cpu_limit, default value is "1"
- **template.spec.containers.resources.requests.memory**: The memory request, assigned by referencing the input variable memory_limit, default value is "2G"
- **template.spec.image_pull_secrets.name**: The image pull secret name, assigned by referencing the input variable image_pull_secret_name, default value is "imagepull-secret"

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# CCI Stateless Workload Configuration
deployment_name = "tf-test-deployment"
namespace_name  = "tf-test-namespace"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="deployment_name=test-deployment" -var="namespace_name=test-namespace"`
2. Environment variables: `export TF_VAR_deployment_name=test-deployment` and `export TF_VAR_namespace_name=test-namespace`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create a CCI stateless workload:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the CCI stateless workload
4. Run `terraform show` to view the details of the created CCI stateless workload

> Note: The stateless workload must be created within an existing namespace. The selector labels must match the pod template labels. If you need to use image pull secrets, please create them beforehand.

## Reference Information

- [Huawei Cloud CCI Product Documentation](https://support.huaweicloud.com/cci/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Stateless Workload](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cci/deployment)
