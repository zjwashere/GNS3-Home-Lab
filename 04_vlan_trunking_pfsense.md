# 04 - VLAN Trunking on pfSense

## Goal

Before we begin, let's understand what VLAN is. 
Normally, one physical (or virtual) network interface = one broadcast domain — everything plugged into the same switch can see each other's broadcast traffic, and you'd need a separate physical cable/interface per network segment to keep them apart. That doesn't scale — you'd need dozens of NICs on pfSense for dozens of networks.
A VLAN (Virtual LAN) solves this by tagging Ethernet frames with a numeric ID so multiple logically separate networks can share the same physical link. A switch port or interface set to trunk mode carries multiple VLAN-tagged streams simultaneously; each device or sub-interface then only "sees" the traffic tagged for its specific VLAN ID, even though it's all riding the same wire.

Two pieces make this work:
* Trunk link — carries multiple tagged VLANs between pfSense and your switch
* Access port — a switch port that carries only one VLAN, untagged, for whatever end device (a client PC, a server) is plugged into it — the device itself has no idea VLANs exist

This is exactly what lets your one pfSense LAN interface eventually serve three separate networks (Users, Servers, Mgmt) instead of needing three physical NICs.

The goal of Objective #4 is to turn the single LAN interface (from Objective #3) into a trunk carrying three separate isolated networks — Users, Servers, and Management — each with its own subnet. This is the segmentation step that makes the rest of the lab actually resemble a real office network instead of one big flat broadcast domain.

## Objectives

- Assign LAN interface IP. 
- Enable WebGUI/console access. 
- Create trunk parent interface. 
- Create VLAN sub-interfaces (e.g., VLAN10-Users, VLAN20-Servers, VLAN99-Mgmt).

## Steps

## Resources