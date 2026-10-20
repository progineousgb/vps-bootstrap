# Contributing

- Keep the cloud-init file and the Ansible roles doing the same thing. A change to one usually needs the other.
- Role variables use the role name as prefix. Use FQCN module names.
- Tasks must be idempotent: a second run should report no changes.
- Before opening a merge request run `cloud-init schema -c cloud-init/user-data.yaml`, `ansible-playbook -i inventory.example.ini --syntax-check site.yml` and `ansible-lint` (from `ansible/`).
