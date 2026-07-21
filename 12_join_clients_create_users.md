# 12 - Join Clients & Create Users

## Goal

This objective is where all the infrastructure work (VLANs, DHCP relay, DNS, OUs) gets proven out end-to-end from an actual end user's perspective. Up to now you've validated everything with VPCS nodes — lightweight, but they can't actually join a domain or log in as a real user. A Windows 10 client joining the domain is the first real test of whether DC01 is fully functional: DHCP relay has to hand it an IP, DNS has to resolve lab.local's SRV records so it can even find a DC, and once joined, Kerberos authentication has to work for a domain user to log in at all.

"Force password change at next logon" matters for a specific, real reason: whoever creates an account (you, the admin) shouldn't be the only one who ever knows that account's password. Forcing a change at first login means the account's actual owner sets a password only they know — standard practice for any account provisioning workflow

## Objectives

- Deploy Windows 10 client(s), confirm DHCP relay works. 
- Create users placed directly into correct OUs, force password change at next logon. 
- Join to domain, confirm computer object landed in expected OU.

## Steps

### Deploy the Windows 10 client node

1. Drag your Windows 10 template onto the canvas, name it something like CLIENT01
2. Connect it to Access-1 (Gi1/2), on the port assigned to VLAN10 (USERS) (Refer back to Objective #7 to configure the switch interface)
3. Start it, console in (VNC)
4. If this is first boot, complete the setup

![Add Windows 10 to network](<docs/screenshots/12_join_clients_create_users.md/new network design.png>)

### Confirm DHCP relay works for a real client

Once in the desktop,
```cmd
ipconfig /all
```
Confirm:
* IP in range 192.168.10.100–200
* Gateway 192.168.10.1
* DNS server 192.168.20.10 (DC01)

![Confirming DHCP on the device](<docs/screenshots/12_join_clients_create_users.md/confirm dhcp relay in cmd.png>)

If it didn't pull a lease automatically, force it:
```cmd
ipconfig /release
ipconfig /renew
```

This is a genuinely more meaningful test than the VPCS one from earlier — a real Windows DHCP client behaves closer to how your eventual real clients will, and confirms the relay path (USERS → pfSense → SERVERS → DC01) works for actual Windows networking stack behavior, not just VPCS's simplified one.

### Confirm name resolution works from the client

```cmd
nslookup lab.local
nslookup dc01.lab.local
ping dc01.lab.local
```

However, running these commands intermittently fails and ping requests timeouts.
![Nslookup fails and sometimes doesn't work](<docs/screenshots/12_join_clients_create_users.md/nslookup and ping fails.png>)

The reason for this is because Core-SW and Access-1 is probably running on full load. This is evident by some pings succeed with normal latency (4-5ms) while others take 20-95ms or time out entirely. That means the 1 vCPU to run the switches are probably overloaded. 
I tried fixing this by increasing the vCPU count in GNS3 for Core-SW and Access-1 from 1 to 2. I also increased the vCPU for pfSense node and the Windows Server DC01 just in case.
![Changing the vCPU on nodes](<docs/screenshots/12_join_clients_create_users.md/changing vcpu in node.png>)

Now, this actually didn't fix the issue, which confused me. Then, I tried taking a packet capture between Core-SW and Access-1. This is what I received:
![The packet capture between core and access switch](<docs/screenshots/12_join_clients_create_users.md/packet capture.png>)

The capture showed there were many packets going to the IP address: 172.184.91.8, which is from Microsoft. This means that my Windows nodes must be flooding my switches with random Microsoft Windows traffic, leading to slow switches. 

Then, when I disconnected from WiFi/Internet connection, I tried running the name resolution test again:
![Nslookup and ping works after removing internet](<docs/screenshots/12_join_clients_create_users.md/nslookup and ping success.png>)

Notice the ping latency is now 5-21 ms. These all should succeed and it seems like the root cause was the internet connection to Microsoft's servers. Thus, to prevent this from happening again, I deleted the link between the NAT & pfSense to temporarily remove internet. 

### Create the AD user(s) — placed directly in the correct OU

On DC01, PowerShell:
```powershell
New-ADUser -Name "Jane Doe" `
  -GivenName "Jane" -Surname "Doe" `
  -SamAccountName "jdoe" `
  -UserPrincipalName "jdoe@lab.local" `
  -Path "OU=Users,OU=Corp,DC=lab,DC=local" `
  -AccountPassword (ConvertTo-SecureString "TempPassword!" -AsPlainText -Force) `
  -ChangePasswordAtLogon $true `
  -Enabled $true
```

Key details:
* -Path places the user directly into your Corp\Users OU from #11 — not the default Users container. This is the part that's easy to fumble in ADUC if you're not paying attention to which container you right-click under
* -ChangePasswordAtLogon $true is exactly the "force change" requirement — the temp password you set only works long enough to log in once and be replaced

Repeat for however many test users you want (e.g., one more to test cross-user file permission behavior later in #16).

![Create AD user in powershell](<docs/screenshots/12_join_clients_create_users.md/create ad user in dc01.png>)

### Join CLIENT01 to the domain

On the client, either via GUI or PowerShell:

GUI: Settings → System → About → "Rename this PC (advanced)" → Change → Domain: lab.local → Change computer name to "CLIENT01" → provide domain admin credentials when prompted → reboot

PowerShell:
```powershell
Add-Computer -DomainName "lab.local" -Credential (Get-Credential) -Restart
```

![Joining CLIENT01 to the domain](<docs/screenshots/12_join_clients_create_users.md/joining client01 to domain via gui.png>)

### Log in as the domain user

After reboot, at the login screen → Click on Other Users.
Type the username of the jdoe user & the temporary password.

![Choosing other users to log into jdoe](<docs/screenshots/12_join_clients_create_users.md/other users to log into jdoe.png>)

Then, change the password (it must match the domain's password policy)

![Changing the temp password for the account](<docs/screenshots/12_join_clients_create_users.md/change the temp password.png>)

Now, a domain profile is created (C:\Users\jdoe)

### Confirm the computer object landed in the expected OU

By default, Add-Computer drops new computer objects into the default Computers container — not wherever you might expect from your OU design. This is a very common surprise, worth checking explicitly:

```powershell
Get-ADComputer -Identity "CLIENT01" -Properties DistinguishedName | Select-Object DistinguishedName
```

If it shows `CN=CLIENT01,CN=Computers,DC=lab,DC=local` (the default container, note CN= not OU=), it did not land where your #11 design intended. Move it:
In the GUI → Server Manager → Tools → Active Directory Users and Computers → Go to /lab.local/Computers (the default container) → Right click CLIENT01 → Choose `Move` → Choose /lab.local/Comp/Computers 

![Moving CLIENT01 via the GUI](<docs/screenshots/12_join_clients_create_users.md/moved client01 via gui.png>)

Or you can do this in Powershell:
```powershell
Move-ADObject -Identity "CN=CLIENT01,CN=Computers,DC=lab,DC=local" -TargetPath "OU=Computers,OU=Corp,DC=lab,DC=local"
```

Then, re-run the Get-ADComputer check to confirm the move worked.

![Verifying the moving of CLIENT01](<docs/screenshots/12_join_clients_create_users.md/verify the move.png>)


### Verify the user landed correctly too

```powershell
Get-ADUser -Identity "jdoe" -Properties DistinguishedName | Select-Object DistinguishedName
```
Should show OU=Users,OU=Corp,DC=lab,DC=local — confirming step 4's -Path worked as expected (users created via New-ADUser -Path don't have the default-container problem the way computer-join does, but worth verifying anyway)

![Verifying the placement of the user](<docs/screenshots/12_join_clients_create_users.md/verify user placement.png>)

## Resources
Windows comptuer join domain: https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/join-computer-to-domain?tabs=cmd&pivots=windows-server-2025
Managing AD Users: https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage-user-accounts-in-windows-server