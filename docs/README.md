# Documentation index

## Standard operating procedures
- [OpenSCAP Ansible Audit SOP (PDF)](sops/OpenSCAP_Ansible_Audit_Automation_SOP.pdf)
- [Self-Healing Automation SOP (PDF)](sops/Self-Healing_Automation_SOP.pdf)

## Guides
- [Automation Hub SOP](automation-hub-sop.md) – How to publish and manage automation content.
- [Proxmox automation reference](proxmox-automation.md) – Migrating Azure Packer workflows to Proxmox with Azure DevOps and Key Vault integration.

## Repository layout notes
- Playbooks live in `ansible/`. Keep future playbooks, inventories, and group vars under this directory so the root stays tidy.
- Use `dist/` for any generated bundles or offline artifacts so that the default Git history remains clean.
