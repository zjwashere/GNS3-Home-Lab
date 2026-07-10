# 01 - Prep the Host (Kali)

## Goal

Before I can use GNS3, I must prepare my Kali host device by downloading all the dependencies and ISOs for VMs in GNS3. 
I'm using Kali Linux in particular because I've already had this OS installed in my device, and I don't plan to install another OS for this project.
Because Kali Linux is Debian-based, the steps to install GNS3 is different if you are running a different OS. 

## Objectives
 - Install GNS3 + local hypervisor (QEMU/VirtualBox). 
 - Download pfSense CE, Windows Server 2022 Eval, Windows 10 Eval ISOs. 
 - Confirm nested networking (loopback/NAT cloud) works.

## Steps

### Install GNS3

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

Insert SS

### Installing VirtualBox

1. Prep
```sh
sudo apt install virtualbox linux-headers-generic
sudo apt install virtualbox-ext-pack
```

2. Verify
```sh
virtualbox
```

Insert SS

### Download ISOs

1. Pfsense CE: https://www.pfsense.org/download/
2. Windows Sever 2022: https://software-download.microsoft.com/download/pr/20348.1.210507-1500.fe_release_SERVER_EVAL_x64FRE_en-us.iso
3. Windows 10 Eval: https://software-static.download.prss.microsoft.com/dbazure/988969d5-f34g-4e03-ac9d-1f9786c66750/19045.2006.220908-0225.22h2_release_svc_refresh_CLIENTENTERPRISEEVAL_OEMRET_x64FRE_en-us.iso

### Testing NAT

1. Drag the NAT from Browse All Devices

Insert SS

Error while creating node from template: NAT interface virbr0 is missing, please install libvirt

Fix: 

```sh
# Install libvirt, standard utilities, and firewalld dependencies
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils firewalld -y

# Start and enable the libvirt service
sudo systemctl enable --now libvirtd
sudo systemctl enable --now firewalld
```

Step 2: Activate the Default NAT Network (virbr0)Even after installing libvirt, the default virtual network might still be inactive. Use these virsh commands to create and permanently activate virbr0:

# Define the default network if it doesn't exist
sudo virsh net-define /etc/libvirt/qemu/networks/default.xml 2>/dev/null

# Set the default network to start automatically on boot
sudo virsh net-autostart default

# Start the default network right now
sudo virsh net-start default

# Restart the daemon to sync changes
sudo systemctl restart libvirtd

Restart System


## Verification

## Issues & Fixes

## Resources
GNS3 Linux install: https://docs.gns3.com/docs/getting-started/installation/linux
VirtualBox for Kali: https://www.kali.org/docs/virtualization/install-virtualbox-host/
