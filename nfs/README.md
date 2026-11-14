# Network File Sharing with NFS

## 1. Overview

This section documents the deployment, configuration, testing, and security validation of Network File System (NFS) on Debian GNU/Linux 13.

The objective is to provide secure Linux-native file sharing between a Debian server and a Kali Linux client using NFSv4.2.

## 2. Lab Environment

| Component | Configuration |
|---|---|
| NFS Server | Debian GNU/Linux 13 |
| NFS Client | Kali Linux |
| Server Hostname | `srv` |
| Server IP | `10.10.10.10` |
| Client IP | `10.10.10.20` |
| Network | `10.10.10.0/24` |
| Virtual Network | `NSLAB-NET` |
| NFS Version | NFSv4.2 |
| Transport Protocol | TCP |
| NFS Port | 2049 |

## 3. NFS Server Installation

The NFS server package was installed on Debian:

```bash
sudo apt update
sudo apt install nfs-kernel-server -y
```

The installation provides the NFS server and the required supporting components.

The service was enabled at system startup:

```bash
sudo systemctl enable nfs-server
```

The service status was verified:

```bash
sudo systemctl is-active nfs-server
sudo systemctl is-enabled nfs-server
```

Expected results:

```text
active
enabled
```

## 4. Shared Directory Configuration

The shared directory was created on the Debian server:

```bash
sudo mkdir -p /srv/nfs/partage
```

Ownership was assigned to the administrative account `tech1`:

```bash
sudo chown tech1:tech1 /srv/nfs/partage
```

Directory permissions were configured:

```bash
sudo chmod 2770 /srv/nfs/partage
```

Permission verification:

```bash
ls -ld /srv/nfs/partage
```

Expected output:

```text
drwxrws--- ... tech1 tech1 ... /srv/nfs/partage
```

The setgid bit ensures that newly created files and directories inherit the directory's group ownership.

## 5. NFS Export Configuration

The NFS export configuration is stored in:

```text
/etc/exports
```

The following export rule was added:

```text
/srv/nfs/partage 10.10.10.20(rw,sync,no_subtree_check,root_squash)
```

### Export Options

| Option | Description |
|---|---|
| `rw` | Allows read and write operations. |
| `sync` | Acknowledges writes after committing them to stable storage. |
| `no_subtree_check` | Disables subtree verification. |
| `root_squash` | Maps remote root requests to an anonymous identity. |

Access is restricted to the Kali client at `10.10.10.20`.

The export configuration was applied:

```bash
sudo exportfs -rav
```

Active exports were verified:

```bash
sudo exportfs -v
```

The expected export includes:

```text
/srv/nfs/partage
        10.10.10.20(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
```

## 6. NFS Server Validation

The NFS server listens on TCP port 2049.

The listening socket was checked with:

```bash
sudo ss -ltnp 'sport = :2049'
```

The available NFS protocol versions were inspected:

```bash
sudo cat /proc/fs/nfsd/versions
```

The server supports NFSv3 and NFSv4.x, including NFSv4.2.

NFSv4.2 was explicitly selected for the client connection.

## 7. NFS Client Configuration

The NFS client utilities were installed on Kali Linux:

```bash
sudo apt update
sudo apt install nfs-common -y
```

The local mount point was created:

```bash
sudo mkdir -p /mnt/nfs-partage
```

The shared directory was mounted using NFSv4.2:

```bash
sudo mount -t nfs4 -o vers=4.2 \
10.10.10.10:/srv/nfs/partage /mnt/nfs-partage
```

The mounted filesystem was verified:

```bash
findmnt -t nfs4
```

Expected configuration includes:

```text
TARGET             SOURCE
/mnt/nfs-partage   10.10.10.10:/srv/nfs/partage
```

The negotiated protocol version was verified as:

```text
vers=4.2
```

## 8. File Sharing Tests

### 8.1. File Creation from Kali

A test file was created from the Kali client:

```bash
touch /mnt/nfs-partage/test-kali.txt
```

The file was listed:

```bash
ls -l /mnt/nfs-partage/
```

### 8.2. Writing Data from Kali

Test content was written to the shared file:

```bash
echo "NFSv4.2 - Test de partage entre Kali et Debian" \
> /mnt/nfs-partage/test-kali.txt
```

### 8.3. Reading Data from Debian

On the Debian server, the file was read directly from the exported directory:

```bash
cat /srv/nfs/partage/test-kali.txt
```

Expected output:

```text
NFSv4.2 - Test de partage entre Kali et Debian
```

This confirmed successful file creation, remote writing, and data visibility between the client and server.

### 8.4. UID/GID Mapping

Both accounts use the same numeric UID and GID:

| System | User | UID | GID |
|---|---|---:|---:|
| Debian | `tech1` | 1000 | 1000 |
| Kali | `yatus` | 1000 | 1000 |

With the default `AUTH_SYS` authentication mechanism, NFS uses numeric UID/GID values to identify users.

Consequently, a file owned by `tech1` on Debian appears owned by `yatus` on Kali because both accounts have matching numeric identifiers.

## 9. Service Restart and Export Persistence

The NFS server service was restarted:

```bash
sudo systemctl restart nfs-server
```

The active exports were checked again:

```bash
sudo exportfs -v
```

The export remained available after the service restart.

The Kali client was also tested after the restart:

```bash
ls -l /mnt/nfs-partage/
```

The existing test file remained accessible.

The service is enabled at system startup. A full server reboot test has not yet been performed.

## 10. Security Validation

### 10.1. Root Squashing

The export uses the `root_squash` option.

To validate this security mechanism, a write operation was attempted from Kali using root privileges:

```bash
sudo touch /mnt/nfs-partage/root-test.txt
```

Observed result:

```text
Permission denied
```

The operation was correctly rejected.

Remote root requests are mapped to an anonymous identity on the NFS server. Since this identity does not have write permissions on the shared directory, the operation fails.

### 10.2. Access Restrictions

The export is restricted to the authorized Kali client:

```text
10.10.10.20
```

The shared directory uses restrictive permissions:

```text
2770
```

These settings implement basic access control and the principle of least privilege.

### 10.3. Security Considerations

- Restrict NFS exports to trusted client IP addresses.
- Avoid exporting sensitive directories unnecessarily.
- Use restrictive filesystem permissions.
- Keep `root_squash` enabled.
- Restrict network access to TCP port 2049.
- Use network segmentation to isolate NFS traffic.
- Remember that `AUTH_SYS` relies on client-supplied UID/GID information and does not encrypt NFS traffic.

For production environments, consider stronger authentication such as Kerberos and network-layer encryption.

## 11. Troubleshooting

### Check NFS Service Status

```bash
sudo systemctl status nfs-server
```

### Display Active Exports

```bash
sudo exportfs -v
```

### Reload Export Configuration

```bash
sudo exportfs -rav
```

### Check NFS Port

```bash
sudo ss -ltnp 'sport = :2049'
```

### Verify Client Mount

```bash
findmnt -t nfs4
```

### Unmount the Shared Directory

```bash
sudo umount /mnt/nfs-partage
```

### Check NFS Client Connectivity

```bash
showmount -e 10.10.10.10
```

Note: `showmount` relies on the mount service and may not provide a complete view of NFSv4-only exports. Use `findmnt` and an actual NFSv4 mount test to validate NFSv4 connectivity.

## 12. Conclusion

The NFS infrastructure was successfully deployed and validated between Debian GNU/Linux 13 and Kali Linux.

The implementation demonstrates:

- NFS server installation and service management.
- Export configuration and client access restrictions.
- NFSv4.2 mounting.
- Remote file creation and data transfer.
- UID/GID-based ownership mapping.
- Service restart and export persistence.
- Root squashing security validation.

The laboratory now provides both SMB-based file sharing through Samba and Linux-native file sharing through NFS.
