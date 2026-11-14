# DHCP Infrastructure with Kea

## 1. Overview

This phase documents the deployment and validation of an IPv4 DHCP server using ISC Kea on Debian GNU/Linux 13.

The objective is to provide automatic IPv4 address allocation and network configuration to clients connected to the isolated `NSLAB-NET` laboratory network.

## 2. Lab Environment

| Component | Configuration |
|---|---|
| DHCP server | Debian GNU/Linux 13 |
| DHCP software | Kea DHCPv4 2.6.3 |
| Server hostname | `srv` |
| Server interface | `enp0s8` |
| Server IP address | `10.10.10.10/24` |
| DHCP client | Kali Linux |
| Client interface | `eth1` |
| Client MAC address | `08:00:27:35:4A:64` |
| Laboratory network | `10.10.10.0/24` |
| DNS server | `10.10.10.10` |
| DNS domain | `nslab.test` |

The Debian server also has NAT and ZeroTier interfaces. Kea is explicitly configured to listen only on `enp0s8`, preventing DHCP service exposure on those networks.

## 3. Network Addressing Plan

| Device / Purpose | IPv4 Address |
|---|---|
| Network address | `10.10.10.0/24` |
| Debian server (`srv`) | `10.10.10.10` |
| Kali static address | `10.10.10.20` |
| DHCP pool | `10.10.10.100 - 10.10.10.200` |
| Broadcast address | `10.10.10.255` |
| Subnet mask | `255.255.255.0` |

The DHCP pool excludes the statically assigned addresses.

No default gateway is distributed because `NSLAB-NET` is an isolated laboratory network and the Debian server is not configured as a router.

## 4. Installation

Update the Debian package index:

```bash
sudo apt update
```

Install the Kea DHCPv4 server:

```bash
sudo apt install kea-dhcp4-server
```

Check the installed version:

```bash
kea-dhcp4 -v
```

The installed package version in this lab is `2.6.3-1+deb13u1`.

## 5. Configuration

The main configuration file is:

`/etc/kea/kea-dhcp4.conf`

A backup of the original configuration was created before replacing the sample configuration:

`/etc/kea/kea-dhcp4.conf.backup`

The active configuration is:

```json
{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": [ "enp0s8" ]
    },

    "control-socket": {
      "socket-type": "unix",
      "socket-name": "/run/kea/kea4-ctrl-socket"
    },

    "lease-database": {
      "type": "memfile",
      "lfc-interval": 3600
    },

    "valid-lifetime": 3600,
    "renew-timer": 900,
    "rebind-timer": 1800,

    "subnet4": [
      {
        "id": 1,
        "subnet": "10.10.10.0/24",
        "pools": [
          {
            "pool": "10.10.10.100 - 10.10.10.200"
          }
        ],
        "option-data": [
          {
            "name": "domain-name-servers",
            "data": "10.10.10.10"
          },
          {
            "name": "domain-name",
            "data": "nslab.test"
          },
          {
            "name": "domain-search",
            "data": "nslab.test"
          }
        ]
      }
    ],

    "loggers": [
      {
        "name": "kea-dhcp4",
        "output-options": [
          {
            "output": "stdout"
          }
        ],
        "severity": "INFO"
      }
    ]
  }
}
```

### Configuration details

| Parameter | Value | Purpose |
|---|---|---|
| Listening interface | `enp0s8` | Restricts DHCP to the lab network |
| Subnet | `10.10.10.0/24` | Defines the managed network |
| Address pool | `10.10.10.100 - 10.10.10.200` | Defines dynamic allocations |
| Valid lifetime | 3600 seconds | Lease duration |
| Renewal timer (T1) | 900 seconds | Client lease renewal |
| Rebinding timer (T2) | 1800 seconds | Client rebinding |
| DNS server | `10.10.10.10` | Internal DNS resolver |
| Domain name | `nslab.test` | DHCP domain option |
| Domain search | `nslab.test` | DNS search suffix |

No router option is configured. Clients therefore do not receive a default gateway from this DHCP server.

The lease database uses Kea's `memfile` backend.

## 6. Configuration Validation

Validate the configuration using the Kea service account:

```bash
sudo -u _kea kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

The validation output confirmed that:

- The `10.10.10.0/24` subnet was loaded.
- The lease timers were accepted.
- Kea selected the raw DHCP socket type.
- The daemon identified `enp0s8` as its listening interface.

## 7. Service Management

Restart the DHCP service:

```bash
sudo systemctl restart kea-dhcp4-server
```

Check its status:

```bash
sudo systemctl status kea-dhcp4-server --no-pager
```

The service was confirmed as:

`Active: active (running)`

Check the DHCP listening socket:

```bash
sudo ss -lunp 'sport = :67'
```

Observed result:

```text
UNCONN  0  0  10.10.10.10:67  0.0.0.0:*  users:(("kea-dhcp4",pid=6161,fd=13))
```

This confirms that Kea listens on UDP port 67 at the laboratory server address.

## 8. DHCP Client Test

The DHCP test was performed from Kali Linux using the `eth1` laboratory interface.

The original static NetworkManager profile was preserved:

| Setting | Original value |
|---|---|
| IPv4 method | `manual` |
| IPv4 address | `10.10.10.20/24` |
| Gateway | None |
| DNS | `10.10.10.10` |
| DNS search domain | `~nslab.test` |
| Never default | `yes` |

A separate NetworkManager profile named `client-dhcp` was used for the test.

Its IPv4 method was already set to `auto`. The `ipv4.never-default` property was enabled to prevent the laboratory interface from replacing the primary default route.

Activate the DHCP profile:

```bash
sudo nmcli connection up client-dhcp
```

Inspect the assigned address:

```bash
ip -4 addr show dev eth1
```

Observed result:

```text
inet 10.10.10.100/24 scope global dynamic eth1
valid_lft 3577sec preferred_lft 3577sec
```

The client successfully received the first available address in the configured DHCP pool.

## 9. DHCP Options Verification

Inspect the network parameters received by the client:

```bash
nmcli device show eth1
```

Observed values:

| DHCP parameter | Received value |
|---|---|
| IPv4 address | `10.10.10.100/24` |
| DNS server | `10.10.10.10` |
| Domain | `nslab.test` |
| DNS search domain | `nslab.test` |
| Default gateway | None |

The DNS resolver was also tested without explicitly specifying the DNS server:

```bash
getent hosts srv.nslab.test
```

Observed result:

```text
10.10.10.10 srv.nslab.test
```

This confirms that the DNS configuration received through DHCP works with the existing BIND9 infrastructure.

## 10. Routing Verification

The client routing table was checked during the DHCP test:

```bash
ip route
```

Observed result:

```text
default via 192.168.68.1 dev eth0
10.10.10.0/24 dev eth1
192.168.68.0/22 dev eth0
```

The primary default route remained on `eth0`. No default route was installed through `eth1`.

## 11. Server-Side Lease Verification

The DHCP server logs were inspected:

```bash
sudo journalctl -u kea-dhcp4-server -n 40 --no-pager
```

The logs confirmed the DHCP exchange:

- `DHCPOFFER` — Kea offered `10.10.10.100`.
- `DHCPREQUEST` — the client requested the offered address.
- `DHCP_LEASE_ALLOC` — Kea allocated the lease.
- `DHCPACK` — Kea acknowledged the allocation.

The server reported:

```text
lease 10.10.10.100 has been allocated for 3600 seconds
```

The client was identified by its MAC address:

```text
08:00:27:35:4A:64
```

## 12. Restoring the Static Client Configuration

After the DHCP test, the original static profile was reactivated:

```bash
sudo nmcli connection up eth1
```

The address was verified:

```bash
ip -4 addr show dev eth1
```

Observed result:

```text
inet 10.10.10.20/24 scope global noprefixroute eth1
valid_lft forever preferred_lft forever
```

DNS resolution was tested again:

```bash
getent hosts srv.nslab.test
```

The server name continued to resolve to `10.10.10.10`.

The separate `client-dhcp` profile remains available for future DHCP tests.

## 13. Security and Design Considerations

- Kea listens only on the dedicated laboratory interface.
- The DHCP pool excludes statically assigned addresses.
- No default gateway is advertised.
- The primary client Internet connection remains on `eth0`.
- The original static client profile is preserved.
- The original Kea configuration is backed up.
- DHCP service logs are available through `systemd-journald`.

## 14. Validation Summary

| Test | Result |
|---|---|
| Kea configuration validation | Passed |
| Kea service startup | Passed |
| UDP port 67 listening | Passed |
| Interface binding to `enp0s8` | Passed |
| DHCP address allocation | Passed |
| DNS option distribution | Passed |
| Domain search distribution | Passed |
| DNS resolution using DHCP settings | Passed |
| No default gateway advertised | Passed |
| Primary Internet route preserved | Passed |
| Server-side lease confirmation | Passed |
| Static client configuration restored | Passed |

## 15. Conclusion

The Kea DHCPv4 server was successfully deployed on Debian GNU/Linux 13 and integrated with the existing laboratory DNS infrastructure.

Kali Linux obtained a dynamic IPv4 address, DNS server, and DNS search domain from Kea. Server-side logs confirmed the complete DHCP allocation and acknowledgement process.

The test was completed without changing the client's primary Internet route. The original static laboratory configuration was successfully restored afterward.

The DHCP infrastructure is operational and validated.
