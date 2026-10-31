# Wireless Access Point (VLAN 40)

A consumer wireless router is repurposed as a dedicated access point on VLAN 40. It performs no routing, NAT, or DHCP, so OPNsense remains the only Layer 3 boundary and the only firewall in the path.

* **Operation Mode:** The router's operation mode was changed from Router to **Access Point**. This disables its NAT, firewall, and DHCP server and turns its internet/WAN port into a standard bridged port.
* **Static Addressing:** The AP's management address was set statically within the WiFi subnet.

```
  | Setting         | Value                                       |
  |-----------------|---------------------------------------------|
  | IP Address      | `192.168.40.2`                              |
  | Subnet Mask     | `255.255.255.0`                             |
  | Default Gateway | `192.168.40.1` (OPNsense VLAN 40 interface) |
  | DNS Server      | `192.168.40.1` (OPNsense Unbound)           |
```

* **Physical Connection:** The AP's uplink port connects to a keystone jack leading to Switch Port 6, an untagged access port with PVID 40. The AP is VLAN-unaware, so all wireless clients land in VLAN 40 and receive DHCP leases from OPNsense.
* **DHCP Reservation:** `192.168.40.2` is excluded from the OPNsense DHCP pool (or reserved by MAC) to prevent address conflicts.
* **Hardening:**
  * Default admin credentials replaced with a unique passphrase
  * WPA3 enabled with a strong passphrase
  * WPS, UPnP, and remote management disabled
  * Guest network and any built-in router features disabled, since OPNsense handles isolation
  * Firmware updated before deployment
