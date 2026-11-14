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
- SSH
- DNS
- DHCP (Kea)
- Samba / SMB
- NFS
- NTP
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
| NFS | Linux file sharing | Planned |
| NTP | Time synchronization | Planned |
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
├── ntp/
├── firewall/
├── monitoring/
└── screenshots/
```

## Security Principles

- Apply the principle of least privilege.
- Prefer SSH key-based authentication.
- Restrict unnecessary network access.
- Protect credentials and private keys.
- Document security configurations and validation tests.

## Project Status

### Phase 1 — Network Infrastructure

The Debian server and Kali client have been configured on the same isolated VirtualBox internal network. Inter-machine connectivity has been successfully validated.

### Phase 2 — Secure Remote Administration

The OpenSSH server has been deployed and validated. Public-key authentication is configured for two authorized users (`tech1` and `tech2`). Password authentication and direct root login are disabled.

### Phase 3 — DNS Infrastructure

BIND9 has been deployed and configured on the Debian server to provide authoritative forward and reverse DNS resolution for the laboratory network.

The DNS infrastructure also supports recursive queries for external domains through configured forwarders.

On the Kali client, NetworkManager and systemd-resolved provide split DNS routing, directing internal laboratory queries to the Debian DNS server while preserving the main network's Internet DNS configuration.

The configuration was validated through direct DNS queries, forward and reverse lookups, external name resolution, and a reboot persistence test.

Detailed documentation: [DNS Infrastructure](dns/README.md)

### Phase 4 — DHCP Infrastructure

Kea DHCPv4 has been deployed on Debian GNU/Linux 13 to provide automatic IPv4 configuration to clients connected to the `NSLAB-NET` laboratory network.

The server listens exclusively on the `enp0s8` interface and dynamically allocates addresses from the `10.10.10.100–10.10.10.200` pool.

DHCP clients receive the internal DNS server (`10.10.10.10`) and the `nslab.test` domain configuration. No default gateway is advertised because the laboratory network is isolated.

The implementation was validated through a real DHCP client test from Kali Linux. Address allocation, DNS option distribution, name resolution, server-side lease logs, and preservation of the primary Internet route were successfully verified.

The original static client configuration was restored after testing.

Detailed documentation: [DHCP Infrastructure](dhcp/README.md)

---

**Author:** Ibrahima Yatma Thiam  
**Project:** Network Services Lab  
**Focus:** Linux System Administration | Networking | Cybersecurity
