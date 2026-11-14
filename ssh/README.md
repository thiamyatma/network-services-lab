# Secure Remote Administration with SSH

## 1. Overview

This section documents the deployment, configuration, security hardening, and validation of the OpenSSH server on Debian GNU/Linux 13.

The objective is to provide secure remote administration from a separate Kali Linux client using public-key authentication.

## 2. Lab Environment

| Component | Configuration |
|---|---|
| Server | Debian GNU/Linux 13 |
| Client | Kali Linux |
| SSH Server | OpenSSH |
| SSH Port | TCP 22 |
| Server Lab IP | `10.10.10.10` |
| Client Lab IP | `10.10.10.20` |
| Authentication | ED25519 public-key authentication |

## 3. SSH Server Configuration

The SSH service is enabled and running on the Debian server.

The effective SSH security configuration was inspected using:

```bash
sudo sshd -T
```

The following settings were verified:

| Parameter | Value | Purpose |
|---|---|---|
| `PermitRootLogin` | `no` | Prevents direct root login |
| `PubkeyAuthentication` | `yes` | Enables public-key authentication |
| `PasswordAuthentication` | `no` | Disables password-based SSH login |
| `KbdInteractiveAuthentication` | `no` | Disables keyboard-interactive authentication |
| `PermitEmptyPasswords` | `no` | Rejects empty passwords |
| `MaxAuthTries` | `3` | Limits authentication attempts |
| `LoginGraceTime` | `30` | Limits authentication time |
| `AllowUsers` | `tech1 tech2` | Restricts SSH access to authorized accounts |

## 4. User Access Management

Two user accounts are authorized to connect remotely:

- `tech1`
- `tech2`

SSH public keys are stored in each user's `authorized_keys` file.

SSH directory permissions were verified:

| Path | Permission |
|---|---|
| `/home/tech1/.ssh` | `700` |
| `/home/tech1/.ssh/authorized_keys` | `600` |
| `/home/tech2/.ssh` | `700` |
| `/home/tech2/.ssh/authorized_keys` | `600` |

ED25519 public keys are used for authentication. Private keys remain on their respective client machines.

## 5. Security Measures

The following security controls were verified:

- Direct root login is disabled.
- Password-based SSH authentication is disabled.
- Public-key authentication is enabled.
- SSH access is restricted to authorized users.
- Authentication attempts are limited.
- SSH key files have restrictive permissions.
- A custom SSH login banner warns against unauthorized access.

## 6. Validation and Testing

### Network connectivity

From Kali Linux:

```bash
ping -c 4 10.10.10.10
```

Result: 4 packets received, 0% packet loss.

### SSH port availability

```bash
nc -vz 10.10.10.10 22
```

Result: TCP port 22 is reachable.

### Public-key authentication

From Kali Linux:

```bash
ssh tech1@10.10.10.10
```

Result: Successful authentication using the ED25519 key.

A remote command was also executed successfully:

```bash
ssh tech1@10.10.10.10 "id && hostname"
```

### Second user validation

The `tech2` account was also tested from another computer.

Result: Successful SSH connection using its own authorized public key.

## 7. Current Status

**SSH remote administration: Successfully deployed and tested.**

The service provides encrypted remote administration with public-key authentication and restricted user access.

## 8. Scope and Future Improvements

Potential future improvements include:

- Reviewing unnecessary SSH forwarding features.
- Applying firewall rules to restrict SSH access to trusted networks.
- Monitoring authentication logs.
- Reviewing SSH configuration periodically.

---

**Project:** Network Services Lab  
**Platform:** Debian 13 / Kali Linux  
**Focus:** Linux Administration | Network Security | SSH Hardening
