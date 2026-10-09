# Deploy CCE Autopilot Application

## Application Scenario

Cloud Container Engine (CCE) Autopilot is a serverless Kubernetes cluster mode provided by Huawei Cloud, delivering O&M-free container orchestration for cloud-native applications. Autopilot clusters fully manage the control plane and node resources, so you do not need to create or manage nodes; you only need to submit workloads to run containerized applications. In practice, users usually need to upload a packaged Helm chart to the cluster and deploy the application as a release in a specified namespace, completing the release and upgrade of cloud-native applications.

This best practice will introduce how to use Terraform to automatically create a CCE Autopilot cluster, upload a Helm chart, and deploy a release, including VPC and subnet creation, CCE Autopilot cluster creation, chart upload, and release deployment.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [CCE Autopilot Cluster (huaweicloud_cce_autopilot_cluster)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_autopilot_cluster)
- [CCE Autopilot Chart (huaweicloud_cce_autopilot_chart)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_autopilot_chart)
- [CCE Autopilot Release (huaweicloud_cce_autopilot_release)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_autopilot_release)

### Resource/Data Source Dependencies

```
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            └── huaweicloud_cce_autopilot_cluster
                    └── huaweicloud_cce_autopilot_release
                            └── huaweicloud_cce_autopilot_chart
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a VPC

Add the following script in the TF file (such as main.tf) to create a VPC:

```hcl
# Create a VPC in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr

### 3. Create a Subnet

Add the following script in the TF file (such as main.tf) to create a subnet:

```hcl
# Create a subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "subnet_name" {
  description = "The subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
}

variable "gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: The ID of the VPC to which the subnet belongs, assigned by referencing the ID of the VPC created in the previous step
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The CIDR block of the subnet, assigned by referencing the input variable subnet_cidr; when the variable is empty, the subnet CIDR is automatically calculated based on the VPC CIDR
- **gateway_ip**: The gateway IP address of the subnet, assigned by referencing the input variable gateway_ip; when the variable is empty, the gateway IP is automatically calculated based on the subnet CIDR

### 4. Create a CCE Autopilot Cluster

Add the following script in the TF file (such as main.tf) to create a CCE Autopilot cluster:

```hcl
# Create a CCE Autopilot cluster in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "cluster_name" {
  description = "The CCE autopilot cluster name"
  type        = string
}

variable "cluster_version" {
  description = "The Kubernetes version of the cluster"
  type        = string
  default     = "v1.34"
}

variable "cluster_alias" {
  description = "The alias of the cluster"
  type        = string
  default     = ""
}

variable "cluster_description" {
  description = "The description of the cluster"
  type        = string
  default     = ""
}

variable "cluster_category" {
  description = "The cluster type. Only Turbo is supported"
  type        = string
  default     = "Turbo"
}

variable "cluster_type" {
  description = "The master node architecture. Valid values are: VirtualMachine"
  type        = string
  default     = "VirtualMachine"
}

variable "cluster_custom_san" {
  description = "The custom SAN field in the API server certificate of the cluster"
  type        = list(string)
  default     = []
}

variable "cluster_eip_id" {
  description = "The EIP ID of the cluster"
  type        = string
  default     = ""
}

variable "cluster_enable_snat" {
  description = "Whether SNAT is configured for the cluster"
  type        = bool
  default     = false
}

variable "cluster_enable_swr_image_access" {
  description = "Whether SWR image access is enabled for the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_efs" {
  description = "Whether to delete the associated EFS when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_eni" {
  description = "Whether to delete the associated ENI when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_net" {
  description = "Whether to delete the associated network resources when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_obs" {
  description = "Whether to delete the associated OBS when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_sfs_turbo" {
  description = "Whether to delete the associated SFS Turbo when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_lts_reclaim_policy" {
  description = "The LTS reclaim policy. Valid values are: delete, retain"
  type        = string
  default     = "retain"
}

variable "cluster_enterprise_project_id" {
  description = "The ID of the enterprise project to which the cluster belongs"
  type        = string
  default     = "0"
}

variable "cluster_configurations_override" {
  description = "The component configuration items for override"
  type        = list(object({
    name  = string
    value = string
  }))
  default     = []
}

variable "cluster_configurations_override_name" {
  description = "The component name for configurations override"
  type        = string
  default     = ""
}

variable "cluster_tags" {
  description = "The tags of the cluster"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_cce_autopilot_cluster" "test" {
  name        = var.cluster_name
  flavor      = "cce.autopilot.cluster"
  version     = var.cluster_version
  alias       = var.cluster_alias
  description = var.cluster_description
  category    = var.cluster_category
  type        = var.cluster_type
  custom_san  = var.cluster_custom_san
  eip_id      = var.cluster_eip_id

  enable_snat             = var.cluster_enable_snat
  enable_swr_image_access = var.cluster_enable_swr_image_access

  delete_efs   = var.cluster_delete_efs
  delete_eni   = var.cluster_delete_eni
  delete_net   = var.cluster_delete_net
  delete_obs   = var.cluster_delete_obs
  delete_sfs30 = var.cluster_delete_sfs_turbo

  lts_reclaim_policy = var.cluster_lts_reclaim_policy

  host_network {
    vpc    = huaweicloud_vpc.test.id
    subnet = huaweicloud_vpc_subnet.test.id
  }

  container_network {
    mode = "eni"
  }

  eni_network {
    subnets {
      subnet_id = huaweicloud_vpc_subnet.test.ipv4_subnet_id
    }
  }

  extend_param {
    enterprise_project_id = var.cluster_enterprise_project_id
  }

  dynamic "configurations_override" {
    for_each = length(var.cluster_configurations_override) > 0 ? [1] : []

    content {
      name = var.cluster_configurations_override_name

      dynamic "configurations" {
        for_each = var.cluster_configurations_override
        content {
          name  = configurations.value.name
          value = configurations.value.value
        }
      }
    }
  }

  tags = var.cluster_tags
}
```

**Parameter Description**:
- **name**: The cluster name, assigned by referencing the input variable cluster_name
- **flavor**: The cluster flavor, fixed to cce.autopilot.cluster
- **version**: The Kubernetes version of the cluster, assigned by referencing the input variable cluster_version
- **alias**: The cluster alias, assigned by referencing the input variable cluster_alias
- **description**: The cluster description, assigned by referencing the input variable cluster_description
- **category**: The cluster category, assigned by referencing the input variable cluster_category
- **type**: The master node architecture, assigned by referencing the input variable cluster_type
- **custom_san**: The custom SAN field in the API server certificate of the cluster, assigned by referencing the input variable cluster_custom_san
- **eip_id**: The EIP ID of the cluster, assigned by referencing the input variable cluster_eip_id
- **enable_snat**: Whether SNAT is configured for the cluster, assigned by referencing the input variable cluster_enable_snat
- **enable_swr_image_access**: Whether SWR image access is enabled, assigned by referencing the input variable cluster_enable_swr_image_access
- **delete_efs**: Whether to delete the associated EFS when deleting the cluster, assigned by referencing the input variable cluster_delete_efs
- **delete_eni**: Whether to delete the associated ENI when deleting the cluster, assigned by referencing the input variable cluster_delete_eni
- **delete_net**: Whether to delete the associated network resources when deleting the cluster, assigned by referencing the input variable cluster_delete_net
- **delete_obs**: Whether to delete the associated OBS when deleting the cluster, assigned by referencing the input variable cluster_delete_obs
- **delete_sfs30**: Whether to delete the associated SFS Turbo when deleting the cluster, assigned by referencing the input variable cluster_delete_sfs_turbo
- **lts_reclaim_policy**: The LTS reclaim policy, assigned by referencing the input variable cluster_lts_reclaim_policy
- **host_network**: The host network configuration of the cluster, where vpc and subnet are assigned by referencing the IDs of the VPC and subnet created in the previous steps
- **container_network**: The container network configuration of the cluster, where mode is fixed to eni
- **eni_network**: The ENI network configuration of the cluster, where subnet_id is assigned by referencing the IPv4 subnet ID of the subnet created in the previous step
- **extend_param**: The extended parameters of the cluster, where enterprise_project_id is assigned by referencing the input variable cluster_enterprise_project_id
- **configurations_override**: The component configuration override items of the cluster, assigned by referencing the input variables cluster_configurations_override and cluster_configurations_override_name
- **tags**: The cluster tags, assigned by referencing the input variable cluster_tags

### 5. Upload a Helm Chart

Add the following script in the TF file (such as main.tf) to upload a Helm chart:

```hcl
# Upload a Helm chart in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "chart_content" {
  description = "The path of the chart package to be uploaded"
  type        = string
}

variable "chart_parameters" {
  description = "The parameters of the chart"
  type        = string
  default     = "{\"override\":true,\"skip_lint\":true,\"source\":\"package\"}"
}

resource "huaweicloud_cce_autopilot_chart" "test" {
  content    = var.chart_content
  parameters = var.chart_parameters
}
```

**Parameter Description**:
- **content**: The path of the chart package to be uploaded, assigned by referencing the input variable chart_content
- **parameters**: The chart upload parameters, assigned by referencing the input variable chart_parameters

### 6. Deploy a Helm Release

Add the following script in the TF file (such as main.tf) to deploy a Helm release:

```hcl
# Deploy a Helm release in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "release_name" {
  description = "The name of the release"
  type        = string
}

variable "release_namespace" {
  description = "The namespace to deploy the release"
  type        = string
  default     = "default"
}

variable "release_version" {
  description = "The version of the release"
  type        = string
}

variable "release_description" {
  description = "The description of the release"
  type        = string
  default     = ""
}

variable "release_action" {
  description = "The release updating action. Valid values are: upgrade, rollback"
  type        = string
  default     = ""
}

variable "release_image_tag" {
  description = "The image tag"
  type        = string
  default     = "latest"
}

variable "release_image_pull_policy" {
  description = "The image pull policy"
  type        = string
  default     = "IfNotPresent"
}

variable "release_dry_run" {
  description = "Whether to dry run"
  type        = bool
  default     = false
}

variable "release_name_template" {
  description = "The release name template"
  type        = string
  default     = ""
}

variable "release_no_hooks" {
  description = "Whether to disable hooks during installation"
  type        = bool
  default     = false
}

variable "release_replace" {
  description = "Whether to replace the release with the same name"
  type        = bool
  default     = false
}

variable "release_recreate" {
  description = "Whether to rebuild the release"
  type        = bool
  default     = false
}

variable "release_reset_values" {
  description = "Whether to reset values during an update"
  type        = bool
  default     = false
}

variable "release_rollback_version" {
  description = "The version of the rollback release"
  type        = number
  default     = 0
}

variable "release_include_hooks" {
  description = "Whether to enable hooks during an update or deletion"
  type        = bool
  default     = false
}

resource "huaweicloud_cce_autopilot_release" "test" {
  cluster_id  = huaweicloud_cce_autopilot_cluster.test.id
  chart_id    = huaweicloud_cce_autopilot_chart.test.id
  name        = var.release_name
  namespace   = var.release_namespace
  version     = var.release_version
  description = var.release_description
  action      = var.release_action

  values {
    image_tag         = var.release_image_tag
    image_pull_policy = var.release_image_pull_policy
  }

  parameters {
    dry_run         = var.release_dry_run
    name_template   = var.release_name_template
    no_hooks        = var.release_no_hooks
    replace         = var.release_replace
    recreate        = var.release_recreate
    reset_values    = var.release_reset_values
    release_version = var.release_rollback_version
    include_hooks   = var.release_include_hooks
  }

  depends_on = [
    huaweicloud_cce_autopilot_cluster.test,
    huaweicloud_cce_autopilot_chart.test
  ]
}
```

**Parameter Description**:
- **cluster_id**: The ID of the cluster to which the release belongs, assigned by referencing the ID of the CCE Autopilot cluster created in the previous step
- **chart_id**: The ID of the chart used by the release, assigned by referencing the ID of the chart uploaded in the previous step
- **name**: The release name, assigned by referencing the input variable release_name
- **namespace**: The namespace to deploy the release, assigned by referencing the input variable release_namespace
- **version**: The release version, assigned by referencing the input variable release_version
- **description**: The release description, assigned by referencing the input variable release_description
- **action**: The release updating action, assigned by referencing the input variable release_action
- **values**: The values configuration of the release, where image_tag and image_pull_policy are assigned by referencing the input variables release_image_tag and release_image_pull_policy respectively
- **parameters**: The deployment parameters of the release, where each field is assigned by referencing the input variables release_dry_run, release_name_template, release_no_hooks, release_replace, release_recreate, release_reset_values, release_rollback_version, and release_include_hooks respectively

### 7. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through the `tfvars` file, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Cluster variables
vpc_name        = "tf_test_vpc"
subnet_name     = "tf_test_subnet"
cluster_name    = "tf-test-autopilot-cluster"
chart_content   = "./your-chart-version.tgz"
release_name    = "my-release"
release_version = "1.0.0"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of the `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 8. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the CCE Autopilot cluster and deploying the application
4. Run `terraform show` to view the created CCE Autopilot cluster and release

## Reference Information

- [Huawei Cloud Cloud Container Engine Product Documentation](https://support.huaweicloud.com/cce/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For CCE Autopilot Application](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cceautopilot/cce-autopilot-chart-release)
