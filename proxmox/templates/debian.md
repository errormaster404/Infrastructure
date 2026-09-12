## Debian template creation
SSH into your proxmox

```
cd /var/lib/vz/import/

apt install libguestfs-tools  #needed for the virt-customize tool

wget https://cloud.debian.org/images/cloud/trixie/latest/debian-13-generic-amd64.qcow2 #the current Trixie latest cloud image available

sha512sum debian-13-generic-amd64.qcow2  #check against https://cloud.debian.org/images/cloud/trixie/latest/SHA512SUMS


## Customize the image:

virt-customize -a debian-13-generic-amd64.qcow2  --install qemu-guest-agent,curl,wget,nano,rsync,htop,tmux #tools I like to add

virt-customize -a debian-13-generic-amd64.qcow2 --run-command "sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config" #to enable root login which is off by default

virt-customize -a debian-13-generic-amd64.qcow2 --run-command "sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config" #to enable SSH authentication by password - my use choice

qemu-img convert -O qcow2 -c -o preallocation=off debian-13-generic-amd64.qcow2 debian-13-generic-amd64-shrink.qcow2 #convert & compress qcow2 image
```

In PVE host:

- Create a VM 999999 # use {vmid} you wish
- Name: `Template-Debian13` #name I used
- Do not use any media, Qemu Agent (ticked), Remove all disk, 2 cores x86-64-v2-AES, 2048 RAM #settings I chose
- Set to not run on completion

In Host node shell:
```
qm importdisk 999999 debian-13-generic-amd64-shrink.qcow2 Storage --format qcow2 #change 999999 to the actual {vmid} you used above & required Storage location, I used 'Storage'
```

In GUI of VM 999999 (or your {vmid} as above):

- Remove the CD/DVD drive # I don't need one
- Add unused disk as scsi0, ssd on, discard on, Cache write back #my choices, adjust accordingly
- Add cloud-init drive as ide0
- In cloud-init set User as root & password, set IPV4 as dhcp #my choices, adjust accordingly
- In VM Options set boot order scsi0 as first & only boot device
- Convert VM 999999 (or your {vmid} as above) to Template

Done #I make a backup of this VM template once done.

Testing:

- Create a Full clone
- Set Cloud-Init settings of the clone as required & regenerate Cloud-Init image
- Start cloned VM (This will fully setup the VM & should fully Update & Upgrade it)
- Shutdown
Done

Credit:

[PVE9 Create a VM Template for a Debian Trixie Server with Cloud-Init](https://forum.proxmox.com/threads/pve9-create-a-vm-template-for-a-debian-trixie-server-with-cloud-init.170206/)

## TODO
- [x] Template creation
- [ ] Enable 2FA for freeipa users