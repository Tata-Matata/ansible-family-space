# Ansible configuration for Hetzner VMs procured via Terraform + HashiCorp Vault for Kubernetes cluster

## Threat model assumptions

- Compromise of a VM must not compromise the SSH CA
- Compromise of CI must not compromise the SSH CA
- No implicit trust on first use (TOFU)
- No SSH CA private key leaves Vault
- VMs never authenticate to Vault for host signing



# SSH host certificates + host_key_checking = True

## Components

| Component         | Role                   | Location
| ----------------- | ---------------------- |---------
| SSH Host CA       | Signs host public keys | Secure admin machine / CI secret / HashiCorp Vault?
| Trusted CA pubkey | Installed on clients   | Client machines (in ~/.ssh/known_hosts)
| Host key pair     | Generated on VM        | VM
| Host certificate  | Generated at boot      | VM

## Create a SSH Host CA on secure machine

ssh-keygen -t ed25519 -f ssh_host_ca -C "infra-ssh-host-ca"

Will generate
- ssh_host_ca (private CA key)
- ssh_host_ca.pub (public CA key)

## Ansible client (local or CI): Add CA public key to local SSH configuration

```
echo "@cert-authority *.infra.tatamatata.com $(cat ssh_host_ca.pub)" >> ~/.ssh/known_hosts
```

Now, clients trust any host if its hostname matches `` *.infra.tatamatata.com `` AND it presents host certificate signed by this CA

` ~/.ssh/known_hosts ` contains only the CA, no per-host entries, no IPs, no fingerprints, no TOFU, no ssh-keyscan


## Terraform: inject CA private key (for signing the host cert) + metadata into the new VM via user_data


```
resource "hcloud_server" "node" {
  name = "node-1.infra.tatamatata.com"

  user_data = templatefile("cloud-init.yml", {
    ca_pubkey = file("ssh_host_ca")
    hostname  = "node-1.infra.tatamatata.com"
  })
}

```

## cloud-init: generate host key + certificate

Inside VM at first boot - cloud-init.yml

```
packages:
  - openssh-client

write_files:
  - path: /etc/ssh/ssh_host_ca.pub
    permissions: "0644"
    content: ${ca_pubkey}

runcmd:
  # Generate host key (if not exists)
  - ssh-keygen -t ed25519 -f /etc/ssh/ssh_host_ed25519_key -N ""

  # Create host certificate
  - |
    ssh-keygen \
      -s /etc/ssh/ssh_host_ca \
      -I ${hostname} \
      -h \
      -n ${hostname} \
      -V +52w \
      /etc/ssh/ssh_host_ed25519_key.pub

  # Configure sshd to use certificate
  - echo "HostCertificate /etc/ssh/ssh_host_ed25519_key-cert.pub" >> /etc/ssh/sshd_config
  - systemctl restart ssh

```

TO CLARIFY


In real setups, signing is usually done by:

pulling cert from secure endpoint, or

using short-lived bootstrap tokens
(not embedding CA private key — see note below)

Offline CA signs via:

CI pipeline

one-shot provisioning job

secure signing service

VM submits public key → receives certificate

## Ansible 
[defaults]
host_key_checking = True

ADD link to alternatives in Obvious repo


TO CLARIFY 

| Property           | Result        |
| ------------------ | ------------- |
| First-connect MITM | ❌ impossible  |
| Ephemeral nodes    | ✅ seamless    |
| Key rotation       | ✅ automatic   |
| Host rebuilds      | ✅ no warnings |
| Central trust      | ✅ clean       |
| Auditability       | ✅ strong      |

TO CLARIFY

Certificate constraints (often overlooked)

Host certificates can be limited by:

hostname (-n)

validity (-V)

principals

IP ranges (with extensions)

This means:

stolen certs expire

stolen keys alone are useless
---------- END TO CLARIFY


# SSH host CA in HashiCorp Vault

Vault (SSH Host CA)
   |
   |-- signs host public keys
   |
Admin laptop / CI (authenticated to Vault)
   |
   |-- fetch signed host cert
   |
Terraform / Ansible
   |
   |-- install cert + host key onto VM


# HashiCorp Vault and Consul

Provider: Hetzner

##### 2 VMs with private network between then:

- vault-1
- consul-1

##### Consul snapshots are pushed to Hetzner Object Storage
No HA for now (3 Consul VMs for quorum)
Manual Vault init & unseal (for now)

