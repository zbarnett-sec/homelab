# Home Lab
Documentation, architecture design, and deployment write-ups for my segmented home lab.

## 🌐 Network Architecture Information

The core of the network is powered by OPNsense running on a Protectli Vault, which handles routing, firewall rules, and core network security straight from the ONT. Traffic is distributed via an 8-port managed switch using targeted VLAN segmentation.


### Repository Structure

```
├── README.md
├── diagrams/
│   └── topology.md
└── docs/
    ├── 00-opnsense.md
    ├── 01-managed-switch.md
    ├── vlan10-mgmt.md
    ├── vlan20-labs.md
    ├── vlan30-ids.md
    └──  vlan40-wifi.md
```
