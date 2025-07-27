# Audit-Ready Reporting Automation with OpenSCAP + Ansible

## Overview
Automates compliance scanning for Red Hat-based systems using OpenSCAP and Ansible.

## Features
- CIS or STIG profile scans
- HTML and XML report generation
- Fully automated with Ansible

## Usage
1. Install packages:
   ```bash
   sudo dnf install -y openscap-scanner scap-security-guide
   ```
2. Run:
   ```bash
   ansible-playbook -i inventory.ini openscap_scan.yml
   ```

## Output
- `/tmp/openscap-report.html` (HTML Report)
- `/tmp/openscap-results.xml` (Raw Scan Results)

## License
MIT License
