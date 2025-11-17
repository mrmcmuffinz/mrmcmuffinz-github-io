+++
title = "KVM + Libvirt Setup Guide"
date = "2025-11-16T20:30:25-06:00"
draft = false
tags = ["kvm", "libvirt", "qemu", "cloud-init", "virtualization"]
categories = ["Homelab", "DevOps"]
+++

This guide outlines a complete, reproducible workflow for setting up a virtual machine using QEMU/KVM with Libvirt on Ubuntu. It includes steps for configuring a bridged network with Netplan, preparing cloud-init for automated VM provisioning, addressing file permission issues, and accessing the VM using SSH.

---

## 1. Install KVM, QEMU, and Libvirt

Update your system and install required virtualization packages:

```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients virtinst bridge-utils cloud-image-utils
```

Add your user to the necessary groups:

```bash
sudo usermod -aG libvirt,kvm $USER
```

> Log out and back in (or reboot) to apply group changes.

Verify virtualization support:

```bash
lscpu | grep -i virtualization
lsmod | grep kvm
qemu-system-x86_64 --version
```

---

## 2. Prepare Directories and Download Cloud Image

```bash
mkdir -p ~/muffins/{images,cloudinit}
```

Navigate and download the Ubuntu cloud image:

```bash
cd ~/muffins/images
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img -O ubuntu-24.04-cloudimg.qcow2
```

Create a working copy for the VM:

```bash
cp ubuntu-24.04-cloudimg.qcow2 muffins-vm01.qcow2
```

---

## 3. Configure Bridged Networking Using Netplan

Use this when NetworkManager doesn't manage your Ethernet interface.

### 3.1 Create the Netplan file

```bash
sudo nano /etc/netplan/01-br0.yaml
```

Paste the following (update `eno1` if needed):

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eno1:
      dhcp4: false
      dhcp6: false
  bridges:
    br0:
      interfaces: [eno1]
      dhcp4: true
      dhcp6: false
      parameters:
        stp: false
        forward-delay: 0
```

### 3.2 Apply changes

```bash
sudo netplan apply
```

### 3.3 Verify network

```bash
ip a
bridge link
```

Expected: `eno1` has no IP, `br0` receives a LAN IP.

---

## 4. Fix Permissions Using ACL

Allow Libvirt to traverse your home directory and VM folders:

```bash
sudo setfacl -m u:libvirt-qemu:--x /home/$USER
sudo setfacl -R -m u:libvirt-qemu:r-x /home/$USER/muffins
```

---

## 5. Configure Cloud-Init

### `user-data`

```yaml
#cloud-config
hostname: muffins-vm01
users:
  - name: abraham
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: users, admin
    home: /home/abraham
    shell: /bin/bash
    lock_passwd: false
    ssh_authorized_keys:
      - ssh-ed25519 AAAA...yourkeyhere
chpasswd:
  expire: false
```

### `meta-data`

```yaml
instance-id: muffins-vm01
local-hostname: muffins-vm01
```

### Create the seed ISO

```bash
cd ~/muffins/cloudinit
sudo cloud-localds muffins-vm01-seed.iso user-data meta-data
sudo chown libvirt-qemu:kvm muffins-vm01-seed.iso
sudo chmod 640 muffins-vm01-seed.iso
```

---

## 6. Create the VM with virt-install

```bash
sudo virt-install --name muffins-vm01 --ram 4096 --vcpus 2 --disk path=/home/$USER/muffins/images/muffins-vm01.qcow2,format=qcow2 --disk path=/home/$USER/muffins/cloudinit/muffins-vm01-seed.iso,device=cdrom --os-variant ubuntu24.04 --network bridge=br0,model=virtio --graphics none --import
```

---

## 7. Verify and Access the VM

### Check VM status:

```bash
sudo virsh list --all
sudo virsh domiflist muffins-vm01
```

### Console access:

```bash
sudo virsh console muffins-vm01
```
> Exit: `Ctrl + ]`

### SSH access:

```bash
ssh abraham@<vm-ip> -i ~/.ssh/id_ed25519
```

Find the VM IP:

- From cloud-init output during boot
- In router DHCP leases
- By scanning the network:

```bash
sudo nmap -sn 192.168.2.0/24
```
