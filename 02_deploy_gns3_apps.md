# 02 - Deploy GNS3 Appliances

## Goal

The goal of Objective #2 is to get every "building block" node into GNS3's local library so that from this point forward, building out the lab is just drag-and-drop — no more downloading, importing, or installing base OS images mid-lab.

pfSense will act as the firewall/router for the entire lab and allows us to make VLANs, NAT, firewall rules, IDS/IPS, and VPN.
A WebTerm node lets us get a CLI into the topology without booting a full VM just to run a ping or check a config, saving time and RAM.
Making a Windows Server & Windows 10 templates will be most resource- and time-expensive nodes in the whole lab. Getting them installed, licensed (eval), and stable once now means every later phase (AD DS, DHCP, DNS, file server, GPOs, client join) is just spinning up a clone of a known-good template instead of reinstalling Windows from scratch each time.
VirtIO/Guest tools is installed to speed up disk and network I/O.


## Objectives

- Import pfSense, a switch appliance, and a WebTerm node
- Build Windows Server and Windows 10 QEMU templates. 
- Install VirtIO/Guest tools for performance.

## Steps

## Resources