# TODO

## Infrastructure Improvements

### Bastion Security Hardening
- [ ] Refactor `playbooks/01-bastion-security-hardening.yaml` into a reusable role
  - Extract SSH hardening tasks to `roles/bastion_security/tasks/ssh.yaml`
  - Extract iptables INPUT chain rules to `roles/bastion_security/tasks/firewall.yaml`
  - Extract Fail2Ban setup to `roles/bastion_security/tasks/fail2ban.yaml`
  - Extract port/upgrade checks to `roles/bastion_security/tasks/verification.yaml`
  - Create defaults for configurable options (SSH port, allowed ports, etc.)
  - Document in `roles/bastion_security/README.md`
  - Update playbook to simply call the role
