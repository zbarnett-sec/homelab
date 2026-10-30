# Home Lab
Documentation, architecture design, and deployment write-ups for my segmented home lab.

## 🌐 Network Architecture Information

The core of the network is powered by OPNsense running on a Protectli Vault, which handles routing, firewall rules, and core network security straight from the ONT. Traffic is distributed via an 8-port managed switch using targeted VLAN segmentation.

```
VLAN ID  |  Name        |  Subnet Scope |  Services
---------------------------------------------------------------------------------------------

VLAN 10  | MGMT         | Main PC       | Secure access to network infrastructure management

VLAN 20  | Labs         | 2x Pi 3B+     | Security testing labs

VLAN 30  | IDS          | Pi 4B         | Suricata, Log2Ram, Cron, Webhook script

VLAN 40  | WiFi         | Router        | Router in AP-only mode
```


### Repository Structure

```
homelab/
├── README.md
├── diagrams/
│   └── topology.md
└── docs/
    ├── 00-opnsense.md
    ├── 01-managed-switch.md
    ├── vlan10-mgmt.md
    ├── vlan20-labs.md
    ├── vlan30-ids.md
    └── vlan40-wifi.md
```
