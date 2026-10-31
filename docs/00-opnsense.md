# Phase 1: OPNsense on the Protectli Vault

Initial configuration was performed offline, with a monitor and keyboard connected directly to the Protectli Vault, to minimize lockout risk. The core interfaces were built from the OPNsense console menu before the web GUI was used.

## 1. First Boot and Interface Assignment

On first boot, the system loaded the interactive OPNsense console menu.

* **Interface Assignment:** `igc0` was assigned as the WAN port and `igc1` as the internal interface, which later serves as the trunk to the managed switch.
* **Bootstrap Management Address:** `igc1` was given the static address `192.168.10.1/24` to establish the MGMT gateway. The DHCP server was left disabled on this interface so that all administrative endpoints use static addressing.
* **Credential Hardening:** The default `root` password was replaced with a unique, high-entropy passphrase. [Optional: a dedicated admin account was created and direct root login disabled.]
* **Administrative Access:** The Main PC was given a static IP in the MGMT subnet (e.g., `192.168.10.10/24`), providing access to the HTTPS WebGUI at `https://192.168.10.1`.

## 2. VLAN Segmentation and Layer 3 Assignment

With the GUI available, VLANs were created under **Interfaces ▸ Devices ▸ VLAN**, then assigned and enabled under **Interfaces ▸ Assignments**. Each VLAN device uses `igc1` as its parent and has a static IPv4 gateway address. 

```
| VLAN | Name   | Gateway IP     | DHCP                              |
|------|--------|----------------|-----------------------------------|
| 10   | MGMT   | 192.168.10.1   | Disabled (static only)            |
| 20   | Labs   | 192.168.20.1   | Disabled (static only)            |
| 30   | IDS    | 192.168.30.1   | Disabled (static only)            |
| 40   | WiFi   | 192.168.40.1   | Enabled (router is AP-only)       |
```

OPNsense is the sole Layer 3 boundary for all VLANs. Every inter-VLAN flow is routed and filtered by the firewall.

## 3. Firewall Policy and Inter-VLAN Routing

Segmentation follows a default-deny model. Traffic is dropped unless an explicit rule allows it, and a small set of floating rules enforces global protections.

### Floating Rules

```
| # | Action        | Interfaces       | Direction | Source  | Destination | Purpose                                |
|---|---------------|------------------|-----------|---------|-------------|----------------------------------------|
| 1 | Block (quick) | VLAN 20, 30, 40  | In        | Any     | MGMT net    | Prevents lateral movement/MGMT Access  |
```
### Interface Rules

```
| Interface    | Order | Action | Destination                              | Purpose                       |
|--------------|-------|--------|------------------------------------------|-------------------------------|
| VLAN 10 MGMT | 1     | Pass   | Designated hosts   | Administrative SSH to specific endpoints only       |
| VLAN 10 MGMT | 2     | Pass   | Internet           | Updates and package access                          |
| VLAN 40 WiFi | 1     | Pass   | This firewall, DNS | Lets clients resolve names via the gateway          |
| VLAN 40 WiFi | 2     | Block  | RFC 1918 alias     | Isolates wireless clients from all local subnets    |
| VLAN 40 WiFi | 3     | Pass   | Any                | Internet-only access                                |
| VLAN 20 Labs | 4     | Pass   | Internet           | Updates and package access                          |
| VLAN 30 IDS  | 5     | Pass   | Internet           | Updates, package access, Discord webhook alerts     |
```

The DNS pass rule on VLAN 40 must sit above the RFC 1918 block, because the gateway's own address is a private address and would otherwise be blocked too.

## 4. Post-WAN Configuration

Once the WAN link was connected, the following was completed:

* **WAN Addressing:** `igc0` was configured for [DHCP / PPPoE / static] from the ONT.
* **Outbound NAT:** [Automatic] outbound NAT translates all internal VLAN subnets to the WAN address.
* **DNS-over-TLS:** Unbound DNS was configured to forward queries to [provider, e.g., Quad9 / Cloudflare] over TLS (**Services ▸ Unbound DNS ▸ DNS over TLS**), and clients receive their VLAN gateway as their DNS server. [Optional: outbound TCP/UDP 53 from client VLANs is blocked or redirected to prevent DNS bypass.]
