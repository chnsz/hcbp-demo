# Frequently Asked Questions

## How to Download and Deploy Local Huawei Cloud Provider

When the local execution machine can normally access the registry.io network, we can download the Huawei Cloud provider to the working directory through the terraform init command. However, for users with network restrictions, using this command may take a long time or even fail. Deploying and loading the Huawei Cloud Provider locally can greatly speed up the initialization process.

You can quickly set up a usable Provider environment locally through the following steps:

1. Download the expected version of the compressed package from [Huawei Cloud Provider repository release list](https://github.com/huaweicloud/terraform-provider-huaweicloud/releases) through other means, and store the version compressed package to the local machine through file transfer and other methods (skip this step if local network download is allowed).

2. Prepare the working directory for the local Provider environment.

- **Linux**:

  ```
  ~/.terraform.d/plugins/<local-registry>/<organization>/huaweicloud/<version>/<os_arch>
  ```

- **Windows**:

  ```
  %APPDATA%\terraform.d\plugins\<local-registry>\<organization>\huaweicloud\<version>\<os_arch>
  ```

- **MacOS**:

  ```
  ~/.terraform.d/plugins/<local-registry>/<organization>/huaweicloud/<version>/<os_arch>
  ```

  **Note:**
  - **local-registry**: The name of the local registry directory, can be customized.
  - **organization**: Organization name, can be customized.
  - **version**: Version number, format is `a.b.c`.
  - **os_arch**: Execution machine architecture type, such as windows_amd64 (Note: 32-bit operating systems are no longer supported). For MacOS, Apple Silicon architecture type is darwin_arm64, Intel is darwin_amd64

3. Extract the compressed package to the working directory prepared in step 2, you can only extract the executable file with the terraform-provider-huaweicloud (prefix) name.

   **Note:** Do not place the compressed package in the working directory prepared in step 2.

4. Linux and MacOS users please check whether the executable file permissions include read and execute permissions for the group where the current operating user belongs.

5. Use the Terraform code block adapted for the local Provider to load the Provider.

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

   If the following error message appears, please recheck the local configuration according to steps 2-5

   ```bash
   Error: Could not retrieve the list of available versions for provider local-registry/huaweicloud/huaweicloud: could not connect to local-registry: Failed to request discovery document: Get "https://local-registry/.well-know/terraform.json": dial tcp: lookup local-registry: no such host
   ```

## How to Generate Detailed Logs During Terraform Execution

Terraform has the ability to print detailed execution requests, responses, debug records and other detailed log information during execution. You need to configure the following two environment variables in the execution environment. After configuration, subsequent command executions will automatically generate log files in the configuration directory.

| Variable Name | Variable Value |
| ------------- | -------------- |
| TF_LOG | DEBUG |
| TF_LOG_PATH | ./terraform.log |

Linux and MacOS users can configure directly through the export command in the command line or bash.rc script:

```bash
$ export TF_LOG=DEBUG
$ export TF_LOG_PATH=./terraform.log
```

Windows users can configure user environment variables or system environment variables in **This PC - Properties - Advanced System Settings - Environment Variables** (the entry for different versions of operating systems will be different, here using Win11 as an example) or configure through PowerShell command line by typing the following commands.

```bash
> $env:TF_LOG="DEBUG"
> $env:TF_LOG_PATH=".\terraform.log"
```

## How to Configure Terraform Command Auto-completion

The auto-completion feature currently only supports bash and zsh. You can enable auto-completion for terraform commands in new sessions by executing the following command.

```bash
$ terraform -install=autocomplete
```
