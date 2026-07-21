# 09 - DHCP Server & Relay

## Goal

Right now, DHCP is coming from pfSense's own built-in DHCP server (something you may have enabled back in Objective #4 just to get a test node an IP). That's fine for early connectivity testing, but it's not how a real domain environment works — in an AD environment, DHCP is almost always run from a domain-integrated server, not the firewall.
* Centralizing DHCP on DC01 means one authoritative source of truth for "who has what IP," which matters for troubleshooting, auditing, and later integrating with DNS (dynamic DNS updates when a DHCP client registers)

This objective is about migrating DHCP responsibility from pfSense to DC01, and because DC01 isn't sitting directly on the USERS VLAN's broadcast domain (it's on SERVERS), you also need DHCP relay — a mechanism that forwards DHCP broadcast requests across VLAN boundaries to a DHCP server that isn't locally attached.

## Objectives

- Disable pfSense's built-in DHCP on USERS first
- Configure DHCP relay/IP helper pointing at DC01
- Verify a test client pulls a lease.

## Steps

### Disable pfSense's built-in DHCP on USERS

1. pfSense GUI → Services → DHCP Server → USERS tab
2. Uncheck Enable DHCP server on USERS interface
3. Save

![Disabled DHCP on USERS interface](<docs/screenshots/09_dhcp_server_relay/disable users dhcp.png>)

### Install the DHCP Server role on DC01

Run in powershell: 
```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
```
![Installed DHCP](<docs/screenshots/09_dhcp_server_relay/install dhcp.png>)

### Authorize DC01 in AD

```powershell
Add-DhcpServerInDC -DnsName "dc01.lab.local" -IPAddress 192.168.20.10
```

Then, verify with the command: `Get-DhcpServerInDC`
This should list DC01.

![DC01 listed as a DHCP server](<docs/screenshots/09_dhcp_server_relay/make dhcpserverindc.png>)

### Create the DHCP scope for USERS

```powershell 
Add-DhcpServerv4Scope `
  -Name "USERS-VLAN10" `
  -StartRange 192.168.10.100 `
  -EndRange 192.168.10.200 `
  -SubnetMask 255.255.255.0 `
  -State Active
```

### Set scope options (gateway, DNS, domain name)

```powershell
Set-DhcpServerv4OptionValue `
  -ScopeId 192.168.10.0 `
  -Router 192.168.10.1 `
  -DnsServer 192.168.20.10 `
  -DnsDomain "lab.local"
```
* Router = pfSense's USERS VLAN gateway
* DnsServer = DC01 itself (this is what makes #10's DNS forwarding actually reach clients)
* DnsDomain = so clients auto-append lab.local for unqualified name lookups

### Configure DHCP relay on pfSense (USERS interface)

Since DC01 lives on SERVERS, not USERS, DHCP broadcasts from USERS clients need to be relayed across the VLAN boundary:

1. pfSense GUI → Services → DHCP Relay
2. Enable DHCP relay on interface: select USERS
* To enable DHCP relay, DHCP must be disabled on SERVERS and MGMT. This can be temporary and static IPs can be made on those nodes. 

3. Upstream server: 192.168.20.10 (DC01)
4. Save → Apply Changes

![Setup DHCP relay](<docs/screenshots/09_dhcp_server_relay/setup dhcp relay.png>)

### Test from a USERS client

From your VPCS test node on USERS, run:
```sh
ip dhcp
```
This should now pull an address in the 192.168.10.100–200 range.

![Users node receives DHCP](<docs/screenshots/09_dhcp_server_relay/users node receive dhcp.png>)

### Verify from the DC01 side too

Run in powershell:
```powershell
Get-DhcpServerv4Lease -ScopeId 192.168.10.0
```

![DHCP lease shown in DC01](<docs/screenshots/09_dhcp_server_relay/dhcp server verify.png>)

This should show the lease your test client just picked up, confirming DC01 is actually the one issuing it.

## Resources

Install and configure DHCP Server on Windows AD: https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/quickstart-install-configure-dhcp-server?tabs=powershell