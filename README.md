# Network Services Lab

**A hands-on network administration and cybersecurity laboratory.**

This project documents the deployment, configuration, testing, and security hardening of essential network services using Debian 13 and Kali Linux in a VirtualBox environment.

## Project Objectives

- Deploy and manage essential Linux network services.
- Understand network protocols and service architectures.
- Configure secure remote administration.
- Test services from a separate Linux client.
- Apply system administration and network security best practices.
- Document practical implementations and troubleshooting procedures.

## Lab Architecture

| Component | Operating System | Role | IP Address |
|---|---|---|---|
| Server | Debian 13 | Network services host | `10.10.10.10/24` |
| Client | Kali Linux | Testing and administration | `10.10.10.20/24` |

**Virtual Network:** `NSLAB-NET`  
**Network:** `10.10.10.0/24`  
**Virtualization:** Oracle VirtualBox

## Technologies

- Debian GNU/Linux 13
- Kali Linux
- Oracle VirtualBox
- TCP/IP
- OpenSSH
- Samba / SMB
- BIND9 (DNS)
- Kea DHCPv4
- NFSv4.2
- Chrony / NTP
- Linux Firewall
- Network Monitoring

## Implementation Roadmap

| Service | Objective | Status |
|---|---|---|
| Network connectivity | Establish server-client communication | Completed |
| SSH | Secure remote administration | Completed |
| Samba | Network file sharing | Completed |
| DNS | Name resolution | Completed |
| DHCP | Automatic IP configuration | Completed |
| NFS | Linux network file sharing | Completed |
| NTP | Centralized time synchronization | Completed |
| Firewall | Network traffic filtering | Planned |
| Monitoring | Service and system monitoring | Planned |

## Network Validation

Initial connectivity test between Kali Linux and Debian:

```bash
ping -c 4 10.10.10.10
```

**Result:** 4 packets transmitted, 4 received, 0% packet loss.

Average latency: 0.581 ms.

## Repository Structure

```text
network-services-lab/
├── README.md
├── architecture/
├── ssh/
├── samba/
├── dns/
│   └── README.md
├── dhcp/
│   └── README.md
├── nfs/
│   └── README.md
├── ntp/
│   └── README.md
├── firewall/
├── monitoring/
└── screenshots/
```

## Security Principles

- Apply the principle of least privilege.
- Prefer SSH key-based authentication.
- Restrict network access to authorized clients.
- Protect credentials and private keys.
- Apply appropriate file and directory permissions.
- Disable unnecessary services and protocols where appropriate.
- Document security configurations and validation tests.

## Project Status

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

Full operating system reboot persistence and firewall-specific filtering were not tested.

Detailed documentation: [NTP Infrastructure](ntp/README.md)

---

**Author:** Ibrahima Yatma Thiam  
**Project:** Network Services Lab  
**Focus:** Linux System Administration | Networking | Cybersecurity
