# 10 - DNS Forwarding

## Goal

DC01's DNS server currently only knows about lab.local — it has zero idea how to resolve anything outside your domain (google.com, github.com, Windows Update endpoints, etc.). That's expected: when AD DS installed DNS in Objective #8, it only auto-created the zones for your own domain. Objective #10 closes that gap by configuring forwarders — telling DC01 "for anything you don't know, ask these external resolvers instead."

## Objectives

- Create DNS forwarders to 1.1.1.1/8.8.8.8
- Confirm internal (lab.local) and external resolution via nslookup
- Update DHCP scope option so clients use DC01 as their only DNS server.

## Steps

### Add forwarders on DC01

In the Windows Server:
1. Open DNS Manager (dnsmgmt.msc)
2. Right-click your server (DC01) → Properties
3. Go to the Forwarders tab
4. Click Edit
5. Add: 1.1.1.1 & 8.8.8.8
6. Delete the existing 192.168.20.1 IP address
6. OK → Apply → OK

![Adding forwarders in DC01](<docs/screenshots/10_dns_forwarding/add forwarders.png>)

Or you could run this in PowerShell:
```powershell
Set-DnsServerForwarder -IPAddress 1.1.1.1,8.8.8.8
```

### Confirm pfSense allows DC01 outbound DNS (UDP/TCP 53) to WAN

Since forwarding means DC01 itself now needs to reach the internet on port 53, we will be creating an explicit rule in pfSense to allow outbound DNS to WAN.

### Create an alias for DC01

1. Firewall → Aliases → Add
2. Name: DC01
3. Type: Host(s)
4. Address: 192.168.20.10
5. Save → Apply Changes

![Created DC01 alias](<docs/screenshots/10_dns_forwarding/create dc01 alias.png>)

### Add the rule on the SERVERS interface

In pfSense: 
1. Firewall → Rules → SERVERS tab
2. Add
3. Configure:
* Action: Pass
* Interface: SERVERS
* Address Family: IPv4
* Protocol: TCP/UDP (DNS uses UDP normally, but falls back to TCP for large responses/zone transfers — allow both to be safe)
* Source: Choose `Address or Alias` and type `DC01` in source address (the alias — or type 192.168.20.10 directly if you skipped step 1)
* Destination: Choose "any" since DC01 is forwarding to 1.1.1.1/8.8.8.8 which are out on the internet, not to pfSense's WAN IP itself.
* Destination port range: DNS (53) — pfSense has this as a predefined port alias in the dropdown, select it 
* Description: Allow DC01 outbound DNS to forwarders 
4. Save → Apply Changes

![Created DNS rule in SERVERS](<docs/screenshots/10_dns_forwarding/create dns rule in servers.png>)

### Add the DNS rule on the USERS interface

Then, we must create a rule to allow DNS from USERS VLAN to DC01 

In pfSense: 
1. Firewall → Rules → USERS tab
2. Add
3. Configure:
* Action: Pass
* Interface: USERS
* Address Family: IPv4
* Protocol: TCP/UDP 
* Source: Choose `USERS subnet` 
* Destination: Choose `Address or Alias` and type `DC01` 
* Destination port range: DNS (53)  
* Description: Allow DNS in USERS VLAN 
4. Save → Apply Changes

![Created DNS rule in USERS](<docs/screenshots/10_dns_forwarding/create dns rule in users.png>)

### Verify the rule 

From DC01, in PowerShell, run: `nslookup google.com`

![Running nslookup to check DNS](<docs/screenshots/10_dns_forwarding/dns rule nslookup check.png>)

It should succeed by displaying the IP address for google.com. If you want to be extra rigorous, temporarily disable your existing baseline permissive rule (the Non_Mgmt_VLANs one, or whatever covers SERVERS currently) just to confirm this specific rule

### Test internal resolution

From DC01, run the following:
```powershell
nslookup lab.local
nslookup dc01.lab.local
```
Both should resolve to 192.168.20.10, answered directly by DC01 without needing forwarders at all — these are zones it owns.

![Testing internal resolution](<docs/screenshots/10_dns_forwarding/internal resolution nslookup check.png>)

### Update DHCP scope option 006 (DNS Servers)

You technically already set this in Objective #9, but confirm it explicitly and make sure it's DC01 only — no secondary DNS entry pointing at pfSense or 8.8.8.8 directly, since that would let clients bypass DC01 and break the "one authoritative DNS path" goal:

```powershell
Set-DhcpServerv4OptionValue -ScopeId 192.168.10.0 -DnsServer 192.168.20.10
```

Then, verify:
```powershell
Get-DhcpServerv4OptionValue -ScopeId 192.168.10.0 -OptionId 6
```
Should show only 192.168.20.10 listed.

![Updated DHCP scope option](<docs/screenshots/10_dns_forwarding/update dhcp scope.png>)

### Verify from an actual client

From your USERS VLAN VPCs test node:
* Renew the DHCP lease so it picks up the updated option: `ip dhcp`
* Confirm DNS server assigned is 192.168.20.10 only: `show ip`

Then, to verify working DNS, run: `ping google.com`

![Verified DNS on USERS VLAN](<docs/screenshots/10_dns_forwarding/dns verify.png>)

## Resources
Windows DNS: https://learn.microsoft.com/en-us/windows-server/networking/dns/dns-overview