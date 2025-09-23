# How to Install Terraform

This document will introduce how to install Terraform on different operating systems.

## Pre-installation Preparation

Before installing Terraform, please ensure your system meets the following requirements:

- 64-bit operating system
- Supported operating systems: Linux, Windows, MacOS
- Network connection (for downloading installation packages)

## Linux System Installation

### Ubuntu/Debian System

1. **Update system package index**
   ```bash
   sudo apt-get update
   sudo apt-get install -y gnupg software-properties-common curl
   ```

2. **Add HashiCorp GPG key**
   ```bash
   wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
   ```

3. **Add official HashiCorp Linux repository**
   ```bash
   echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
   ```

4. **Update and install**
   ```bash
   sudo apt update
   sudo apt install terraform
   ```

### CentOS/RHEL System

1. **Install yum-config-manager**
   ```bash
   sudo yum install -y yum-utils
   ```

2. **Add HashiCorp repository**
   ```bash
   sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
   ```

3. **Install Terraform**
   ```bash
   sudo yum -y install terraform
   ```

## Windows System Installation

### Method 1: Using Chocolatey Package Manager (Recommended)

1. **Install Chocolatey** (if not already installed)
   ```powershell
   Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
   ```

2. **Install Terraform using Chocolatey**
   ```powershell
   choco install terraform
   ```

### Method 2: Manual Installation

1. Visit [Terraform Download Page](https://developer.hashicorp.com/terraform/downloads)
2. Download the Windows version zip file
3. Extract the file to a specified directory (e.g., C:\terraform)
4. Add this directory to the system environment variable Path:
   - Right-click "This PC" → "Properties" → "Advanced System Settings" → "Environment Variables"
   - Find "Path" in "System Variables"
   - Click "Edit" → "New"
   - Add the Terraform directory path (e.g., C:\terraform)
   - Click "OK" to save settings

## MacOS System Installation

### Method 1: Using Homebrew (Recommended)

1. **Install Homebrew** (if not already installed)
   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```

2. **Install Terraform**
   ```bash
   brew tap hashicorp/tap
   brew install hashicorp/tap/terraform
   ```

### Method 2: Manual Installation

1. Download the MacOS version zip file
2. Extract the file
3. Move the terraform binary to system path:
   ```bash
   sudo mv terraform /usr/local/bin/
   ```

## Verify Installation

After installation, please verify that Terraform is correctly installed:

```bash
terraform version
```

If installation is successful, you will see output similar to:
```
Terraform v1.x.x
```

## Post-installation Configuration

### 1. Configure Command Line Auto-completion (Optional)

For Bash users:
```bash
terraform -install-autocomplete
```

### 2. Verify Installation Environment

Create a test directory and initialize:
```bash
mkdir terraform-test
cd terraform-test
terraform init
```

## Reference Information

- [Terraform Installation Guide](https://learn.hashicorp.com/tutorials/terraform/install-cli)
- [Terraform Official Download Page](https://developer.hashicorp.com/terraform/downloads)
