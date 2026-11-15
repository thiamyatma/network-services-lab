# Phase 9 — Documentation, Backup and Final Security Audit

**Project:** Network Services Lab  
**Audit Date:** November 15, 2026  
**Server:** Debian GNU/Linux 13 (`srv`)  
**Linux Client:** Kali Linux (`yatus`)  
**Virtualization:** Oracle VirtualBox

---

## 1. Objectives

The purpose of this phase is to consolidate the network laboratory and assess its operational and security status.

The audit covers:

- Verification of deployed network services.
- Network access restrictions and firewall policies.
- Service availability and startup persistence.
- Configuration backup and integrity verification.
- Listening ports and active services.
- Identification of residual risks and hardening recommendations.

## 2. Network Architecture

| Component | IP Address | Role |
|---|---|---|
| Debian Server | `10.10.10.10/24` | Network services host |
| Kali Linux | `10.10.10.20/24` | Testing and administration client |
| Debian — ZeroTier | `10.185.225.10/24` | Remote administration |
| Windows — ZeroTier | `10.185.225.22` | Remote client |
| Laboratory Network | `10.10.10.0/24` | NSLAB-NET |

## 3. Deployed Services and Validation Results

| Service | Technology | Result |
|---|---|---|
| Remote administration | OpenSSH | Validated |
| Network file sharing | Samba / SMB3 | Validated |
| DNS resolution | BIND9 | Validated |
| Dynamic IP allocation | Kea DHCPv4 | Validated |
| Linux file sharing | NFSv4.2 | Validated |
| Time synchronization | Chrony / NTP | Validated |
| Web server | Apache2 | HTTP 200 OK |
| Database server | MariaDB | Configuration audited; final remote test pending |
| Secure file transfer | FTPS / vsftpd | Validated with FileZilla |
| Remote desktop | XRDP | Windows connection validated |
| Browser-based VNC | noVNC | Windows connection validated |
| Network filtering | nftables | Validated |

## 4. Firewall Configuration — nftables

A restrictive firewall policy was implemented using nftables.

Default policies:

- INPUT: DROP
- FORWARD: DROP
- OUTPUT: ACCEPT

The firewall also allows established and related connections and applies interface-based and source-address-based access restrictions.

### Authorized inbound services

| Port | Protocol | Service | Access restriction |
|---|---|---|---|
| 22 | TCP | SSH | Laboratory LAN and ZeroTier |
| 53 | TCP/UDP | DNS | Laboratory LAN and ZeroTier |
| 67 | UDP | DHCP | Laboratory interface |
| 80 | TCP | Apache HTTP | Laboratory LAN and ZeroTier |
| 123 | UDP | NTP | Laboratory LAN |
| 21 | TCP | FTPS | ZeroTier |
| 40000–40100 | TCP | FTPS passive mode | ZeroTier |
| 445 | TCP | Samba | ZeroTier |
| 2049 | TCP | NFS | Kali client `10.10.10.20` |
| 3306 | TCP | MariaDB | Authorized client `10.185.225.21` |
| 3389 | TCP | XRDP | ZeroTier |
| 6080 | TCP | noVNC | ZeroTier |

The firewall configuration was syntax-checked and successfully loaded.

The nftables service was verified as:

- Enabled at system startup.
- Active at the time of the final audit.
- Configured with restrictive inbound filtering.

## 5. Functional Validation Tests

### 5.1 DNS — BIND9

- Internal DNS resolution validated from Kali Linux.
- DNS resolution validated from Windows over ZeroTier.
- Forward and reverse DNS zone configurations checked.
- BIND configuration and zone files validated.

### 5.2 DHCP — Kea DHCPv4

- Complete DHCP DORA exchange observed: Discover, Offer, Request and ACK.
- Dynamic allocation of `10.10.10.100` verified.
- DNS server and domain options verified.
- Lease persistence enabled through `persist: true`.
- Original Kali static network configuration restored after testing.

### 5.3 Web Server — Apache2

- Application virtual host returned HTTP 200 OK.
- Intranet virtual host returned HTTP 200 OK.
- Access validated from the laboratory network and ZeroTier.

### 5.4 Samba — SMB3

- Authentication using the `tech1` account validated.
- File creation and reading tested from Windows.
- Shared directory permissions and group-based access control verified.

### 5.5 NFS — NFSv4.2

- NFSv4.2 mount successfully established from Kali Linux.
- Read and write operations validated.
- `root_squash` behavior tested.
- NFSv3 disabled.

### 5.6 NTP — Chrony

- Chrony synchronization validated.
- Kali selected `srv.nslab.test` as its synchronization source.
- System clock synchronization confirmed.

### 5.7 FTPS — vsftpd

- TLS-protected connection validated using FileZilla.
- File transfer functionality confirmed.
- Passive FTP port range configured.

### 5.8 Remote Desktop — XRDP

- TCP connectivity to port 3389 validated.
- Windows Remote Desktop connection successfully established.
- XFCE desktop session verified.

### 5.9 Browser-based Remote Access — noVNC

- noVNC service confirmed active.
- TCP connectivity to port 6080 validated.
- Browser-based VNC connection successfully established from Windows over ZeroTier.

## 6. Configuration Backup and Integrity Verification

A configuration backup was created on the Debian server.

Backup directory:

`/root/network-services-lab-backup-20261115/`

Backup files:

- `configurations.tar.gz`
- `SHA256SUMS`

The archive includes selected configurations for:

- nftables
- Kea DHCP
- BIND9
- Apache2
- vsftpd
- Samba
- NFS
- MariaDB
- XRDP
- noVNC systemd service

### Backup security

| Component | Permissions |
|---|---|
| Backup directory | `700` |
| Archive | `600` |
| SHA-256 manifest | `600` |

The archive was successfully tested and its SHA-256 checksum was verified.

SHA-256:

`569adff720600bce65acde03d994a8893a9daf19d49910f7bb7e265591613179`

The backup is currently stored locally on the Debian server.

## 7. Service and Listening Port Audit

Listening sockets were inspected and compared against the nftables ruleset.

The audit identified additional RPC/NFS and NetBIOS-related listeners. Their inbound ports are not authorized by the current firewall policy and are therefore blocked by the default DROP policy.

The final systemd failure check reported:

`0 loaded units listed.`

No failed systemd units were reported during the final check.

## 8. Residual Risks and Hardening Recommendations

| Observation | Recommendation |
|---|---|
| RETBleed CPU vulnerability warning | Review CPU mitigations, host microcode and available updates |
| vmwgfx virtual graphics warning | Review the VirtualBox virtual graphics controller |
| Additional RPC listeners | Evaluate whether all RPC components are required |
| nmbd and TCP port 139 listeners | Consider disabling NetBIOS if it is not required |
| noVNC uses HTTP | Consider HTTPS through a reverse proxy |
| MariaDB remote access | Perform the final connectivity test from `10.185.225.21` |
| Local-only backup | Maintain a second encrypted copy outside the server |

These observations do not invalidate the successful functional tests, but they represent opportunities for further security hardening.

## 9. Final Assessment

Phase 9 successfully consolidated the laboratory documentation, validated the deployed network services, implemented restrictive nftables filtering, verified firewall startup persistence and completed a configuration backup with SHA-256 integrity verification.

The laboratory now provides a functional and documented network services infrastructure with controlled remote access and tested service configurations.

Additional hardening and a final remote MariaDB connectivity test remain recommended.

---

**Project:** Network Services Lab  
**Author:** Ibrahima Yatma Thiam  
**Focus:** Linux System Administration | Networking | Cybersecurity
