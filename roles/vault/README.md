# vault

Installs and configures HashiCorp Vault with iptables firewall rules and systemd service management.

## Requirements

- systemd-based Linux distribution
- iptables for firewall management
- Vault binary already installed or path specified

## Role Variables

See `group_vars/all.yaml` for complete variable definitions. Key variables:

### Vault Configuration
- `vault_user`: System user for Vault process
- `vault_group`: System group for Vault process
- `vault_config_dir`: Directory for Vault configuration files
- `vault_binary_path`: Path to Vault binary
- `vault_api_port`: API port (default: 8200)
- `vault_cluster_port`: Cluster port (default: 8201)

### TLS Configuration
- `vault_tls_enabled`: Enable/disable TLS mode (boolean)
- `consul_tls_enabled`: Enable/disable Consul backend TLS (boolean)
- `vault_tls_dir`: Directory for TLS certificates
- `vault_server_cert_path`: Path to server certificate
- `vault_server_key_path`: Path to server private key
- `vault_ca_cert_path`: Path to CA certificate

### Backend Configuration
- Vault uses Consul as storage backend
- Consul connection settings derived from `consul_api_addr`

## Dependencies

- Role: `systemd_service` (for service management)

## Tasks

The role executes tasks in this order:
1. Firewall rules (iptables) - allow bastion and consul access
2. User/group creation
3. Directory structure creation
4. Vault binary installation
5. Configuration file deployment (from template)
6. Systemd service installation and management

## Example Usage

```yaml
- name: Set Vault TLS mode
  hosts: vault
  tasks:
    - set_fact:
        vault_tls_enabled: false
        consul_tls_enabled: false

- name: Install and configure Vault
  hosts: vault
  become: true
  roles:
    - vault
```

## Configuration Template

The role uses `vault.hcl.j2` template which conditionally renders:
- HTTP vs HTTPS listener based on `vault_tls_enabled`
- Consul backend connection with optional TLS
- Cluster addressing
- UI enablement

## Behavior

- **First install**: Vault is started with initial configuration
- **Reconfiguration**: If config changes, Vault is restarted
- **No changes**: Service state is verified (idempotent)

## Health Check

After service start, the role waits for Vault API to respond on the health endpoint with acceptable status codes (200, 429, 501).

## License

Proprietary
