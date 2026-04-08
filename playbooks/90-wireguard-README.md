# 90-wireguard

The full administrator access setup is split across two wrappers:

- `playbooks/85-vault-human-auth.yaml` configures Vault human OIDC auth and the SSH-signing roles that depend on Keycloak.
- `playbooks/90-wireguard.yaml` configures the bastion WireGuard server, VPN-side DNS/routing, and basic WireGuard runtime verification.

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

WireGuard authentication uses its own keypairs.

- Each laptop has its own WireGuard private key and public key.
- Bastion has its own WireGuard private key and public key.
- Bastion accepts a VPN peer only if that laptop public key is configured on the server.
- The laptop uses Bastion's WireGuard public key to verify the remote VPN peer.

These WireGuard keys are only for VPN access. They are not SSH keys and are not signed by Vault.

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

Vault does not replace the administrator's SSH keypair.

- The administrator still keeps a normal SSH private key and public key on the workstation.
- The SSH public key is submitted to Vault.
- Vault signs that public key with the cluster SSH CA.
- The resulting SSH certificate is used together with the administrator's existing SSH private key.

### SSH CA Trust on Hosts

Every host that should accept administrator SSH certificates must trust the Vault SSH CA public key.

- The public key of the Vault SSH CA is installed on each host.
- `sshd` is configured to trust that CA for user certificates.
- Because hosts trust the CA, they do not need individual administrator public keys in `authorized_keys`.

This means hosts trust the Vault SSH CA, not the administrator public key directly.

- The administrator proves possession of the SSH private key locally.
- The host verifies that the presented SSH certificate was signed by the trusted Vault SSH CA.
- Access is granted only if both the private key and the signed certificate are valid.

### How Vault SSH Authorization Relates To Linux Permissions

Vault controls whether a user is allowed to obtain an SSH certificate and what kind of certificate can be issued.

That control is separate from the normal Linux authorization model on the target host.

In practice, there are two layers:

1. Vault decides whether the administrator may receive a signed SSH certificate for a specific SSH role.
2. The target Linux host decides what that logged-in Unix user is allowed to do after login.

Vault SSH roles can restrict certificate issuance in ways such as:

- which Unix username is allowed in the certificate
- which Unix username becomes the default username
- how long the certificate remains valid
- which SSH signing endpoint the authenticated user is allowed to call

That means Vault can prevent a user from obtaining a certificate for an arbitrary Unix account.

For example, if the Vault SSH role allows only the Unix account `ansible`, then Vault will not issue a valid certificate for `root`.

But after login, Linux still applies its own permissions:

- file ownership and file permissions
- group membership
- `sudo` rules
- PAM and other local access controls

So a Vault-signed certificate does not bypass Linux authorization.

It only proves that:

- Vault authorized the user to log in as the Unix account encoded into the SSH certificate
- the user possesses the matching SSH private key

Whether that Unix account is effectively an administrator depends on Linux configuration on the host.

If the Unix account has broad `sudo` rights, then the user may become root after login through normal Linux privilege escalation.
If the Unix account does not have such rights, the signed certificate alone does not grant root access.

This separation is intentional:

- Vault controls who may get short-lived login credentials and for which Unix account.
- Linux controls what that Unix account may actually do on the host.

So the system is designed to avoid handing out direct root SSH identity by default.
Instead, Vault can issue a short-lived certificate for a limited Unix account, and any further privilege escalation remains subject to host-level policy.

## Authentication Layers And Key Material

There are three separate trust layers in this setup.

### 1. WireGuard Authentication

Purpose: get the workstation onto the private network.

- Key material: WireGuard keypair per laptop.
- Verified by: Bastion WireGuard server configuration.
- Result: network-level access to private services over the VPN.

### 2. OIDC Authentication

Purpose: prove the human user's identity to Vault.

- Key material: no local SSH or WireGuard keys are used for this step.
- Verified by: Keycloak credentials and OIDC flow.
- Result: a short-lived Vault token tied to the human auth role.

### 3. SSH Authentication With Vault-Signed Certificate

Purpose: log in to a target host.

- Key material: the workstation's normal SSH keypair plus a short-lived SSH certificate returned by Vault.
- Verified by: the target host's trusted Vault SSH CA public key.
- Result: SSH login as the allowed Unix account for the configured role, subject to that host's normal Linux permissions and `sudo` policy.

## Administrator Workflow

### One-Time Workstation Preparation

1. Install WireGuard.
2. Install the Vault CLI.
3. Generate a WireGuard key pair for the workstation and register its public key on the bastion.
4. Ensure a separate SSH key pair exists on the workstation for host access.
5. Ensure the workstation trusts the internal CA used by private HTTPS services if browser-based OIDC login requires it.

### Routine Login Workflow

1. Connect to the VPN.
2. Confirm internal DNS and routing work.
3. Log in to Vault with OIDC.
4. Complete the Keycloak login flow in the browser.
5. Receive a short-lived Vault token.
6. Ask Vault to sign the workstation SSH public key.
7. Save the returned SSH certificate next to the local SSH private key.
8. SSH to the target host using the local SSH private key and the signed certificate.

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