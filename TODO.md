# TODO

## Infrastructure Improvements

### Bastion Security Hardening
- [ ] Refactor `playbooks/01-bastion-security-hardening.yaml` into a reusable role
  - Consider incorporating `playbooks/01-add-known-hosts-to-bastion.yaml` into the role as well
  - Extract SSH hardening tasks to `roles/bastion_security/tasks/ssh.yaml`
  - Extract iptables INPUT chain rules to `roles/bastion_security/tasks/firewall.yaml`
  - Extract Fail2Ban setup to `roles/bastion_security/tasks/fail2ban.yaml`
  - Extract port/upgrade checks to `roles/bastion_security/tasks/verification.yaml`
  - Create defaults for configurable options (SSH port, allowed ports, etc.)
  - Document in `roles/bastion_security/README.md`
  - Update playbook to simply call the role

### TLS File Permissions Config
- [ ] Refactor duplicated file/directory permissions from ensure and assert tasks
  - Extract permissions definitions to defaults/variables for reusability
  - Reduce duplication in file ownership/mode specifications across:
    - `playbooks/41-distribute-tls-to-bastions.yaml` (ensure + assert tasks)
    - `tasks/bootstrap-tls-vault-consul/31_distribute_tls_consul.yaml` (ensure + assert tasks)
    - `tasks/bootstrap-tls-vault-consul/30_distribute_tls_vault.yaml` (ensure + assert tasks)
  - Create a centralized source of truth for TLS file permissions
  - Makes future permission changes simpler and less error-prone
