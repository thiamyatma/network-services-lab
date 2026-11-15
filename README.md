# Network Services Lab

**A hands-on network administration and cybersecurity laboratory.**

This project documents the deployment, configuration, testing, and security hardening of essential network services using Debian 13 and Kali Linux in a VirtualBox environment.

## Project Objectives

- Deploy and manage essential Linux network services.
- Understand network protocols and service architectures.
- Configure secure remote administration.
- Test services from separate Linux and Windows clients.
- Apply system administration and network security best practices.
- Implement network traffic filtering and access restrictions.
- Document practical implementations, security audits, and troubleshooting procedures.

## Lab Architecture

| Component | Operating System | Role | IP Address |
|---|---|---|---|
| Server | Debian 13 | Network services host | `10.10.10.10/24` |
| Client | Kali Linux | Testing and administration | `10.10.10.20/24` |
| Remote Client | Windows | Remote administration and testing | `10.185.225.22` |

**Virtual Network:** `NSLAB-NET`
**Network:** `10.10.10.0/24`
**Remote Network:** ZeroTier
**Virtualization:** Oracle VirtualBox

## Technologies

- Debian GNU/Linux 13
- Kali Linux
- Microsoft Windows
- Oracle VirtualBox
- TCP/IP
- OpenSSH
- Samba / SMB3
- BIND9 (DNS)
- Kea DHCPv4
- NFSv4.2
- Chrony / NTP
- Apache2
- MariaDB
- vsftpd / FTPS
- XRDP
- noVNC
- nftables
- ZeroTier
- Linux system administration and network troubleshooting

## Implementation Roadmap

| Phase | Service / Component | Objective | Status |
|---|---|---|---|
| 1 | Network Infrastructure | Establish server-client communication | Completed |
| 2 | SSH | Secure remote administration | Completed |
| 3 | Samba | Authenticated network file sharing | Completed |
| 4 | DNS | Internal and external name resolution | Completed |
| 5 | DHCP | Automatic IPv4 configuration | Completed |
| 6 | NFS | Linux network file sharing | Completed |
| 7 | NTP | Network time synchronization | Completed |
| 8 | Apache2 | Web server and intranet hosting | Completed |
| 8 | MariaDB | Database server deployment and access control | Configured / validation pending |
| 8 | FTPS | Secure file transfer | Completed |
| 8 | XRDP | Remote desktop access | Completed |
| 8 | noVNC | Browser-based remote desktop access | Completed |
| 9 | nftables | Network traffic filtering and firewall hardening | Completed |
| 9 | Security Audit | Service verification and security review | Completed |
| 9 | Backup | Configuration backup and integrity verification | Completed |
| 10 | Monitoring | Centralized service and system monitoring | Planned |

## Network Validation

Initial connectivity between Kali Linux and Debian was validated using ICMP:

```bash
ping -c 4 10.10.10.10
```

**Result:** 4 packets transmitted, 4 received, 0% packet loss.

Average latency: 0.581 ms.

## Repository Structure

```text
network-services-lab/
├── README.md
├── audit/
│   └── PHASE-9-AUDIT.md
├── dhcp/
│   └── README.md
├── dns/
│   └── README.md
├── nfs/
│   └── README.md
├── ntp/
│   └── README.md
├── samba/
├── ssh/
└── .git/
```

## Security Principles

- Apply the principle of least privilege.
- Prefer SSH key-based authentication.
- Restrict network access to authorized clients and interfaces.
- Protect credentials and private keys.
- Apply appropriate file and directory permissions.
- Disable unnecessary services and protocols where appropriate.
- Use firewall rules to restrict inbound network traffic.
- Validate service exposure from authorized and unauthorized clients.
- Maintain configuration backups and verify their integrity.
- Document security configurations, tests, and residual risks.

---

## Implementation Details

### Phase 1 — Network Infrastructure

The Debian server and Kali client have been configured on the same isolated VirtualBox internal network.

Inter-machine connectivity has been successfully validated through ICMP testing.

### Phase 2 — Secure Remote Administration (SSH)

The OpenSSH server has been deployed and validated.

Public-key authentication is configured for two authorized users (`tech1` and `tech2`). Password authentication and direct root login are disabled.

### Phase 3 — Network File Sharing (Samba)

Samba has been deployed on Debian to provide authenticated network file sharing.

A restricted shared directory has been configured with group-based access control and appropriate file and directory permissions.

Access and file operations have been validated from Kali Linux and a Windows client.

### Phase 4 — DNS Infrastructure

BIND9 has been deployed and configured on the Debian server to provide authoritative forward and reverse DNS resolution for the laboratory network.

The DNS infrastructure also supports recursive queries for external domains through configured forwarders.

On the Kali client, NetworkManager and systemd-resolved provide split DNS routing, directing internal laboratory queries to the Debian DNS server while preserving the main network's Internet DNS configuration.

The configuration was validated through direct DNS queries, forward and reverse lookups, external name resolution, and a reboot persistence test.

Detailed documentation: [DNS Infrastructure](dns/README.md)

### Phase 5 — DHCP Infrastructure

Kea DHCPv4 has been deployed on Debian GNU/Linux 13 to provide automatic IPv4 configuration to clients connected to the `NSLAB-NET` laboratory network.

The server listens exclusively on the `enp0s8` interface and dynamically allocates addresses from the `10.10.10.100–10.10.10.200` pool.

DHCP clients receive the internal DNS server (`10.10.10.10`) and the `nslab.test` domain configuration. No default gateway is advertised because the laboratory network is isolated.

The implementation was validated through a real DHCP client test from Kali Linux. Address allocation, DNS option distribution, name resolution, server-side lease logs, and preservation of the primary Internet route were successfully verified.

The original static client configuration was restored after testing.

Detailed documentation: [DHCP Infrastructure](dhcp/README.md)

### Phase 6 — Network File Sharing (NFS)

NFS has been deployed on Debian GNU/Linux 13 to provide Linux-native network file sharing with the Kali Linux client.

The NFS server exports `/srv/nfs/partage` and restricts access to the authorized client `10.10.10.20`.

The export is configured with the following options:

- `rw`: allows read and write operations.
- `sync`: synchronizes write operations before acknowledging them.
- `no_subtree_check`: disables subtree verification.
- `root_squash`: maps remote root requests to an anonymous identity.

The client mounts the shared directory using NFSv4.2:

```bash
sudo mount -t nfs4 -o vers=4.2 \
10.10.10.10:/srv/nfs/partage /mnt/nfs-partage
```

The implementation was validated through:

- Successful NFSv4.2 mounting from Kali Linux.
- File creation and write operations from the client.
- Verification of shared file contents on the Debian server.
- Successful NFS service restart.
- Verification that the export remained available after the restart.
- Confirmation that the Kali client retained access.
- A `root_squash` security test, which correctly denied root write access from the client.

The NFS service is enabled at system startup.

Detailed documentation: [NFS Infrastructure](nfs/README.md)

### Phase 7 — Network Time Synchronization (NTP)

Chrony has been deployed on Debian GNU/Linux 13 as the internal NTP server and on Kali Linux as an NTP client.

The Debian server synchronizes its clock with external NTP sources using the Debian NTP pool:

`2.debian.pool.ntp.org`

The internal client network (`10.10.10.0/24`) is authorized through the Chrony access control directive:

```conf
allow 10.10.10.0/24
```

Kali Linux uses the Debian server (`10.10.10.10`) as its preferred NTP source:

```conf
server 10.10.10.10 iburst prefer
```

The service uses UDP port 123.

The implementation was validated through:

- Successful Chrony installation and service activation on both systems.
- Debian external time synchronization, with a normal leap status.
- Successful authorization of the Kali client.
- Verification of the UDP port 123 listener.
- Selection of `srv.nslab.test` as the Kali synchronization source.
- Confirmation that the Kali system clock is synchronized.
- Verification that Debian recognizes Kali as an NTP client, with no dropped requests in the observed result.
- Alignment of the Debian and Kali timezone configuration to `Africa/Dakar` (UTC+0).
- Firewall filtering validation after nftables deployment.

Detailed documentation: [NTP Infrastructure](ntp/README.md)

### Phase 8 — Additional Network Services and Remote Access

This phase extends the laboratory infrastructure with web hosting, database services, secure file transfer, and remote graphical administration.

**Apache2 — Web Server**

Apache2 has been deployed on Debian to host web content and internal web resources.

Both configured websites were tested from the laboratory network and through the ZeroTier network. HTTP requests returned successful `200 OK` responses.

**MariaDB — Database Server**

MariaDB has been installed and configured on Debian.

Database access is restricted through firewall rules to the authorized remote client `10.185.225.21` on TCP port `3306`.

The firewall restriction was tested against an unauthorized remote client. Final connectivity validation from the authorized client remains pending.

**FTPS — Secure File Transfer**

vsftpd has been deployed to provide FTP over TLS.

Secure file transfer was validated using FileZilla from a remote Windows client through ZeroTier.

The passive FTP data port range is restricted to TCP ports `40000–40100`.

**XRDP — Remote Desktop**

XRDP has been configured to provide remote graphical access to the Debian desktop environment.

A successful remote desktop session was established from Windows using the Remote Desktop Connection client.

**noVNC — Browser-Based Remote Desktop**

noVNC has been deployed to provide browser-based remote desktop access.

Access was successfully tested from Windows through the ZeroTier network.

The noVNC web interface currently uses HTTP and is restricted by firewall rules to the remote network.

### Phase 9 — Firewall Hardening, Backup and Security Audit

The final phase focuses on network access control, configuration backup, service validation, and security documentation.

**nftables — Firewall Configuration**

nftables has been deployed and enabled on Debian GNU/Linux 13.

The firewall applies the following default policies:

- `INPUT`: DROP
- `FORWARD`: DROP
- `OUTPUT`: ACCEPT

Only explicitly authorized traffic is permitted.

Access restrictions are applied according to service, protocol, network interface, and authorized source address.

The configured rules protect SSH, DNS, DHCP, HTTP, NTP, FTPS, Samba, NFS, MariaDB, XRDP, and noVNC.

Examples of access restrictions include:

- NFS access limited to the authorized Kali client (`10.10.10.20`).
- MariaDB access limited to the authorized remote client (`10.185.225.21`).
- Samba and remote administration services restricted to authorized network access.
- DHCP traffic restricted to the laboratory network interface.

The firewall configuration was validated through service connectivity tests from authorized and unauthorized clients.

**Configuration Backup**

A backup of important server configurations was created under:

`/root/network-services-lab-backup-20261115/`

The backup includes a compressed configuration archive and a SHA-256 checksum manifest.

File permissions were restricted to protect the backup contents:

- Backup directory: `700`
- Archive and checksum manifest: `600`

The archive integrity was tested, and its SHA-256 checksum was verified.

**Final Security Audit**

A final audit was performed to review:

- Deployed network services and their operational status.
- Network listeners and exposed ports.
- Firewall policies and access restrictions.
- Authentication and file permissions.
- Network file-sharing security.
- Remote administration services.
- Backup integrity.
- Remaining warnings and security considerations.

No failed systemd units were reported during the audit.

Some residual risks and improvement opportunities remain documented, including the absence of TLS for the noVNC HTTP interface, local-only backup storage, and pending MariaDB connectivity validation from the authorized client.

Full audit report: [Phase 9 — Documentation, Backup and Final Security Audit](audit/PHASE-9-AUDIT.md)

---

## Future Improvements

The following activities are planned for future development:
- Document Grafana security hardening and access control validation ([Grafana Hardening Report](monitoring/GRAFANA-HARDENING.md)).
- Deploy centralized monitoring for system resources and network services.
- Implement service availability checks and alerting.
- Improve remote web administration security with HTTPS.
- Add an off-host backup destination.
- Complete MariaDB remote connectivity validation from the authorized client.
- Review unnecessary network listeners and disable unused services where appropriate.
- Expand the laboratory with additional network security and monitoring tools.

---

**Author:** Ibrahima Yatma Thiam
**Project:** Network Services Lab
**Focus:** Linux System Administration | Networking | Cybersecurity
