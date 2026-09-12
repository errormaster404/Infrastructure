## How to use ZFS native encryption - preserving all data

```sh
# zfs list
NAME                           USED  AVAIL  REFER  MOUNTPOINT
rpool                         7.44G   892G   104K  /rpool
rpool/ROOT                    2.59G   892G   192K  /rpool/ROOT
rpool/ROOT/pve-1              2.59G   892G  2.42G  /
rpool/data                    3.83G   892G   200K  /rpool/data
rpool/data/subvol-102-disk-0  2.15G   118G  2.10G  /rpool/data/subvol-102-disk-0
rpool/data/vm-100-disk-0      1.68G   892G  1.67G  -
rpool/var-lib-vz               981M   892G   981M  /var/lib/vz

# zfs get encryption
NAME                               PROPERTY    VALUE        SOURCE
rpool                              encryption  off          default
rpool/ROOT                         encryption  off          -
rpool/ROOT/pve-1                   encryption  off          -
rpool/data                         encryption  off          -
rpool/data/subvol-102-disk-0       encryption  off          -
rpool/data/vm-100-disk-0           encryption  off          -
rpool/var-lib-vz                   encryption  off          -

```


## Encrypting the `rpool/ROOT` dataset

Proxmox installs its system inside the `rpool/ROOT` dataset. This is what we will encrypt first.

First, boot into the initramfs. On the startup menu, press `e` to edit the boot argument. Remove `root=ZFS=rpool/ROOT/pve-1 boot=zfs` from the argument and press `enter`.

Load in the `zfs` kernel module:

```sh
modprobe zfs
```

We now need to import the `rpool` ZFS pool so we can start tinkering with it:

```
zpool import -f rpool

zfs snapshot -r rpool/ROOT@copy

zfs send -R rpool/ROOT@copy | zfs receive rpool/copyroot

zfs destroy -r rpool/ROOT

zpool set autotrim=on rpool

zfs create  \
  -o encryption=on -o keyformat=passphrase \
  -o acltype=posix -o xattr=sa -o atime=off -o checksum=blake3 -o overlay=off \ 
  rpool/ROOT
  
  
zfs send -R rpool/copyroot/pve-1@copy | zfs receive -o encryption=on rpool/ROOT/pve-1

zfs destroy -r rpool/copyroot

zfs set mountpoint=/ rpool/ROOT/pve-1

zpool export rpool

reboot -f

```


## Encrypting the `rpool/data` and `rpool/var-lib-vz` dataset

```
mkdir /keys

openssl rand -base64 256 | tr -dc 'A-Za-z0-9' | cut -c -80 > /keys/data.key

openssl rand -base64 256 | tr -dc 'A-Za-z0-9' | cut -c -80 > /keys/var-lib-vz.key

chmod 400 /keys/data.key /keys/vr-lib-vz.key

chattr +i /keys/data.key /keys/var-lib-vz.key


zfs snapshot -r rpool/data@copy 

zfs send -R rpool/data@copy | zfs receive rpool/copydata

zfs destroy -r rpool/data

zfs send -R rpool/copydata@copy | zfs receive -o encryption=on -o keyformat=passphrase -o keylocation=file:///keys/data.key  -o acltype=posix -o xattr=sa -o atime=off -o checksum=blake3 -o overlay=off  rpool/data
zfs destroy -r rpool/copydata
 

zfs snapshot -r rpool/var-lib-vz@copy

zfs send -R rpool/var-lib-vz@copy | zfs receive rpool/copyvar

zfs unmount -f rpool/var-lib-vz

zfs destroy -r rpool/var-lib-vz

zfs send -R rpool/copyvar@copy | zfs receive -o encryption=on -o keyformat=passphrase -o keylocation=file:///keys/var-lib-vz.key  -o acltype=posix -o xattr=sa -o atime=off -o checksum=blake3 -o overlay=off  rpool/var-lib-vz


zfs destroy -r rpool/copyvar

rm -rf /var/lib/vz/*

# Test mount
zfs mount rpool/var-lib-vz
```

 Create service file - `/etc/systemd/system/zfs-load-key.service`
```sh
[Unit]
Description=Load encryption keys
DefaultDependencies=no
After=zfs-import.target
Before=zfs-mount.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/zfs load-key -a

[Install]
WantedBy=zfs-mount.service
```

```
systemctl enable zfs-load-key

systemctl status zfs-load-key
```


```
BEFORE>

NAME                        USED  AVAIL  REFER  MOUNTPOINT
rpool                      28.5G   402G   104K  /rpool
rpool/ROOT                 1.77G   402G   192K  /rpool/ROOT
rpool/ROOT/pve-1           1.77G   402G  1.70G  /
rpool/data                 8.76G   402G    96K  /rpool/data
rpool/data/vm-100-disk-0   2.99G   402G  2.58G  -
rpool/data/vm-1000-disk-0    56K   402G    56K  -
rpool/data/vm-110-disk-0   5.77G   402G  5.77G  -
rpool/var-lib-vz           17.9G   402G  17.9G  /var/lib/vz


AFTER>

NAME                        USED  AVAIL  REFER  MOUNTPOINT
rpool                      28.5G   402G   104K  /rpool
rpool/ROOT                 1.77G   402G   192K  /rpool/ROOT
rpool/ROOT/pve-1           1.77G   402G  1.70G  /
rpool/data                 8.81G   402G   192K  /rpool/data
rpool/data/vm-100-disk-0   3.02G   402G  2.60G  -
rpool/data/vm-1000-disk-0    88K   402G    88K  -
rpool/data/vm-110-disk-0   5.80G   402G  5.80G  -
rpool/var-lib-vz           17.9G   402G  17.9G  /var/lib/vz
```

## Setting up `dropbear` for easy unlock on boot

```
apt-get --assume-yes --no-install-recommends install dropbear-initramfs
```

Edit `/etc/dropbear/initramfs/dropbear.conf`
```
DROPBEAR_OPTIONS="-p 4748 -s -j -k"
```

Edit `/etc/dropbear/initramfs/authorized_keys`
```
no-port-forwarding,no-agent-forwarding,no-x11-forwarding,command="/usr/bin/zfsunlock" <your ssh public key goes here>
```

Edit the `/etc/initramfs-tools/initramfs.conf` and add the static IP address for it to use.
```
#IP=IP Address::Gateway:Netmask:Hostname

IP=10.0.0.1::10.0.0.254:255.255.255.0:dropbear.proxmox.local
```

```
update-initramfs -u
```

Delete unnecessary snapshots
```
zfs destroy rpool/data@copy
zfs destroy rpool/data/vm-100-disk-0@copy
zfs destroy rpool/data/vm-1000-disk-0@copy
zfs destroy rpool/data/vm-110-disk-0@copy
zfs destroy rpool/var-lib-vz@copy
```
## Credit
https://bitgrounds.tech/posts/proxmox-zfs-encryption/

https://privsec.dev/posts/linux/using-native-zfs-encryption-with-proxmox/