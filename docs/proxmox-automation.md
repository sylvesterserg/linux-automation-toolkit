# Proxmox Automation Playbook

This playbook outlines how to transition existing Azure-focused image automation to an on-premises Proxmox VE environment while preserving Azure DevOps-driven workflows and secure secret management through Azure Key Vault.

## 1. Architecture Overview

| Component | Purpose |
| --- | --- |
| Proxmox VE Cluster | Hosts VM templates and production workloads. |
| HashiCorp Packer | Builds immutable VM templates for Proxmox using the existing Azure image definitions as a baseline. |
| Ansible | Configures guest operating systems post-provisioning (optional if existing roles are available). |
| Azure DevOps Pipelines | Orchestrates image builds, template publication, and environment roll-outs. |
| Azure Key Vault | Stores sensitive variables (API tokens, service principals, certificates). |
| Git Repositories | Hold Packer templates, Ansible playbooks, and pipeline definitions. |

## 2. Transition Strategy from Azure to Proxmox

1. **Inventory current assets**
   - Export the Azure Packer template definitions (JSON/HCL) and identify reusable provisioning scripts.
   - Catalogue any Azure-specific builders (e.g., `azure-arm`) and post-processors.

2. **Map Azure-specific features to Proxmox equivalents**
   - Replace the Azure builder with the [`proxmox` Packer builder](https://developer.hashicorp.com/packer/plugins/builders/proxmox/ve) or a community plugin that targets Proxmox VE.
   - Swap Azure Managed Image publication steps for Proxmox template creation and optional [content library](https://pve.proxmox.com/wiki/Storage) syncs.

3. **Refactor variables**
   - Normalize build variables (e.g., VM size, storage pool, ISO path) into a shared `variables.pkr.hcl` file.
   - Externalize secrets to Azure Key Vault-backed variable groups in Azure DevOps.

4. **Validate builds locally**
   - Run Packer builds against a Proxmox development cluster using service accounts with limited privileges.
   - Capture golden images as Proxmox templates (e.g., `proxmox_template` post-processor).

## 3. Proxmox Template Creation Workflow

1. **Prerequisites**
   - Proxmox VE API token with `PVEVMAdmin` role on the target node or resource pool.
   - SSH key pair for provisioning.
   - ISO images uploaded to a Proxmox storage accessible to the builder.

2. **Sample Packer Configuration (HCL)**

   ```hcl
   packer {
     required_plugins {
       proxmox = {
         version = ">= 1.0.0"
         source  = "github.com/hashicorp/proxmox"
       }
     }
   }

   variable "proxmox_url" {}
   variable "token_id" {}
   variable "token_secret" {}

   source "proxmox-clone" "ubuntu" {
     proxmox_url   = var.proxmox_url
     username      = var.token_id
     token         = var.token_secret
     template_name = "ubuntu-22.04-cloudinit"
     node          = "pve01"
     pool          = "golden-images"
     ssh_username  = "packer"
   }

   build {
     name    = "ubuntu-2204-template"
     sources = ["source.proxmox-clone.ubuntu"]

     provisioner "shell" {
       inline = [
         "sudo apt-get update",
         "sudo apt-get -y dist-upgrade",
         "sudo cloud-init clean"
       ]
     }

     post-processor "shell-local" {
       inline = [
         "echo Template build complete"
       ]
     }
   }
   ```

3. **Template Hardening**
   - Integrate OpenSCAP/Ansible roles from this repository to enforce security baselines.
   - Run CIS/STIG remediation during the build to guarantee compliance.

## 4. Azure DevOps Pipeline Integration

1. **Repository Layout**
   - `/packer` – HCL templates and scripts.
   - `/ansible` – Roles/playbooks for post-provisioning.
   - `.azure-pipelines/` – YAML definitions for CI/CD.

2. **Pipeline Stages**
   1. `lint` – Validate HCL, Ansible, and YAML syntax (`packer fmt`, `ansible-lint`).
   2. `build` – Execute `packer build` against Proxmox using a self-hosted agent with network access to the cluster.
   3. `publish` – Convert build artifacts to Proxmox templates (e.g., `qm template <vmid>`), tag them, and optionally trigger replication.
   4. `deploy` (optional) – Kick off Terraform/Ansible jobs to instantiate VMs from templates.

3. **Sample Azure Pipeline Snippet**

   ```yaml
   trigger:
     branches:
       include: [ main ]

   variables:
     - group: Proxmox-Automation-Secrets  # linked to Azure Key Vault

   stages:
     - stage: BuildTemplate
       jobs:
         - job: packer
           pool:
             name: SelfHosted-Proxmox
           steps:
             - task: Bash@3
               displayName: Install Packer
               inputs:
                 targetType: inline
                 script: |
                   curl -fsSL https://releases.hashicorp.com/packer/${PACKER_VERSION}/packer_${PACKER_VERSION}_linux_amd64.zip -o packer.zip
                   unzip -o packer.zip -d $HOME/bin
             - task: Bash@3
               displayName: Build Template
               env:
                 PROXMOX_URL: $(proxmoxUrl)
                 PROXMOX_TOKEN_ID: $(proxmoxTokenId)
                 PROXMOX_TOKEN_SECRET: $(proxmoxTokenSecret)
               inputs:
                 targetType: inline
                 script: |
                   packer init packer
                   packer build -var proxmox_url=$PROXMOX_URL \
                     -var token_id=$PROXMOX_TOKEN_ID \
                     -var token_secret=$PROXMOX_TOKEN_SECRET \
                     packer/ubuntu-2204.pkr.hcl
   ```

## 5. Azure Key Vault Integration

1. **Create Key Vault secrets** for Proxmox API tokens, SSH keys, and any third-party credentials.
2. **Link Key Vault to Azure DevOps** through a Variable Group with `Azure Key Vault` reference.
3. **Grant access policies** allowing the Azure DevOps service principal `Get`/`List` permissions.
4. **Reference secrets** in pipelines using `$(secretName)` syntax, ensuring no secrets are committed to Git.

## 6. Deployment Pipeline (Optional Enhancement)

- Use Terraform with the [Telmate Proxmox provider](https://registry.terraform.io/providers/Telmate/proxmox/latest) to provision VMs from templates.
- Trigger Terraform from Azure DevOps once a template is published.
- Apply Ansible automation (e.g., OpenSCAP audits, configuration hardening) post-deployment for drift detection.

## 7. Operational Runbook

1. **Routine template refresh** every patch cycle (monthly) to maintain security posture.
2. **Automated testing** of templates via integration tests (Serverspec/Testinfra) to validate services and hardening controls.
3. **Versioning** templates with semantic tags (e.g., `ubuntu-2204-v2024.05.01`) and maintain a changelog.
4. **Rollback plan** leveraging previous template snapshots stored in Proxmox or an offsite backup location.

---

By following this playbook, teams can reuse their Azure image automation patterns while leveraging Proxmox VE for on-premises workloads, all while keeping secrets secure and deployment processes traceable in Azure DevOps.
