# Samba File Sharing

## 1. Overview

This phase implements a private network file-sharing service using Samba on Debian GNU/Linux 13.

The objective is to provide authenticated file sharing between the Debian server, a Kali Linux client, and a physical Windows client connected through ZeroTier.

## 2. Lab Environment

| Component | Configuration |
|---|---|
| Server OS | Debian GNU/Linux 13 |
| SMB Server | Samba 4.22.11 |
| Linux Client | Kali Linux |
| Windows Client | Windows (Physical Machine) |
| SMB Client | smbclient 4.25.0 |
| SMB Port | TCP 445 |
| Internal Server IP | 10.10.10.10 |
| ZeroTier Server IP | 10.185.225.10 |
| ZeroTier Windows IP | 10.185.225.22 |
| Share Name | Partage |

## 3. Network Architecture

The Samba server is accessible through two network interfaces:

- Internal VirtualBox network (`NSLAB-NET`): used by the Kali client.
- ZeroTier virtual network: used by the physical Windows client.

### Network Addresses

| Device | Network Interface | IP Address |
|---|---|---|
| Debian Server | Internal Network | 10.10.10.10 |
| Kali Client | Internal Network | 10.10.10.20 |
| Debian Server | ZeroTier | 10.185.225.10 |
| Windows Client | ZeroTier | 10.185.225.22 |

## 4. Samba Installation

Samba and its client utilities were installed on Debian:

```bash
sudo apt update
sudo apt install samba smbclient
```

The Samba daemon was enabled and started:

```bash
sudo systemctl enable --now smbd
```

## 5. Shared Directory Configuration

The shared directory was created at:

```text
/srv/samba/Partage
```

Directory ownership and permissions:

```bash
sudo chown root:sambashare /srv/samba/Partage
sudo chmod 2770 /srv/samba/Partage
```

The `sambashare` group controls access to the shared directory.

The SGID bit ensures that newly created files and directories inherit the directory's group.

## 6. Samba User Management

The Linux user `tech1` was added to the `sambashare` group:

```bash
sudo usermod -aG sambashare tech1
```

A separate Samba password was configured:

```bash
sudo smbpasswd -a tech1
```

## 7. Samba Share Configuration

The following section was added to `/etc/samba/smb.conf`:

```ini
[Partage]
   comment = Partage reseau prive
   path = /srv/samba/Partage
   browseable = yes
   read only = no
   guest ok = no
   valid users = @sambashare
   force group = sambashare
   create mask = 0660
   directory mask = 2770
```

Configuration validation:

```bash
sudo testparm -s
```

The Samba service was restarted after configuration:

```bash
sudo systemctl restart smbd
```

## 8. Security and Access Controls

The following controls were implemented:

- Authentication required for share access.
- Guest access disabled for the private share.
- Access restricted to members of the `sambashare` group.
- Read and write access enabled for authorized users.
- Files created through the share receive a maximum permission mask of `0660`.
- Directories created through the share receive a maximum permission mask of `2770`.
- SMB1 was not enabled; modern SMB protocol negotiation was used during testing.

## 9. Validation and Testing

### 9.1 Local Server Test

The share was listed and accessed locally using:

```bash
sudo smbclient -L localhost -U tech1
smbclient //localhost/Partage -U tech1
```

A test file was successfully uploaded to the share.

### 9.2 Kali Linux Client Test

The share was discovered remotely:

```bash
smbclient -L //10.10.10.10 -U tech1
```

Remote access:

```bash
smbclient //10.10.10.10/Partage -U tech1
```

The file `kali-test.txt` was uploaded successfully from Kali to Debian.

### 9.3 Windows Client Test

ZeroTier connectivity was verified:

```powershell
ping 10.185.225.10
```

SMB port accessibility was verified:

```powershell
Test-NetConnection 10.185.225.10 -Port 445
```

The result was:

```text
TcpTestSucceeded : True
```

The share was accessed through Windows File Explorer:

```text
\\10.185.225.10\Partage
```

Windows successfully accessed the existing files and uploaded `windows-test.txt.txt`.

### 9.4 Server-Side Verification

The uploaded files were verified on Debian:

```bash
sudo ls -l /srv/samba/Partage/
```

Observed files:

- `test-samba.txt`
- `kali-test.txt`
- `windows-test.txt.txt`

All three files had the expected `tech1:sambashare` ownership and `660` permissions.

## 10. Current Status

**Completed:** Samba file sharing successfully deployed and tested between Debian, Kali Linux, and Windows over ZeroTier.

## 11. Future Improvements

- Review and harden Samba global configuration.
- Configure firewall rules with interface-specific access restrictions.
- Review Samba authentication and protocol policies.
- Implement centralized logging and monitoring.
- Add additional users and test group-based access restrictions.
