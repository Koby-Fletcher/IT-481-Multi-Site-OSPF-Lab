# IT-481-Multi-Site-OSPF-Lab
Three-site enterprise network using VLAN segmentation, 802.1Q trunking, router-on-a-stick, and OSPF Area 0 dynamic routing.
## Technologies Used
- Cisco IOS
- VLAN (802.1Q)
- OSPF (Area 0)
- Point-to-Point WAN Links

## Introduction
This lab focuses on designing and configuring a three-site enterprise-style network using VLAN segmentation and OSPF dynamic routing. The goal is to connect HQ-Site, Site-B, and Site-C so that all end devices and loopback interfaces can communicate across WAN links.
This lab reinforces several core networking concepts:
-	VLAN creation and access port configuration
-	Trunking between switches and routers
-	Router-on-a-Stick inter-VLAN routing
-	Point-to-point WAN configuration using /30 networks
-	OSPF single-area (Area 0) routing
-	Troubleshooting Layer 2 vs Layer 3 issues
One of the biggest takeaways for this lab was how tightly Layer 2 and Layer 3 depend on each other. OSPF could be configured correctly, and traffic would not pass if VLANs or trunking were misconfigured. This reinforced the importance of validating switching configurations before assuming that routing is the issue when troubleshooting. For this lab, most of the problems I encountered were related to trunk encapsulation, VLAN forwarding state, and incorrect OSPF process configuration.

## Network Overview
This lab is built around three separate sites — HQ, Site-B, and Site-C — that are connected using point-to-point WAN links in a triangle topology.
### Sites
-	PC1 → SW-HQ → R1
-	PC2 → SW-B → R2
-	PC3 → SW-C → R3
Each site contains:
-	One VLAN
-	One router performing inter-VLAN routing
-	One PC
The routers are connected in a triangle topology using /30 point-to-point WAN links:
-	R1 ↔ R2 → 10.12.1.0/30
-	R1 ↔ R3 → 10.13.1.0/30
-	R2 ↔ R3 → 10.23.1.0/30
Each router also contains a loopback interface used to simulate internal networks and verify OSPF route advertisement. Although this lab does not simulate failover scenarios, the triangular WAN design reflects how real enterprise networks introduce redundancy to avoid single points of failure. 
Network Topology Diagram
 
## IP Addressing
| Device | IP Address       | Subnet Mask | Default Gateway |
|--------|------------------|------------|-----------------|
| PC1    | 142.100.64.11    | /24        | 142.100.64.1    |
| PC2    | 142.100.65.21    | /24        | 142.100.65.1    |
| PC3    | 142.100.66.31    | /24        | 142.100.66.1    |
## VLAN Assignment
| Site | VLAN ID | VLAN Name |
|--------|--------|----------------|
| HQ | 100 | HQ-VLAN |
| Site-B | 200 | Site-B-VLAN |
| Site-C | 300 | Site-C-VLAN |
## WAN Links

| Link | Network |
|--------------|----------------|
| R1 ↔ R2 | 10.12.1.0/30 |
| R1 ↔ R3 | 10.13.1.0/30 |
| R2 ↔ R3 | 10.23.1.0/30 |
## Loopback Interfaces

| Router | Loopback IP |
|--------|------------------|
| R1 | 142.1.64.254 |
| R2 | 142.1.65.254 |
| R3 | 142.1.66.254 |
## Step 1 – Switch configuration
### SW-HQ
Create VLAN 100
```cisco
conf t
vlan 100
name HQ-VLAN
end
wr
```

PC1 Access Port(e0/0)
```cisco
conf t
interface e0/1
switchport mode access
switchport access vlan 100
spanning-tree portfast
no shutdown
end
wr
```

Trunk to R1 (e0/1)
```cisco
conf t
interface e0/0
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk allowed vlan 100
switchport trunk native vlan 100
spanning-tree portfast trunk
no shutdown
end
wr
```

Verify
```cisco
show vlan brief
show interfaces trunk
```

### SW-B
Create VLAN 200
```cisco
conf t
vlan 200
name SITE-B-VLAN
end
wr
```

PC2 Access Port (e0/0)
```cisco
conf t
interface e0/0
switchport mode access
switchport access vlan 200
spanning-tree portfast
no shutdown
end
wr
```

Trunk to R2 (e0/1)
```cisco
conf t
interface e0/1
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk allowed vlan 200
switchport trunk native vlan 200
spanning-tree portfast trunk
no shutdown
end
wr
```

Verify
```cisco
show vlan brief
show interfaces trunk
```

### SW-C
Create VLAN 300
```cisco
conf t
vlan 300
name SITE-C-VLAN
end
wr
```

PC3 Access Port (e0/1)
```cisco
conf t
interface e0/1
switchport mode access
switchport access vlan 300
spanning-tree portfast
no shutdown
end
wr
```

Trunk to R3 (e0/1)
```cisco
conf t
interface e0/0
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk allowed vlan 300
spanning-tree portfast trunk
no shutdown
end
wr
```

Verify
```cisco
show vlan brief
show interfaces trunk
```

## Step 2 – Router Configuration
Each of the routers performs two major roles in this lab. First, each router connects to the other routers using /30 point-to-point WAN links. Second, it performs inter-VLAN routing using sub-interfaces configured with 802.1Q encapsulation. Loopback interfaces are added to simulate internal networks and to demonstrate OSPF route advertisement.
### R1 (HQ)
#### WAN Links
```cisco
interface e0/2
ip address 10.12.1.1 255.255.255.252
no shutdown

interface e0/3
ip address 10.13.1.1 255.255.255.252
no shutdown
```

#### VLAN 100 Sub-interface
```cisco
interface e0/0
no shutdown

interface e0/0.100
encapsulation dot1Q 100 native
ip address 142.100.64.1 255.255.255.0
no shutdown
```
#### Loopback
```cisco
interface loopback0
ip address 142.1.64.254 255.255.255.0
end
wr
```

### R2 (Site-B)
#### WAN Links
```cisco
conf t
interface e0/2
ip address 10.12.1.2 255.255.255.252
no shutdown

interface e0/1
ip address 10.23.1.1 255.255.255.252
no shutdown
```

#### VLAN 200 Sub-interface
```cisco
interface e0/0
no shutdown

interface e0/0.200
encapsulation dot1Q 200 native
ip address 142.100.65.1 255.255.255.0
no shutdown
```
#### Loopback
```cisco
interface loopback0
ip address 142.1.65.254 255.255.255.0
end
wr
```

### R3 (Site-C)
#### WAN Links
```cisco
conf t
interface e0/3
ip address 10.13.1.2 255.255.255.252
no shutdown

interface e0/1
ip address 10.23.1.2 255.255.255.252
no shutdown
```

#### VLAN 300 Sub-interface
```cisco
interface e0/0
no shutdown

interface e0/0.300
encapsulation dot1Q 300 native
ip address 142.100.66.1 255.255.255.0
no shutdown
```
#### Loopback
```cisco
interface loopback0
ip address 142.1.66.254 255.255.255.0
end
wr
```

## Step 3 – OSPF Configuration (Area 0)
### R1
```cisco
conf t
router ospf 100
router-id 1.1.1.1
network 10.12.1.0 0.0.0.3 area 0
network 10.13.1.0 0.0.0.3 area 0
network 142.100.64.0 0.0.0.255 area 0
network 142.1.64.254 0.0.0.0 area 0
end
wr
```

### R2
```cisco
conf t
router ospf 200
router-id 2.2.2.2
network 10.12.1.0 0.0.0.3 area 0
network 10.23.1.0 0.0.0.3 area 0
network 142.100.65.0 0.0.0.255 area 0
network 142.1.65.254 0.0.0.0 area 0
end
wr
```

### R3
```cisco
conf t
router ospf 300
router-id 3.3.3.3
network 10.13.1.0 0.0.0.3 area 0
network 10.23.1.0 0.0.0.3 area 0
network 142.100.66.0 0.0.0.255 area 0
network 142.1.66.254 0.0.0.0 area 0
end
wr
```

## Step 4 – PC configuration
### PC1
```cisco
conf t
interface e0/0
ip address 142.100.64.11 255.255.255.0
no shutdown
exit
no ip routing
ip default-gateway 142.100.64.1
end
wr
```

### PC2
```cisco
conf t
interface e0/0
ip address 142.100.65.21 255.255.255.0
no shutdown
exit
no ip routing
ip default-gateway 142.100.65.1
end
wr
```

### PC3
```cisco
conf t
interface e0/0
ip address 142.100.66.31 255.255.255.0
no shutdown
exit
no ip routing
ip default-gateway 142.100.66.1
end
wr
```

## Verification
### Router Verification
```cisco
show ip ospf neighbor
show ip route ospf
```

#### Switch Verification
```cisco
show vlan brief
show interfaces trunk
```

#### Ping Tests
From each PC:
-	Ping both other PCs
-	Ping all the loopback interfaces
Screenshots of all verification commands are included below to demonstrate successful adjacency, route learning, and end-to-end connectivity.
## Verification Screenshots
### R1 OSPF Neighbors
 
### R2 OSPF Neighbors
 
### R3 OSPF Neighbors
 
### R1 OSPF Routes
 
### Switch VLAN & Trunk Verification
### SW-HQ
 
### SW-B
 
### SW-C
 
### PC1 Ping Results
 
### PC2 Ping Results
 
### PC3 Ping Results
 

## Troubleshooting 
Issue 1 – Trunk Encapsulation Error
When configuring trunk ports, the command switchport mode trunk failed due to encapsulation being set to "auto." The interface was set to automatic encapsulation, which prevented trunk mode from being enabled. After explicitly configuring switchport trunk encapsulation dot1q, the trunk successfully formed and VLAN traffic began forwarding correctly.

Issue 2 – VLAN Not Forwarding Traffic
Even though VLANs were created, traffic was not passing to the router. The trunk was not properly forwarding the VLAN. Verifying with show interfaces trunk helped identify the issue, and explicitly allowing the VLAN fixed the problem.

Issue 3 – Default Gateway Misconfiguration
One PC could not reach remote networks because the default gateway was not set properly. Setting ip default-gateway corrected the problem.

Issue 4 – OSPF Process Mismatch
At one point, OSPF neighbors were not forming because different OSPF process IDs were configured incorrectly. Although process IDs do not need to match across routers, missing network statements prevented adjacency from forming. After correcting the OSPF network statements and verifying with show ip ospf neighbor, adjacency reached FULL state.

## Conclusion
This lab reinforced the relationship between Layer 2 switching and Layer 3 routing. Although OSPF configuration was important, most connectivity failures were caused by VLAN and trunk misconfiguration. The biggest lesson learned is to always verify Layer 2 configuration before troubleshooting routing protocols.
The most important lesson from this lab is to troubleshoot methodically. When pings fail, start at Layer 2 and work upward. Verifying trunk status, VLAN assignment, and interface states before analyzing routing protocols prevents unnecessary reconfiguration and saves significant time.
