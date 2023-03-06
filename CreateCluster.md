# Ceph Single Node Cluster Setup

## Pre-req's
1. Any desktop system (Linux, MacOS, Windows)
  - python3
  - podman or docker
2. DHCP on home network
  - using hostname 'node1.home.arpa' as the single node cluster
3. compute device separate from desktop to act as node1
  - old laptop or desktop system (64-bit) capable of running Fedora CoreOS
  - 16GB RAM
  - 250GB harddrive

## Steps
1. Install Fedora CoreOS
  - (desktop) create CoreOS Butane Config file (node1_config.bu)
  - Sample CoreOS Butane config 
```
variant: fcos
version: 1.4.0
passwd:
  users:
    - name: core
      ssh_authorized_keys:
        - ssh-rsa AAAA... <your ssh public key>
storage:
  files:
    - path: /etc/hostname
      mode: 0644
      contents:
        inline: node1.home.arpa
systemd:
  units:
    - name: podman.service
      enabled: yes
    - name: podman.socket
      enabled: yes
```
  - (desktop) Run butane to convert .bu to .ign
```
podman run --interactive --rm quay.io/coreos/butane \
 --strict < node1_config.bu > node1_config.ign
```
  - (desktop) Run temporary http server in the directory where node1_config.ign is located
```
python3 -m http.server 
```
  - Boot node1 from CoreOS ISO (from node1 console)  
NOTE: all commands on node1 need to be run as root user (either become root or use sudo on each command)
  - (node1) install coreos to harddrive
```
coreos-installer install /dev/sda \
 --ignition-url http://desktop.home.arpa:8000/node1_config.ign \
 --insecure-ignition
```
  - Reboot node1 from harddisk (remove iso after shutdown)
```
reboot
```
NOTE: node1.home.arpa must be accessed via ssh with the configured ssh key once the system is up.
  - (desktop) ssh to node1 as core user
```
ssh core@node1
```
2. Install ceph-common ceph-volume cephadm python3
  - (node1) using rpm-ostree to install packages then reboot
```
rpm-ostree install ceph-common ceph-volume cephadm python3
systemctl reboot
```
3. Setup loopback storage backing LVM
  - (node1) issue commands to create LVM storage
```
mkdir /var/osd.disk
fallocate -l 100gib /var/osd.disk/osd.0
fallocate -l 100gib /var/osd.disk/osd.1
losetup /dev/loop0 /var/osd.disk/osd.0
losetup /dev/loop1 /var/osd.disk/osd.1
pvcreate /dev/loop0
pvcreate /dev/loop1
vgcreate osd0 /dev/loop0
vgcreate osd1 /dev/loop1
lvcreate --name data --extents 100%VG osd0
lvcreate --name data --extents 100%VG osd1
```
  - (node1) use systemctl edit to create startup unit makiing the LVM storage available after reboot
```
 systemctl edit losetup.service --force --full
```
  - contents
```
[Unit]
Description=Persistent loop devices for Ceph OSD
DefaultDependencies=no
After=local-fs.target system-udevd.service
Required=systemd-udevd.service

[Service]
Type=oneshot
ExecStart=/usr/sbin/losetup -P /dev/loop0 /var/osd.disk/osd.0
ExecStart=/usr/sbin/losetup -P /dev/loop1 /var/osd.disk/osd.1
ExecStart=/usr/sbin/vgchange -ay osd0
ExecStart=/usr/sbin/vgchange -ay osd1
TimeoutSec=60
RemainAfterExit=n

[Install]
WantedBy=multi-user.target
```
  - (node1) enable the losetup.service unit
```
systemctl enable losetup.service
```
  - (node1) reboot to ensure the LVM volume groups are ready after reboot
```
systemctl reboot
```  
4. Create Ceph Cluster
  - (node1) using cephadm command
```
cephadm bootstrap --allow-fqdn-hostname --single-host-defaults --mon-ip $(getent hosts node1 | awk '{print $1}')
```
NOTE: --mon-ip wont accept hostname, requires IP address  
NOTE: watch for the admin user password displayed at the end  

5. Create OSD's
  - (node1) using ceph-volume command
```
ceph-volume lvm create --data osd0/data --bluestore
ceph-volume lvm create --data osd1/data --bluestore
```
6. Adopt OSD's managed by cephadm
  - (node1) using the cephadm command
```
cephadm adopt --name osd.0 --style legacy
cephadm adopt --name osd.1 --style legacy
```
## Done
A working single-node ceph cluster ready to experiment with.  
https://node1.home.arpa:8443/  
login as admin with the password noted earlier
