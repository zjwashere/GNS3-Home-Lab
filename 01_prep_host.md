# 01 - Prep the Host (Kali)

## Goal

Before I can use GNS3, I must prepare my Kali host device by downloading all the dependencies and ISOs for VMs in GNS3. 
I'm using Kali Linux in particular because I've already had this OS installed in my device, and I don't plan to install another OS for this project.
Because Kali Linux is Debian-based, the steps to install GNS3 is different if you are running a different OS. 

## Objectives
 - Enable VT-x/VT-d in BIOS. 
 - Install GNS3 + local hypervisor (QEMU/VirtualBox). 
 - Download pfSense CE, Windows Server 2022 Eval, Windows 10 Eval ISOs. 
 - Confirm nested networking (loopback/NAT cloud) works.

## Steps

1. Install Dependencies
```sh
sudo apt update && sudo apt upgrade -y
sudo apt install python3 python3-pip pipx python3-pyqt6 python3-pyqt6.qtwebsockets \
    python3-pyqt6.qtsvg qemu-kvm qemu-utils libvirt-clients libvirt-daemon-system \
    virtinst ca-certificates curl gnupg2
```

2. Install GNS3 via pipx
```sh
pipx install gns3-server
pipx install gns3-gui
pipx inject gns3-gui gns3-server PyQt6
```
 
3. Dynamips (dropped from apt on Debian 13/Kali — build from source)
```sh
sudo apt install build-essential cmake libelf-dev libpcap0.8-dev
git clone https://github.com/GNS3/dynamips.git
cd dynamips && mkdir build && cd build && cmake .. && sudo make install
```
 
4. Launch (starts GUI + gns3server)
```sh
gns3
```





## Verification

## Issues & Fixes

## Resources
GNS3 Linux install: https://docs.gns3.com/docs/getting-started/installation/linux
