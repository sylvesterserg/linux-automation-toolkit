# Audit-Ready Reporting Automation with OpenSCAP + Ansible

## Overview
Automates compliance scanning for Red Hat-based systems using OpenSCAP and Ansible.

## Features
- CIS or STIG profile scans
- HTML and XML report generation
- Fully automated with Ansible

## Repository structure
```
├── ansible/           # Playbooks that drive scans and remediations
├── automation-hub/    # Supporting roles, inventories, or task snippets
├── dist/              # Release bundles for offline distribution
├── docs/              # All SOPs and reference guides
└── README.md
```

## Usage
1. Install packages:
   ```bash
   sudo dnf install -y openscap-scanner scap-security-guide
   ```
2. Run the OpenSCAP scan playbook:
   ```bash
   ansible-playbook -i inventory.ini ansible/openscap_scan.yml
   ```

## Output
- `/tmp/openscap-report.html` (HTML Report)
- `/tmp/openscap-results.xml` (Raw Scan Results)

## Additional resources
- Review the [docs index](docs/README.md) for links to SOP PDFs and detailed automation guides.
- [Proxmox automation playbook](docs/proxmox-automation.md) – Guidance for migrating Azure Packer workflows to Proxmox with Azure DevOps and Key Vault integration.

## License
MIT License
