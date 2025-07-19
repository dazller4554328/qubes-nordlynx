# qubes-nordlynx
# 🛡️ How to Set Up NordVPN Wireguard on Qubes OS 4.2

This guide walks you through creating a VPN gateway using NordVPN and routing your Qubes AppVMs through it securely. It includes firewall, DNS, and MTU fixes.

Tested on **Qubes OS 4.2** using **Fedora 41** for both the VPN gateway and AppVMs.

---

## 🔧 Step 1: Create the VPN Gateway Qube

1. **Create a Standalone AppVM** based on `fedora-41`.  
   Example name: `sys-vpn`.

2. Set its **NetVM** to `sys-firewall`.

3. Open **Qube Settings** for `sys-vpn`:
   - Check ✅ **"Provides network access to other qubes"**
   - Under **Services**, add:
     ```
     network-manager
     ```

---

## 📥 Step 2: Install NordVPN

In a terminal inside `sys-vpn`, run:

```bash
sh <(wget -qO - https://downloads.nordcdn.com/apps/linux/install.sh) -p nordvpn-gui
```

Once installed, configure NordVPN:
You can also use the NordVPN GUI. You can add it in applications.

```bash
nordvpn login
nordvpn set firewall enabled
nordvpn set killswitch enabled
nordvpn set threatprotectionlite enabled
nordvpn set lan-discovery enabled
```

---

## 🌐 Step 3: Set DNS Permanently

Edit the startup script:

```bash
sudo nano /rw/config/rc.local
```

Add the following:

```bash
#!/bin/bash
echo -e "nameserver 103.86.96.100\nnameserver 103.86.99.100" > /etc/resolv.conf
```

Then make it executable:

```bash
sudo chmod +x /rw/config/rc.local
```

---

## ⚙️ Step 4: Fix MTU for NordLynx

Create a NetworkManager dispatcher script to set the correct MTU:

```bash
sudo nano /etc/NetworkManager/dispatcher.d/99-nordvpn-mtu
```

Add:

```bash
#!/bin/bash
if [ "$1" = "nordlynx" ] && [ "$2" = "up" ]; then
    ip link set dev nordlynx mtu 1280
fi
```

Make it executable:

```bash
sudo chmod +x /etc/NetworkManager/dispatcher.d/99-nordvpn-mtu
```

---

## 🖥️ Step 5: Configure AppVMs to Use `sys-vpn`

For each AppVM that uses `sys-vpn` as its NetVM:

1. Edit the startup script:

```bash
sudo nano /rw/config/rc.local
```

Add:

```bash
#!/bin/bash
echo -e "nameserver 103.86.96.100\nnameserver 103.86.99.100" > /etc/resolv.conf
ip link set dev eth0 mtu 1280
```

Make it executable:

```bash
sudo chmod +x /rw/config/rc.local
```

---

## ✅ Done!

You now have a working NordVPN gateway (`sys-vpn`) on Qubes OS 4.2 with DNS and MTU settings applied, and full traffic routing from selected AppVMs through a secured VPN tunnel.

