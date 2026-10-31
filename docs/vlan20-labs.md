# Phase 4: Compute Node Provisioning and Vulnerable Workload Staging

> **Status: VLAN 20 (Labs) is being phased out.** The Juice Shop lab nodes on VLAN 20 are being retired. Juice Shop was initially moved to a laptop on [new lab VLAN / network] and is also in the process of being phased out. The Juice Shop sections below are kept as a reference for anyone reproducing the setup on low-resource hardware.

**Why the move:** Juice Shop v20+ requires an open-weight LLM, which the Pi 3B+ cannot run. A laptop with more RAM and CPU hosts it instead. It still runs bare metal, not in Docker or a VM, because the hardware remains constrained. Bare-metal Juice Shop on ARM is still possible on better-equipped boards such as the Raspberry Pi 5, so the ARM procedure is preserved in Section 3.

**⚠️ CRITICAL SECURITY WARNING**
OWASP Juice Shop is an intentionally vulnerable application containing severe flaws, including remote code execution, SQL injection, and cross-site scripting. Running it bare metal exposes the host directly. Deploy it only on an isolated lab network or dedicated testing VLAN, and never on a production network or the main VLAN. The author assumes no liability for compromised hosts, data loss, network pivoting, or other damage resulting from its use.

---

## 1. Current Deployment: Juice Shop on Linux x64 (Laptop)

### Requirements

* `nodejs`
* `npm`
* `ollama`

### Setup

* **Install Node.js and npm:**

  ```
  sudo apt update && sudo apt install nodejs npm -y
  ```

* **Install Ollama:**

  ```
  curl -fsSL https://ollama.com/install.sh | sh
  ```

* **Pull the Gemma 4 (e4b) model:**

  ```
  ollama pull gemma4:e4b
  ```

* **Clone Juice Shop:**

  ```
  git clone https://github.com/juice-shop/juice-shop.git --depth 1
  ```

* **Install dependencies:**

  ```
  cd juice-shop
  npm install
  ```

* **Start Juice Shop:**

  ```
  npm start
  ```

Once running, the application is available at `http://localhost:3000`. [Other lab hosts can reach it at `http://<laptop-ip>:3000`, subject to firewall rules.]

### Recommended Testing Tools

* **Intercepting proxy:** Burp Suite (alternative: OWASP ZAP)
* **Content discovery:** `ffuf` (alternative: `gobuster`) with `wordlists` / `seclists`
* **Hash cracking:** `hashcat` or John the Ripper (alternative: CyberChef, with no install required)

---

## 2. Legacy Baseline: Raspberry Pi OS Lite on VLAN 20

A baseline 64-bit Raspberry Pi OS Lite installation was deployed locally on each Pi 3B+. Because the hardware is resource-constrained, the configuration keeps overhead minimal and applies explicit security controls.

### A. Keyboard Layout and Password Rotation

Typing complex passphrases under the wrong keyboard mapping can lock you out, so the password was rotated in stages:

* **Initial password:** A temporary passphrase with no special characters was set, so it could be entered reliably before the layout was corrected.

* **Keyboard layout:** The default UK mapping was changed to US ANSI:

  ```
  sudo nano /etc/default/keyboard
  ```

  Change `XKBLAYOUT="gb"` to `XKBLAYOUT="us"`, then save (`CTRL + O`, `Enter`) and exit (`CTRL + X`).

  Apply the layout:

  ```
  sudo setupcon
  ```

* **Final password:** With the correct layout active, the password was rotated to a strong passphrase containing alphanumeric and special characters:

  ```
  passwd
  ```

### B. SSH Public-Key Authentication

SSH was set up to use public-key authentication. The public key was transferred from a USB drive. [Optional: the key is generated on and protected by a hardware security key with a passphrase.]

* **Create the SSH directory with restrictive permissions:**

  ```
  mkdir -m 700 ~/.ssh
  ```

* **Mount the USB drive holding the public key:**

  ```
  lsblk
  ```

  ```
  sudo mkdir /mnt/usb
  ```

  ```
  sudo mount /dev/sda1 /mnt/usb
  ```

  ```
  ls -l /mnt/usb
  ```

  The last command lists the files on the drive.

* **Install the public key and restrict its permissions:**

  ```
  cp /mnt/usb/id_rsa.pub ~/.ssh/authorized_keys
  ```

  ```
  chmod 600 ~/.ssh/authorized_keys
  ```

  ```
  cd
  ```

* **Unmount the drive:**

  ```
  sudo umount /mnt/usb
  ```

* **Enable and start the SSH daemon:**

  ```
  sudo systemctl enable ssh --now
  ```

### C. USB Boot Timeout

The Pis boot from a microSD card in a USB adapter, so the firmware's USB wait timeout was adjusted to give the adapter time to initialize at boot:

```
sudo nano /boot/firmware/config.txt
```

Add this line at the bottom of the file, then save and exit (`CTRL + O`, `Enter`, `CTRL + X`):

`program_usb_timeout=1`

### D. Static IP Configuration (VLAN 20)

Each Pi uses a static address instead of DHCP. The Pi is not VLAN-aware: VLAN membership comes from the switch, where its port is an untagged access port with PVID 20. The address is configured with NetworkManager's text interface (`nmtui`) and `nmcli`.

```
sudo nmtui
```

1. Select **Edit a connection** and press `Enter`.
2. Select **Wired connection 1** and press `Enter`.
3. Next to **IPv4 CONFIGURATION**, change `<Automatic>` to `<Manual>`.
4. Select `<Show>` next to **IPv4 CONFIGURATION** and press `Enter`.
5. Select `<Add>` beside **Addresses** and enter the static IP in CIDR form (e.g., `192.168.20.x/24`).
6. Enter the gateway and add a DNS server.
7. Select `<OK>`, press `Enter`, then press `ESC` and select `<Back>` / `<Quit>` to exit.

**Gateway:** It must be in the same subnet as the host's IP. On VLAN 20 this is the OPNsense interface (`192.168.20.1`). Using `.1` for the gateway is a common convention, not a requirement.

**DNS:** Public resolvers such as `1.1.1.1` or `8.8.8.8` work, but if OPNsense provides DNS-over-TLS through Unbound, use the gateway address as the DNS server.

Bind the connection to the interface and disable IPv6:

```
sudo nmcli connection up "Wired connection 1"
```

```
sudo nmcli connection modify "Wired connection 1" connection.interface-name eth0
```

```
sudo nmcli connection modify "Wired connection 1" ipv6.method disabled
```

Verify the address (it should show the static IP):

```
ip a show eth0
```

---

## 3. Legacy: Juice Shop Bare Metal on ARM (Raspberry Pi)

> **Untested:** This procedure could not be fully validated on the Raspberry Pi 3B+ because of resource limits, which is why the project moved to a laptop. Older Juice Shop versions can run on a Pi 3B+, but expect to troubleshoot. This section is better suited to a Raspberry Pi 5 or similar.

Instead of containers, Juice Shop was built and run as a native Node.js application, managed by systemd.

**Install dependencies and clone the project:**

```
sudo apt update && sudo apt install -y nodejs build-essential git cmake automake libtool pkg-config
```

```
sudo apt install npm
```

```
cd /opt
```

```
sudo git clone https://github.com/juice-shop/juice-shop.git --depth 1
```

```
cd juice-shop
```

**Compile TypeScript:**

```
sudo apt install typescript
```

```
sudo npx tsc
```

**Create the systemd service:**

```
cat << 'EOF' | sudo tee /etc/systemd/system/juiceshop.service > /dev/null
[Unit]
Description=OWASP Juice Shop Bare-Metal Service
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/juice-shop
ExecStart=/usr/bin/node /opt/juice-shop/build/app.js
Restart=always
Environment=NODE_ENV=production
Environment=NODE_OPTIONS="--max-old-space-size=512"

[Install]
WantedBy=multi-user.target
EOF
```

**Start the service:**

```
sudo systemctl start juiceshop.service
```

**Verify status:**

```
sudo systemctl status juiceshop.service
```

**Troubleshooting:** If the service fails to start, reload the systemd configuration and clear the failed state:

```
sudo systemctl daemon-reload
```

```
sudo systemctl reset-failed juiceshop.service
```

```
sudo systemctl start
