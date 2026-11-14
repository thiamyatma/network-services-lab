# DNS Infrastructure Deployment and Configuration

## 1. Overview

This section documents the deployment, configuration, security, and validation of a DNS infrastructure using BIND9 on Debian GNU/Linux 13.

The objective is to provide centralized name resolution for the laboratory network, including forward and reverse DNS resolution, while allowing clients to resolve external Internet domains.

The DNS client configuration uses NetworkManager and systemd-resolved to implement split DNS routing.

## 2. Lab Environment

| Component | Configuration |
|---|---|
| DNS Server | Debian GNU/Linux 13 |
| DNS Client | Kali Linux |
| DNS Software | BIND9 |
| BIND Version | 9.20.30 |
| DNS Port | UDP/TCP 53 |
| Server Lab IP | `10.10.10.10` |
| Client Lab IP | `10.10.10.20` |
| DNS Domain | `nslab.test` |
| Forward Zone | `nslab.test` |
| Reverse Zone | `10.10.10.in-addr.arpa` |

## 3. DNS Server Installation

BIND9 and its utilities were installed on the Debian server.

The DNS service is enabled and running.

Configuration validation was performed using:

```bash
sudo named-checkconf
```

No configuration errors were reported.

## 4. BIND9 Configuration

### 4.1 Access Control List

An ACL named `agence` defines the networks authorized to query the DNS server and use recursive resolution.

The following networks are authorized:

- `127.0.0.1` — Localhost.
- `10.185.225.0/24` — ZeroTier network.
- `10.10.10.0/24` — Internal laboratory network.

### 4.2 DNS Server Options

The relevant settings in `/etc/bind/named.conf.options` are:

```conf
acl "agence" {
    127.0.0.1;
    10.185.225.0/24;
    10.10.10.0/24;
};

options {
    directory "/var/cache/bind";

    recursion yes;
    allow-query     { agence; };
    allow-recursion { agence; };
    allow-transfer  { none; };

    forwarders { 1.1.1.1; 8.8.8.8; };

    dnssec-validation auto;
    listen-on-v6 { any; };
};
```

Security measures:

| Parameter | Configuration | Purpose |
|---|---|---|
| `allow-query` | `agence` | Restricts DNS queries to authorized networks |
| `allow-recursion` | `agence` | Restricts recursive queries |
| `allow-transfer` | `none` | Prevents unauthorized zone transfers |
| `dnssec-validation` | `auto` | Enables DNSSEC validation |
| `forwarders` | `1.1.1.1`, `8.8.8.8` | Forwards external DNS queries |

## 5. DNS Zones

Two additional DNS zones were configured for the internal laboratory network.

Existing `polytech.local` and ZeroTier reverse DNS zones were preserved.

### 5.1 Forward Lookup Zone

Configuration file:

`/etc/bind/db.nslab.test`

Zone declaration in `/etc/bind/named.conf.local`:

```conf
zone "nslab.test" {
    type master;
    file "/etc/bind/db.nslab.test";
    allow-transfer { none; };
};
```

DNS records:

```dns
$TTL 3600
@   IN  SOA srv.nslab.test. admin.nslab.test. (
            2026111401
            3600
            900
            604800
            300 )

@       IN  NS      srv.nslab.test.

srv     IN  A       10.10.10.10
kali    IN  A       10.10.10.20
```

The forward zone maps hostnames to IPv4 addresses.

| Hostname | IPv4 Address |
|---|---|
| `srv.nslab.test` | `10.10.10.10` |
| `kali.nslab.test` | `10.10.10.20` |

### 5.2 Reverse Lookup Zone

Configuration file:

`/etc/bind/db.10.10.10`

Zone declaration:

```conf
zone "10.10.10.in-addr.arpa" {
    type master;
    file "/etc/bind/db.10.10.10";
    allow-transfer { none; };
};
```

DNS records:

```dns
$TTL 3600
@   IN  SOA srv.nslab.test. admin.nslab.test. (
            2026111401
            3600
            900
            604800
            300 )

@   IN  NS   srv.nslab.test.

10  IN  PTR  srv.nslab.test.
20  IN  PTR  kali.nslab.test.
```

The reverse zone maps IPv4 addresses to domain names.

| IPv4 Address | Reverse DNS Record |
|---|---|
| `10.10.10.10` | `srv.nslab.test` |
| `10.10.10.20` | `kali.nslab.test` |

### 5.3 Zone Validation

Both zones were checked using `named-checkzone`.

```bash
sudo named-checkzone nslab.test /etc/bind/db.nslab.test
```

Result: Zone loaded successfully, serial `2026111401`, status `OK`.

```bash
sudo named-checkzone 10.10.10.in-addr.arpa /etc/bind/db.10.10.10
```

Result: Zone loaded successfully, serial `2026111401`, status `OK`.

After validation, the configuration was reloaded:

```bash
sudo rndc reload
```

Result: `server reload successful`.

## 6. DNS Client Configuration

The Kali Linux client uses two separate network interfaces:

| Interface | Address | Purpose |
|---|---|---|
| `eth0` | `192.168.68.74/22` | Main network and Internet access |
| `eth1` | `10.10.10.20/24` | Internal laboratory network |

### 6.1 NetworkManager Configuration

The internal interface was configured with the Debian DNS server:

```bash
sudo nmcli connection modify eth1 ipv4.dns "10.10.10.10"
```

The DNS routing domain was configured:

```bash
sudo nmcli connection modify eth1 ipv4.dns-search "~nslab.test"
```

To prevent the laboratory interface from becoming the default route:

```bash
sudo nmcli connection modify eth1 ipv4.never-default yes
```

The connection was reactivated:

```bash
sudo nmcli connection up eth1
```

### 6.2 Split DNS Routing

The configuration uses systemd-resolved to route DNS queries according to their domain.

| DNS Query | Interface | DNS Server |
|---|---|---|
| `srv.nslab.test` | `eth1` | `10.10.10.10` |
| `kali.nslab.test` | `eth1` | `10.10.10.10` |
| External domains | `eth0` | Main network DNS servers |
| Tailscale domains | `tailscale0` | Tailscale-managed DNS configuration |

The routing domain `~nslab.test` directs queries for the laboratory domain to the DNS server on `eth1`.

The default Internet route remains on `eth0`.

### 6.3 systemd-resolved Integration

NetworkManager was explicitly configured to use systemd-resolved.

Relevant configuration in `/etc/NetworkManager/NetworkManager.conf`:

```ini
[main]
plugins=ifupdown,keyfile
dns=systemd-resolved

[ifupdown]
managed=false
```

The `/etc/resolv.conf` symbolic link points to:

`/run/systemd/resolve/stub-resolv.conf`

Applications use the local systemd-resolved stub resolver.

## 7. Troubleshooting and Incident Resolution

During the initial client configuration, NetworkManager stored the DNS server correctly, but systemd-resolved did not receive any DNS server information.

### 7.1 Observed Symptoms

The following command showed no DNS servers associated with the network interfaces:

```bash
sudo resolvectl dns
```

NetworkManager had the correct DNS configuration, but communication with systemd-resolved through D-Bus was failing.

### 7.2 Root Cause Investigation

The investigation included:

- NetworkManager DNS configuration inspection.
- systemd-resolved service verification.
- D-Bus service registration inspection.
- NetworkManager and D-Bus journal analysis.
- Verification of the `org.freedesktop.resolve1` service.

The diagnostic evidence indicated that systemd-resolved was running but was not correctly registered on the system D-Bus.

### 7.3 Applied Correction

Restarting systemd-resolved restored its D-Bus registration:

```bash
sudo systemctl restart systemd-resolved
```

NetworkManager then successfully transmitted its DNS configuration.

The final configuration was validated after a complete system reboot.

## 8. Validation and Testing

### 8.1 DNS Server Direct Queries

From Kali Linux, the following queries were executed.

Forward lookup:

```bash
dig @10.10.10.10 srv.nslab.test A
dig @10.10.10.10 kali.nslab.test A
```

Result: Both hostnames returned their expected IPv4 addresses.

Reverse lookup:

```bash
dig @10.10.10.10 -x 10.10.10.10
dig @10.10.10.10 -x 10.10.10.20
```

Result: Both addresses returned their expected PTR records.

### 8.2 Automatic DNS Resolution

After configuring split DNS, queries were tested without explicitly specifying a DNS server.

```bash
resolvectl query srv.nslab.test
resolvectl query kali.nslab.test
```

Result: Both queries were successfully resolved through `eth1`.

The system resolver was also tested:

```bash
getent hosts srv.nslab.test kali.nslab.test
```

Result: Both internal hostnames returned the expected addresses.

### 8.3 External DNS Resolution

External name resolution was tested:

```bash
resolvectl query debian.org
getent hosts google.com
```

Result: External DNS resolution succeeded through `eth0`.

### 8.4 DNS Routing Verification

The effective DNS configuration was inspected:

```bash
resolvectl status
```

Result:

- `eth1` uses `10.10.10.10` for `~nslab.test`.
- `eth0` retains the main network DNS servers.
- `eth1` is not the default route.
- Tailscale retains its own DNS configuration.

### 8.5 Persistence Test

The system was rebooted to verify configuration persistence.

After reboot, the following were confirmed:

- Internal DNS resolution works without specifying a DNS server.
- External DNS resolution continues to work.
- Split DNS routing remains active.
- Tailscale remains functional.
- The internal interface does not become the default route.

## 9. Current Status

**DNS infrastructure: Successfully deployed, configured, and tested.**

The laboratory now provides:

- Authoritative forward DNS resolution.
- Authoritative reverse DNS resolution.
- Controlled recursive DNS resolution.
- Internal split DNS routing.
- External DNS forwarding.
- Persistent client-side DNS configuration.

## 10. Scope and Future Improvements

Potential future improvements include:

- Implementing DHCP-based DNS distribution.
- Adding DNS monitoring and query logging.
- Configuring secondary DNS servers.
- Implementing DNS redundancy and failover.
- Reviewing firewall rules for DNS traffic.
- Monitoring DNS service availability.

---

**Project:** Network Services Lab
**Platform:** Debian 13 / Kali Linux
**Focus:** Linux Administration | Network Services | DNS | Network Security
