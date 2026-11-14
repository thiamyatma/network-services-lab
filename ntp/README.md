# Network Time Synchronization with Chrony (NTP)

## 1. Overview

This section documents the deployment and validation of a centralized Network Time Protocol (NTP) service using Chrony.

The Debian 13 server acts as the internal NTP server. It synchronizes its clock with external NTP sources and distributes time to authorized clients on the internal laboratory network.

Kali Linux is configured as an NTP client and synchronizes its system clock with the Debian server.

## 2. Lab Environment

| Component | Configuration |
|---|---|
| NTP server | Debian GNU/Linux 13 |
| NTP server hostname | srv |
| NTP server IP | 10.10.10.10 |
| NTP client | Kali Linux |
| NTP client hostname | kali |
| NTP client IP | 10.10.10.20 |
| Internal network | 10.10.10.0/24 |
| NTP protocol | NTP |
| Transport protocol | UDP |
| NTP port | 123 |
| Time synchronization software | Chrony |

## 3. Architecture

The Debian server synchronizes with external NTP sources and provides time synchronization to authorized clients on the internal network.

```text
        External NTP Sources
                 |
                 | UDP/123
                 |
        +-------------------+
        | Debian 13 Server  |
        | srv               |
        | 10.10.10.10       |
        | Chrony NTP Server |
        +-------------------+
                 |
                 | UDP/123
                 |
        +-------------------+
        | Kali Linux        |
        | kali              |
        | 10.10.10.20       |
        | Chrony NTP Client |
        +-------------------+
```

## 4. Debian NTP Server Configuration

### 4.1 Installation

Chrony was installed on Debian:

```bash
sudo apt update
sudo apt install chrony -y
```

Installed version:

- Chrony: 4.6.1-3+deb13u2

The installation replaced systemd-timesyncd because both services manage system time synchronization.

### 4.2 External NTP Sources

The default Debian Chrony configuration uses the Debian NTP pool:

```conf
pool 2.debian.pool.ntp.org iburst
```

The `iburst` option accelerates the initial synchronization process.

### 4.3 Allowing Internal Clients

The following directive was added to `/etc/chrony/chrony.conf`:

```conf
allow 10.10.10.0/24
```

This authorizes clients from the internal laboratory subnet to synchronize with the server.

Chrony access control was tested:

```bash
sudo chronyc accheck 10.10.10.20
```

Observed result:

```text
208 Access allowed
```

### 4.4 Service Management

Chrony was enabled and started:

```bash
sudo systemctl enable --now chrony
```

The service was restarted after configuration changes:

```bash
sudo systemctl restart chrony
```

### 4.5 Server Validation

The server synchronization status was checked with:

```bash
chronyc tracking
```

The server reported a valid synchronization state:

```text
Stratum    : 2
Leap status: Normal
```

The NTP listener was checked with:

```bash
sudo ss -lunp 'sport = :123'
```

Observed result:

```text
UNCONN 0 0 0.0.0.0:123 0.0.0.0:* users:(("chronyd",...))
```

Chrony listens on UDP port 123 on IPv4 interfaces. Client access is controlled by the Chrony ACL.

## 5. Kali Linux NTP Client Configuration

### 5.1 Installation

Chrony was installed on Kali:

```bash
sudo apt update
sudo apt install chrony -y
```

Installed version:

- Chrony: 4.9-2

### 5.2 Configure the Debian Server as the Time Source

The default external pool entry in `/etc/chrony/chrony.conf`:

```conf
pool 2.debian.pool.ntp.org iburst
```

was replaced with:

```conf
server 10.10.10.10 iburst prefer
```

This configures the Debian server as the preferred time source.

The configuration was checked with:

```bash
sudo chronyd -p
```

No configuration errors were reported.

### 5.3 Enable Chrony

The client service was enabled and started:

```bash
sudo systemctl enable --now chrony
```

### 5.4 Verify the Selected NTP Source

The following command was used:

```bash
chronyc sources -v
```

Observed source:

```text
^* srv.nslab.test
```

The `^*` indicator means that the source was selected as the current synchronization source.

### 5.5 Verify System Synchronization

The system clock status was checked:

```bash
timedatectl
```

Observed status:

```text
System clock synchronized: yes
NTP service: active
```

The client timezone was configured as:

```text
Africa/Dakar (GMT, +0000)
```

The Debian server and Kali client therefore display local time using the same timezone.

## 6. Client Verification on Debian

The Debian server was queried for connected NTP clients:

```bash
sudo chronyc clients
```

Observed result:

```text
Hostname          NTP   Drop   Int   IntL   Last
kali.nslab.test     6      0     5     -      38
```

This confirms that the Kali client communicated with the Debian NTP server.

- `NTP`: number of NTP requests received from the client.
- `Drop`: number of client requests dropped.
- `Last`: time since the last client activity, in seconds.

No requests were dropped in the observed result.

## 7. Timezone Configuration

NTP synchronizes the system clock; it does not force all machines to display the same local time.

The timezone determines how the synchronized time is displayed.

The Debian server uses:

```text
Africa/Dakar (UTC+0)
```

Kali was initially configured with:

```text
Europe/Paris (UTC+1)
```

To align local time display, Kali was changed to:

```bash
sudo timedatectl set-timezone Africa/Dakar
```

The final verification confirmed:

```text
Time zone: Africa/Dakar (GMT, +0000)
System clock synchronized: yes
NTP service: active
```

## 8. Security Considerations

- Restrict NTP client access to trusted internal networks.
- Avoid exposing UDP port 123 unnecessarily to untrusted networks.
- Use reliable external time sources.
- Monitor synchronization status and Chrony service health.
- Consider redundant internal NTP servers for production environments.
- Protect time sources against unauthorized modification.

The Chrony `allow` directive restricts which client networks can use the server for time synchronization.

## 9. Validation Summary

| Test | Expected result | Observed result |
|---|---|---|
| Chrony server installation | Successful | Passed |
| Chrony client installation | Successful | Passed |
| Debian external synchronization | Synchronized | Passed |
| Internal client authorization | Access allowed | Passed |
| UDP 123 listener | Listening | Passed |
| Kali NTP source selection | Debian selected | Passed |
| Kali system synchronization | Yes | Passed |
| Debian client monitoring | Kali visible | Passed |
| Client request drops | 0 | Passed |
| Timezone alignment | Africa/Dakar | Passed |

## 10. Limitations and Future Improvements

The following items were not validated during this deployment:

- Full operating system reboot persistence.
- Firewall-specific UDP/123 filtering.
- Redundant internal NTP server operation.
- NTP authentication between server and clients.

These items can be addressed in future improvements.

## 11. Conclusion

Chrony was successfully deployed as a centralized time synchronization service in the laboratory.

The Debian 13 server synchronizes with external NTP sources and provides time to the Kali Linux client over the internal network.

The client synchronization, server access control, UDP listener, and timezone configuration were successfully validated.

This deployment demonstrates the role of centralized time synchronization in reliable system administration, network operations, log correlation, and cybersecurity investigations.
