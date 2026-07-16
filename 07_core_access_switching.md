# 07 - Core & Access Switching

## Goal

The goal for this objective is to create a Layer 2 hierarchy like what real network has: a core switch that only ever talks to other switches and the firewall (nothing plugs directly into it), and one or more access switches where actual end devices (clients, servers, WEB01, DB01 later) physically connect. 
This separation matters for a few concrete reasons:
* Scalability — in a real building, you don't run every device back to one switch; you run access switches per floor/room, uplinked to a core. Your lab should reflect that pattern even at small scale.
* Fault isolation — if an access switch fails or is compromised, only the devices on that switch are affected, not your entire network. A flat single-switch design has no such containment.

Since we are focusing on rebuilding the switch architecture, I will use this opportunity to replace the GNS3 built-in switch with a simulated Cisco switch (IOSvL2) so that we can closely simulate a real network. 

We will also be disabling unused ports and creating a port-to-VLAN documentation. An unused, unconfigured port with no restrictions is a classic way an attacker gets an unintended foothold. Documenting exactly which port carries which VLAN is also just good operational practice — it's the kind of reference sheet any real network has, so troubleshooting doesn't require guessing.

## Objectives

- Replace GNS3 built-in switch with Cisco IOSvL2
- Separate core switch (uplinks to pfSense trunk) and access switch (end devices)
- Configure Cisco switches
- Trunk links carry VLANs 10/20/99; access ports are single-VLAN
- Document a port-to-VLAN map


## Steps

### Get the IOSv2 image from Cisco's CML

1. Go to Cisco Modeling Labs (CML) to download the IOSvL2 reference platform image 
* This requires access to CML or you could go find the file on the internet. The filename to look for is `vios_l2-adventerprisek9-m.ssa.high_iron_20200929.qcow2`.

### Import the GNS3 appliance template

1. In GNS3: File → New template → Install an appliance from the GNS3 server (recommended)
2. Search IOSvL2, select it, Install
![Installing IOSvL2](<docs/screenshots/07_core_access_switching/install appliance.png>)
3. Select Install the appliance on the main server, Next
3. If the status says "Ready to Install" and the .qcow2 file is found locally, click Next
![Ready to install with qcow2 found](<docs/screenshots/07_core_access_switching/found qcow2.png>)
5. Finish the wizard — IOSvL2 now appears in your node list
![IOSvL2 in node list](<docs/screenshots/07_core_access_switching/iosvl2 in node list.png>)

### Redesign the topology

1. Drag two IOSvL2 nodes onto the canvas — rename one Core-SW, one Access-1
2. Link: pfSense (em1) ↔ Core-SW (Gi0/0)
3. Link: Core-SW ↔ Access-1 (Gi0/1 on each)
4. Move your existing test/end-device nodes to connect to Access-1's remaining ports (all port connections in screenshot)

![New topology design](<docs/screenshots/07_core_access_switching/new network.png>)

### Configure Core-SW

Start both switches, console in via Telnet

```sh
enable
configure terminal

hostname Core-SW

# Create the VLANs
vlan 10
 name USERS
vlan 20
 name SERVERS
vlan 99
 name MGMT
exit

# Trunk toward pfSense
interface GigabitEthernet0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 description Uplink-to-pfSense
 no shutdown

# Trunk toward Access-1
interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 description Uplink-to-Access1
 no shutdown

# Shut down every unused port - this is the real version of "disable unused ports"
interface range GigabitEthernet0/2-3
 shutdown
 description UNUSED-DISABLED
interface range GigabitEthernet1/0-3
 shutdown
 description UNUSED-DISABLED
interface range GigabitEthernet2/0-3
 shutdown
 description UNUSED-DISABLED
interface range GigabitEthernet3/0-3
 shutdown
 description UNUSED-DISABLED

# Save
end
copy running-config startup-config
```

### Configure the Access switch's ports

```sh
enable
configure terminal

hostname Access-1

vlan 10
 name USERS
vlan 20
 name SERVERS
vlan 99
 name MGMT
exit

# Trunk toward Core
interface GigabitEthernet0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 description Uplink-to-Core
 no shutdown

# Access port for a USERS test node
interface range GigabitEthernet1/0-1
 switchport mode access
 switchport access vlan 10
 description USERS-nodes
 no shutdown

# Access port for a SERVERS test node
interface GigabitEthernet2/0
 switchport mode access
 switchport access vlan 20
 description SERVERS-nodes
 no shutdown

# Access port for a MGMT test node
interface range GigabitEthernet3/0-1
 switchport mode access
 switchport access vlan 99
 description MGMT-nodes
 no shutdown

# Shut down anything unused
interface range GigabitEthernet0/1-3 #Also do this for gi1/2-3, gi2/1-3, and gi3/2-3 to match the current topology.
 shutdown
 description UNUSED-DISABLED

end
copy running-config startup-config
```

### Verify trunking and VLAN state

To check the configuration of both switches, run these:
```sh
show vlan brief
show interfaces trunk
show interfaces status
```
`show vlan brief` should list your three VLANs with the correct access ports assigned. 
`show interfaces trunk` confirms both trunk links are actually passing 10/20/99. 
`show interfaces status` is a fast way to see which ports are connected vs notconnect vs disabled at a glance — a good screenshot for your docs.

On Core-SW:
![Core switch configurations](<docs/screenshots/07_core_access_switching/core-sw configs.png>)

On Access-1:
![Access-1 switch configurations](<docs/screenshots/07_core_access_switching/access-1 configs.png>)

### Re-run your connectivity matrix

Same tests as before, now through real switch hardware in the path:

```sh
ip dhcp #don't forget to get an IP address 

ping 192.168.10.1     # from USERS node
ping 192.168.20.1     # from SERVERS node
ping 192.168.99.1     # from MGMT node, all should work
```

### Document the port-to-VLAN map

Core switch:
| Port | Peer | Type | VLANs |
| Gi0/0 | pfSense (em1) | Trunk | 10, 20, 99 |
| Gi0/1 | Access-1 (Gi0/0) | Trunk | 10, 20, 99 |

Access-1 switch:
| Port | Peer | Type | VLANs |
| Gi0/0 | Core-SW (Gi0/1) | Trunk | 10, 20, 99 |
| Gi1/0 | USERS node (PC1) | Access | 10 |
| Gi1/1 | USERS node (webterm-1) | Access | 10 |
| Gi2/0 | SERVERS node (PC2) | Access | 20 |
| Gi3/0 | MGMT node (PC3) | Access | 99 |
| Gi3/1 | MGMT node (webterm-2) | Access | 99 |

![Final topology](<docs/screenshots/07_core_access_switching/new topology.png>)

## Resources

Cisco switch: https://www.gns3.com/marketplace/appliances/cisco-iosvl2