# NFS Automount Using Systemd

## NFS Client Setup

 Manually mount the NFS target to verify.
```sh
mkdir /data/bile
mount -t nfs bile:/data/bile /data/bile
umount /data/bile
```

### Using Systemd to Automount
```bash

# File: /etc/systemd/system/data-bile.mount

[Unit]
Description=Bile NFS export

[Mount]
What=bile:/data/bile
Where=/data/bile
Type=nfs

[Install]
WantedBy=multi-user.target


File: /etc/systemd/system/data-bile.automount


[Unit]
Description=Bile NFS export

[Automount]
Where=/data/bile
TimeoutIdleSec=1200

[Install]
WantedBy=multi-user.target
```
```sh
sudo systemctl enable --now data-bile.automount
```

## Hyphen in NFS Export Path

If the NFS exported path contains a hyphen (eg. `/data/1-bile`) the systemd `.mount` and `.automount` filenames need to escape the `-` character with their escaped counterpart `\x2d` (eg. `data-1\x2dbile.mount` and `data-1\x2dbile.automount`).

The systemd-escape command can be used to list the correct escaped filename.
```bash
root@mike:/etc/systemd/system# systemd-escape -p --suffix=automount "/data/1-bile"
data-1\x2dbile.automount

root@mike:/etc/systemd/system# systemd-escape -p --suffix=mount "/data/1-bile"
data-1\x2dbile.mount
```
