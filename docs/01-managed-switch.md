# Phase 2: Managed Switch and VLANs

An 8-port managed switch provides the physical layer for the 802.1Q segmentation defined at the OPNsense firewall. It handles VLAN tag distribution, ingress filtering, and hardware port mirroring for the IDS node.

## 1. Device Hardening and Management Plane Setup

Initial provisioning was done over a direct, isolated connection from the Main PC, with the switch kept off the production network until hardening was complete.

* **Credential Hardening:** The factory default administrator credentials were replaced with a unique, high-entropy passphrase.
* **Management IP Migration:** The switch's vendor-default management address was moved to a static IP in VLAN 10 (`192.168.10.2/24`), with the OPNsense MGMT interface as its default gateway.
* **Access Restriction:** The WebGUI/CLI is reachable only from hosts in the MGMT zone. [Optional: HTTP and Telnet disabled; HTTPS/SSH only.]

## 2. 802.1Q VLAN Provisioning and PVID Mapping

Traffic isolation is enforced at Layer 2 using explicit tagged and untagged membership rules, so each VLAN is its own broadcast domain.

### Port Assignment

```
| Port | Mode     | VLAN(s)         | PVID | Connected Device                        |
|------|----------|-----------------|------|-----------------------------------------|
| 1    | Trunk    | 10, 20, 30, 40  | 1*   | OPNsense (Protectli Vault)              |
| 2    | Access   | 10 (untagged)   | 10   | Main PC                                 |
| 3    | Access   | 20 (untagged)   | 20   | Pi 3B+ (Lab 1)                          |
| 4    | Access   | 40 (untagged)   | 40   | WiFi router (AP-only)                   |
| 5    | Access   | 20 (untagged)   | 20   | Pi 3B+ (Lab 2)                          |
| 6    | Access   | 20 (untagged)   | 20   | Low-resource pen testing laptop         |
| 7    | Disabled | 30 (untagged)   | 30   | Pi 4B, management NIC (alerts/SSH)      |
| 8    | Mirror   | n/a             | -    | Pi 4B, capture NIC (no IP assigned)     |
```

\* Set the trunk's native VLAN to an unused ID (not VLAN 1) if the switch allows it.

### Port Mode Logic

* **Trunk (Port 1):** Carries tagged frames for VLANs 10, 20, 30, and 40 between the switch and OPNsense, which performs all inter-VLAN routing and firewalling.
* **Access (Ports 2-6):** Host-facing ports are untagged members of exactly one VLAN, with the PVID set to match. Untagged frames are tagged with that VLAN ID on ingress and the tag is stripped on egress. Ingress filtering is enabled, so frames carrying a VLAN tag other than the port's own are dropped. This blocks VLAN-hopping attempts such as double tagging.

## 3. Passive Monitoring via SPAN (Port Mirroring)

A hardware port-mirroring session feeds traffic to the Suricata IDS node without putting it in the data path.

* **Source:** Port 1 (OPNsense trunk), both ingress and egress.
* **Destination:** Port 8, connected to the Pi 4B's dedicated capture NIC.
* **Behavior:** The switch copies every frame crossing the trunk, including all inter-VLAN and WAN-bound traffic, out of Port 8. The capture NIC is set to promiscuous mode and has no IP address, so the sensor is invisible on the network. Alerting and management traffic uses the Pi's second NIC on VLAN 30 (Port 5).
* **Coverage Note:** Because the source is the firewall trunk, Suricata sees all routed traffic but not same-VLAN traffic switched locally (e.g., Lab 1 to Lab 2). To capture it, add Ports 3 and 4 as mirror sources.
* **Capacity Note:** A full-duplex mirror of a gigabit port can exceed the destination port's bandwidth under heavy load, causing mirrored frames to be dropped. Production traffic is unaffected.
