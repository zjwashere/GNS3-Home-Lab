# 05 - Management VLAN

## Goal

We already created MGMT VLAN (VLAN 99) in Objective #4. Now this objective is about actually using it for its intended purpose: making it the only place admin access (pfSense GUI, switch management, later RADIUS/NPS) is reachable from, instead of just being "one more VLAN that happens to exist."
This matters because of security. In a real network, if someone compromises a regular user's PC on the Users VLAN, you don't want them one hop away from your firewall's admin panel. Segmenting management traffic onto its own VLAN — and then restricting GUI/SSH/console access to only originate from that VLAN — is a foundational "defense in depth" practice.

## Objectives

*  Stand up a dedicated MGMT VLAN for switch/firewall admin access, separate from user traffic.

## Steps

### Add firewall rules: only allow GUI/SSH access from MGMT source

Before doing this step, I enabled DHCP on the MGMT interface to make it easy to work with. Steps are in Objective #4. 

1. Go to Firewall → Rules → USERS tab (and repeat for SERVERS tab)
2. Add a rule:
* Action: Block
* Source: any
* Destination: This firewall (or the specific pfSense IP)
* Destination port: 443 (HTTPS/GUI), 22 (SSH) if enabled
* Place this rule above your existing permissive "allow all" test rule from Objective #4, since pfSense evaluates rules top-down, first match wins

![Setting firewall rules in USERS](<docs/screenshots/05_management_vlan/set firewall rule.png>)

3. Firewall → Rules → MGMT tab
* Add a rule: Action: Pass, Source: MGMT net, Destination: This firewall, any port — ensuring admin access from MGMT still works

![Setting firewall rules in MGMT](<docs/screenshots/05_management_vlan/set firewall rule mgmt.png>)

### Verify segmentation works

1. Set up two test nodes:
* VPCS (Port 1) & WebTerm (Port 3) on MGMT (VLAN99, access port) 
* VPCS (Port 6) & WebTerm (Port 7) on USERS (VLAN10, access port) 

![New network design](<docs/screenshots/05_management_vlan/new network.png>)

![Configuring VLAN on switch](<docs/screenshots/05_management_vlan/configure vlan switch.png>)

2. From the MGMT VPCS node: 
```sh
ping 192.168.99.1        # Make sure you have the temporary permissive rule for MGMT interface. It should succeed
```

![Ping from MGMT node](<docs/screenshots/05_management_vlan/mgmt ping.png>)

Then try reaching the GUI: open a browser from a WebTerm on MGMT → https://192.168.99.1 → should load pfSense's login page

![pfSense accessible through MGMT WebTerm](<docs/screenshots/05_management_vlan/mgmt pfsense gui.png>)

3. From the USERS node:
```sh
ping 192.168.10.1        # should still succeed - general routing intact
```

![Ping from USER node](<docs/screenshots/05_management_vlan/users ping.png>)

Try reaching the GUI: https://192.168.99.1 or https://192.168.10.1:443 from this node → should now fail/timeout, confirming your block rule worked

![pfSense timeouts from USER VLAN](<docs/screenshots/05_management_vlan/user pfsense timeout.png>)


This successfully shows working segmentation between VLANs and improves the security of the network.

## Resources