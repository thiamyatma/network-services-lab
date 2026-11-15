# Grafana Security Hardening

## 1. Overview

Grafana 13.3.1 was deployed on the Debian network services server to provide centralized monitoring and visualization.

This document records the security hardening performed on November 15, 2026.

## 2. Authentication and Access Control

The following settings were explicitly configured in `/etc/grafana/grafana.ini`.

### User Management

```ini
[users]
allow_sign_up = false
allow_org_create = false
```

- Public user registration disabled.
- Organization creation by users disabled.

### Anonymous Authentication

```ini
[auth.anonymous]
enabled = false
```

- Anonymous access disabled.
- Authentication required to access protected Grafana resources.

## 3. Service Configuration

The Grafana configuration was backed up before modification.

After applying the changes, the Grafana service was restarted.

Service status was verified:

- Service: `grafana-server`
- Expected state: `active (running)`

## 4. Firewall Restrictions

Grafana listens on TCP port `3000`.

Access is restricted through nftables to the authorized Windows client over ZeroTier.

| Parameter | Value |
|---|---|
| Server ZeroTier IP | `10.185.225.10` |
| Authorized client IP | `10.185.225.22` |
| Protocol | TCP |
| Port | `3000` |
| Firewall interface | `ztcdcl3eta` |

The firewall rule allows traffic only when the source is `10.185.225.22` and the destination is `10.185.225.10`.

## 5. Security Validation

The following tests were performed.

### Grafana Health Endpoint

Command:

```bash
curl -s http://127.0.0.1:3000/api/health
```

Result:

- Database status: `ok`
- Grafana version: `13.3.1`

### Unauthenticated API Access

Command:

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" \
http://127.0.0.1:3000/api/user
```

Result:

`HTTP 401`

This confirms that the API endpoint rejects unauthenticated requests.

### Remote Connectivity from Windows

Command:

```powershell
Test-NetConnection 10.185.225.10 -Port 3000
```

Result:

`TcpTestSucceeded : True`

The Grafana login page was also accessed successfully from the Windows client through ZeroTier, and authentication was confirmed.

## 6. Remaining Security Considerations

- Grafana is accessed over HTTP, not HTTPS.
- Access is restricted to the authorized ZeroTier client through nftables.
- HTTPS should be implemented before exposing Grafana through an untrusted network or the public Internet.
- The Grafana administrator account should use a unique, strong password.

## 7. Conclusion

Grafana authentication and user-management settings were hardened successfully.

The health endpoint, unauthenticated API rejection, firewall restriction and remote browser access were validated.

Grafana is operational and remotely accessible through the authorized ZeroTier client.

---

**Project:** Network Services Lab  
**Service:** Grafana 13.3.1  
**Date:** November 15, 2026  
**Focus:** Linux System Administration | Networking | Cybersecurity
