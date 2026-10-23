# Enterprise Campus Network Architecture (Cisco Packet Tracer)

A multi-zone enterprise campus network implemented using Cisco Packet Tracer. The architecture utilizes Layer 3 switches to handle Inter-VLAN routing, centralized DHCP relay configuration, core server farm hosting, and strict network segmentation using Extended Access Control Lists (ACLs).

## 📸 Network Topology Overview

The network is structured into functional departmental zones connected through a central core switch layer:

```
                                  +-------------------+
                                  |   Router 1 / WAN  |
                                  +---------+---------+
                                            |
                                  +---------+---------+
                                  |     Core-SW       | (3560-24PS Layer 3)
                                  |  (IP Routing En)  |
                                  +----+--+---+--+----+
                                       |  |   |  |
      +--------------------------------+  |   |  +--------------------------------+
      |                                   |   |                                   |
+-----+-----+                       +-----+---+---+                         +-----+-----+
| Admin     | (VLAN 10)             | Server Farm | (VLAN 60)             | Guest WiFi| (VLAN 50)
+-----------+                       +-------------+                         +-----------+
      |                                   |                                   |
+-----+-----+                       +-----+-----+                           +-----+-----+
| Faculty   | (VLAN 20)             | core-lab-SW| (L3 Switch)               | Cafeteria | (VLAN 40)
+-----------+                       +-----+-----+                           +-----------+
      |                                   |                                   |
+-----+-----+                 +-----------+-----------+                     +-----------+
| Library   | (VLAN 30)       |           |           |                     |
+-----------+             [VLAN 21]   [VLAN 22]   [VLAN 23/24]              |
                          Lab SW 1    Lab SW 2    Lab SW 3/4                |

```

## 🌐 Network Segmentation & VLAN Addressing Scheme

The IP scheme is based on a `/24` classful subnet structure assigned to dedicated VLAN Virtual Interfaces (SVIs).

| VLAN ID | Subnet / Gateway | Subnet Mask | Department / Functional Zone | Assigned L3 Switch | 
| ----- | ----- | ----- | ----- | ----- | 
| **VLAN 10** | `192.168.10.1` | `255.255.255.0` | Administration | `Core-SW` | 
| **VLAN 20** | `192.168.20.1` | `255.255.255.0` | Faculty Office | `Core-SW` | 
| **VLAN 21** | `192.168.21.1` / `.2` | `255.255.255.0` | Classroom Lab Block 1 | `core-lab-SW` / `Core-SW` | 
| **VLAN 22** | `192.168.22.1` | `255.255.255.0` | Classroom Lab Block 2 | `core-lab-SW` / `Core-SW` | 
| **VLAN 23** | `192.168.23.1` | `255.255.255.0` | Classroom Lab Block 3 | `core-lab-SW` / `Core-SW` | 
| **VLAN 24** | `192.168.24.1` | `255.255.255.0` | Classroom Lab Block 4 | `core-lab-SW` / `Core-SW` | 
| **VLAN 30** | `192.168.30.1` | `255.255.255.0` | Library Area | `Core-SW` | 
| **VLAN 40** | `192.168.40.1` | `255.255.255.0` | Cafeteria Network | `Core-SW` | 
| **VLAN 50** | `192.168.50.1` | `255.255.255.0` | Guest / Wireless Access | `Core-SW` | 
| **VLAN 60** | `192.168.60.1` | `255.255.255.0` | Centralized Server Farm | `Core-SW` / `core-lab-SW` | 

## 🖥️ Server Farm Architecture (VLAN 60)

Centralized services reside within **VLAN 60** (`192.168.60.0/24`). Dynamic IP allocation across all subnets is facilitated by configuring `ip helper-address 192.168.60.5` under each SVI.

| Host IP Address | Server Role | Protocol / Port | Target Accessibility | 
| ----- | ----- | ----- | ----- | 
| **`192.168.60.5`** | Central DHCP Server | UDP 67 (`bootps`) | Accessible by All VLANs | 
| **`192.168.60.8`** | Core DNS Server | UDP 53 (`domain`) | Accessible by All VLANs | 
| **`192.168.60.4`** | FTP File Server | TCP 21 (`ftp`) | Admin & Faculty Office Only | 
| **`192.168.60.7`** | Web (HTTP) Server | TCP 80 (`www`) | Admin Zone Only | 

## 🛡️ Security Policy & Access Control Lists (ACLs)

Inbound Extended Access Control Lists (ACLs) are applied to hardware interfaces on the Layer 3 switches to enforce the principle of least privilege.

```
       [Source Subnet] -------> (Inbound ACL on Interface) -------> [Allowed Services Only]
                                                                        |
                                                                        +--> BOOTP (DHCP)
                                                                        +--> DNS
                                                                        +--> FTP / HTTP (Restricted)

```

### ACL Matrix Rules

```
! Extended ACL: Admin
permit udp 192.168.10.0 0.0.0.255 host 192.168.60.5 eq bootps
permit udp 192.168.10.0 0.0.0.255 host 192.168.60.8 eq domain
permit tcp 192.168.10.0 0.0.0.255 host 192.168.60.4 eq ftp
permit tcp 192.168.10.0 0.0.0.255 host 192.168.60.7 eq www
deny ip 192.168.10.0 0.0.0.255 any

! Extended ACL: FacultyOffice
permit udp 192.168.20.0 0.0.0.255 host 192.168.60.5 eq bootps
permit udp 192.168.20.0 0.0.0.255 host 192.168.60.8 eq domain
permit tcp 192.168.20.0 0.0.0.255 host 192.168.60.4 eq ftp
deny ip 192.168.20.0 0.0.0.255 any

! Extended ACL: Library / GuestWifi / Cafeteria / LabBlock
permit udp <SUBNET> 0.0.0.255 host 192.168.60.5 eq bootps
permit udp <SUBNET> 0.0.0.255 host 192.168.60.8 eq domain
deny ip <SUBNET> 0.0.0.255 any

```

## ⚙️ Key Configuration Highlights

### Core Switch (`Core-SW`)

* **Enable IP Routing:**

  ```
  ip routing
  
  ```

* **DHCP Relay Setup (SVI Example):**

  ```
  interface Vlan10
   ip address 192.168.10.1 255.255.255.0
   ip helper-address 192.168.60.5
  
  ```

* **Binding ACLs to Physical Ports:**

  ```
  interface FastEthernet0/1
   ip access-group FacultyOffice in
  
  ```

### Lab Switch (`core-lab-SW`)

* **Trunk Link Configuration:**

  ```
  interface FastEthernet0/6
   switchport trunk encapsulation dot1q
   switchport mode trunk
  
  ```

##  How to Run in Packet Tracer

1. Clone this repository:

   ```
   git clone https://github.com/your-username/campus-network-l3-switch.git
   
   ```

2. Open **Cisco Packet Tracer** (v8.0 or newer recommended).

3. Open the file `College-Network-Design.pkt`.

4. Run verification commands in the CLI to verify Inter-VLAN connectivity and access control policies:

   ```
   Core-SW# show ip route
   Core-SW# show ip access-lists
   Core-SW# show interface trunk
   
   ```
