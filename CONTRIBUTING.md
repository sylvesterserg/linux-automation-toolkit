# Contributing

Thank you for improving the Linux Automation Toolkit! Follow the guidelines below to keep the repository clean and consistent.

## Repository layout
- **Playbooks**: Place Ansible playbooks, inventories, and vars inside `ansible/`.
- **Documentation**: Markdown guides and SOP PDFs belong in `docs/`.
- **Distribution artifacts**: Generated bundles (ZIPs, tarballs, ISOs) belong in `dist/` and should be regenerated rather than edited in-place.

## Coding standards
- Use descriptive playbook filenames (`openscap_scan.yml`, `restart_nginx.yml`).
- Keep variables and task names human-readable and avoid abbreviations that hide intent.
- Add inline comments for any task that requires contextual knowledge.

## Documentation standards
- Update `README.md` when moving or renaming files referenced there.
- Add new SOPs to `docs/README.md` so readers can find them easily.

## Validation
Before opening a pull request:
1. Run `ansible-playbook --syntax-check ansible/<playbook>.yml` for each modified playbook.
2. Spell-check or lint Markdown files when possible.

## Git hygiene
- Create focused branches per feature/fix.
- Squash commits that only fix typos or formatting errors introduced earlier in the branch.
