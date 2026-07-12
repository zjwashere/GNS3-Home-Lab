# 03 - Install & Configure pfSense

## Goal

With everything installed, we will configure pfSense to create a topology where pfSense will act as the router/firewall. VLANs, firewall rules, DHCP relay, DNS forwarding can't exist until pfSense has a real WAN (for internet-bound traffic) and LAN (for your internal network) with IPs assigned correctly.

## Objectives

- Add pfSense + NAT cloud to canvas. 
- Complete initial setup wizard
- Assign WAN/LAN. 

## Steps

### Build the topology skeleton

1. Drag the pfSense node and NAT node onto the canvas
* The NAT is what gives pfSense's WAN side actual internet access, standing in for "the ISP connection" in a real network

2. Draw a link: NAT node NAT node (nat0) ↔ pfSense's first interface (em0)

SS

3. For the LAN side, Add a switch node (Ethernet0) connected to pfSense's second interface (em1)

SS

4. Start the NAT node and pfSense, console into pfSense 

# Complete initial setup wizard

This should be the same steps from objective #2 when setting up pfSense.

StartupSS

### Configure WAN

From the inital startup, it says that WAN at em0 is already setup. `v4/DHCP4: 192.168.122.244/24`

That is expected because pfSense's WAN interface is set to DHCP, and your NAT cloud runs a DHCP server — so WAN should pull an address automatically.

* However, we can verify this by entering `2` in the console menu for `2) Set interface(s) IP address`
* Then, select interface #1, and for `Configure IPv4 address WAN interface via DHCP?` select `y` 
* This should result in the same IPv4 address that the NAT already gave to us.

configure wanSS

### Configure LAN

LAN needs to be static since this is the network I'll build everything else on top of:

1. From the console menu: `2) Set interface(s) IP address`
2. Select `2` for LAN
3. Select `n` for configuring IPv4 address LAN interface via DHCP
4. Set static IP as `192.168.1.1` and subnet mask as `24`.
SS
5. Press ENTER for none in LAN IPv4 upstream gateway address.
6. Select `n` for configuring IPv6 address LAN interface via DHCP6
7. Press ENTER for none in LAN IPv6 address.
8. Enter `yes` for enable DHCP server on LAN
9. Start range as `192.168.1.100` and end range as `192.168.1.200`.
SS
10. Select `n` for revert to HTTP as webConfigurator protocol. 

SS

### Access the pfSense Web GUI to finish setup

1. Add a WebTerm node connected to the same LAN switch

I ran into this error: `Error while creating node from template: Docker has returned an error: Cannot connect to unix socket /var/run/docker.sock ssl:default [No such file or directory]`

* Fix: WebTerm requires installing docker 
```sh
sudo apt install -y docker.io
sudo systemctl enable docker --now
sudo usermod -aG docker $USER

#Fix permisions
sudo chmod 666 /var/run/docker.sock
```

I ran into this error: `Error while creating node from template: x11 socket file "/tmp/.X11-unix/X100" does not exist`
* Fix: install 
```sh
sudo apt install tigervnc-standalone-server
sudo apt install busybox-static 
```
SS

Now WebTerm finally works, I connected it to the LAN switch

2. Enter https://192.168.1.1 into the address bar

Error: No connection
* Fix: WebTerm containers don't always auto-DHCP on boot, so we have to manually change the /etc/network/interfaces file in WebTerm's terminal 
```sh
#in WebTerm's Terminal 
nano /etc/network/interfaces

# Uncomment the following for DHCP config for eth0 

auto eth0
iface eth0 inet dhcp
```
Then restart the node by turning it off and on.
SS

Now I'm able to access the pfSense's web interface.
SS

## Resources

WebTerm: https://gns3.com/marketplace/appliances/webterm 