# 06 - Baseline Firewall Rules

## Goal

This objective is about giving every VLAN a clean, known-good "everything passes" baseline, so that from here on, any connectivity problem you hit while building AD, DHCP, file shares, etc. is caused by that specific feature — not by an accidental firewall gap you forgot existed. It also sets up interface groups, which is a maintenance quality-of-life feature that makes every future rule pass much faster to manage than editing three separate tabs individually.

## Objectives

- Create interface groups 
- Make a rule that will apply to the whole group 

## Steps

### Create the Interface Group

1. pfSense GUI → Interfaces → Assignments → Select Interface Groups tab
2. Click Add
3. Group Name: `Non_Mgmt_VLANs`
4. Members: select USERS & SERVERS (This is done by clicking and then dragging from USERS to SERVERS. This highlights both the USERS and SERVERS)

`Non_Mgmt_VLANs` shows up as its own tab under Firewall → Rules — any rule you add there applies to all 2 member interfaces at once

![Create group assignment](<docs/screenshots/06_baseline_firewall_rules/group assignment.png>)

### Create a group rule

1. pfSense GUI → Firewall → Rules → Select `Non_Mgmt_VLANs` tab
2. Create rule:
* Action: Block
* Protocol: TCP
* Source: Any
* Destination: This Firewall 
* Destination port: 443 and 22
* Description: Block admin access from non-MGMT VLANs
3. Add again (this becomes rule #3, placed below):
* Action: Pass
* Protocol: Any
* Source: Non_Mgmt_VLANs net
* Destination: Any
* Description: TEMP - baseline permissive, remove/replace in #25
4. Save → Apply Changes

![Created group rules](<docs/screenshots/06_baseline_firewall_rules/group rules.png>)

pfSense evaluates group rules first and then, per-interface rules. Just make sure any group rule doesn't sit in a position that undermines restrictions/rules in the interface-specific tabs. Be mindful of pfSense's rule processing order.

I removed the rules from the USERS and SERVERS tab to test the group rule functionality.

### Re-verify MGMT restriction still holds

1. From a USERS-VLAN test node → https://192.168.99.1 (pfSense GUI) → should still fail 
* The Reject rule on USERS should still take priority since it's more specific/higher in that interface's own rule list

![User node doesn't load pfSense](<docs/screenshots/06_baseline_firewall_rules/user pfsense not loading.png>)

2. From a MGMT-VLAN test node → same URL → should still succeed

![Mgmt node loads pfSense](<docs/screenshots/06_baseline_firewall_rules/mgmt pfsense loads.png>)

This shows that group assignments is very powerful at creating firewall rules that apply to multiple VLANs, saving us time and complexity. 

## Resources

pfSense Interface Groups: https://docs.netgate.com/pfsense/en/latest/interfaces/groups.html