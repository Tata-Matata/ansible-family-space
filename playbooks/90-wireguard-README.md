# 90-wireguard

This setup provides private administrator access to the cluster through three connected pieces:

- WireGuard provides private network access from an administrator workstation into the internal network.
- Keycloak provides human authentication through OIDC.
- Vault authorizes authenticated administrators to obtain short-lived SSH certificates.

## High-Level Design

The administrator does not use long-lived SSH keys copied to every server.

Instead, the flow is:

1. Connect the workstation to the private VPN with WireGuard.
2. Reach internal services such as Vault and Keycloak over that VPN.
3. Authenticate to Vault with OIDC.
4. Vault delegates human login to Keycloak.
5. After successful login, Vault returns a short-lived token with permission to request an SSH certificate.
6. The administrator submits a local SSH public key to Vault.
7. Vault signs that public key with the cluster SSH CA for a limited time.
8. The administrator uses the local private key together with the signed SSH certificate to connect to the target host.
9. When the certificate expires, SSH access expires automatically and a new certificate must be requested.

## Technical Background

### WireGuard

WireGuard is the entry point into the private environment.

- The bastion acts as the WireGuard server.
- VPN clients receive access to the internal network ranges.
- Private DNS is available over the VPN so internal service names resolve from the administrator workstation.

This keeps Vault, Keycloak, and the target hosts private. They do not need to be exposed publicly for human administration.

### Keycloak

Keycloak is the identity provider for human users.

- The administrator logs in with Keycloak credentials.
- Keycloak returns OIDC identity information to Vault.
- Group membership is used for authorization decisions.

The intended restricted administrator group is `infra-admins`.

### Vault

Vault is the trust broker between human authentication and SSH access.

- Vault trusts Keycloak as its OIDC provider.
- Vault maps successful OIDC login to a Vault role and policy.
- That policy does not need to grant broad Vault administration.
- It only needs permission to request a signed SSH certificate for the approved SSH role.

Vault also owns the SSH certificate authority used for human logins.

### SSH CA Trust on Hosts

Every host that should accept administrator SSH certificates must trust the Vault SSH CA public key.

- The public key of the Vault SSH CA is installed on each host.
- `sshd` is configured to trust that CA for user certificates.
- Because hosts trust the CA, they do not need individual administrator public keys in `authorized_keys`.

## Administrator Workflow

### One-Time Workstation Preparation

1. Install WireGuard.
2. Install the Vault CLI.
3. Ensure an SSH key pair exists on the workstation.
4. Ensure the workstation trusts the internal CA used by private HTTPS services if browser-based OIDC login requires it.

### Routine Login Workflow

1. Connect to the VPN.
2. Confirm internal DNS and routing work.
3. Log in to Vault with OIDC.
4. Complete the Keycloak login flow in the browser.
5. Receive a short-lived Vault token.
6. Ask Vault to sign the workstation SSH public key.
7. Save the returned SSH certificate next to the local private key.
8. SSH to the target host using the local private key and the signed certificate.

### When Access Expires

1. Re-authenticate to Vault if the Vault token has expired.
2. Request a new SSH certificate.
3. Start a new SSH session with the refreshed certificate.

## Why This Model Exists

This model was intended to replace static administrator key distribution.

Benefits:

- Human access stays behind the private VPN.
- Authentication is centralized in Keycloak.
- Authorization is centralized in Vault.
- SSH access is short-lived by default.
- Host trust is simple because each host only trusts one SSH CA.
- Revocation is mostly handled by short TTLs rather than manual key removal on every server.

## Future Reminder Checklist

When revisiting this setup later, verify these pieces in order:

1. WireGuard connectivity from the workstation to the private network.
2. Private DNS resolution for internal services.
3. Keycloak login flow and administrator group membership.
4. Vault OIDC login configuration.
5. Vault SSH signing role and policy.
6. SSH CA public key installed and trusted on all target hosts.
7. Short-lived SSH certificate issuance from Vault.
8. Direct SSH login to a host using the signed certificate.