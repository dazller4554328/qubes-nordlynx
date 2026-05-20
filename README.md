# qubes-nordlynx

# 🛡️ How to Set Up NordVPN WireGuard on Qubes OS 4.2 / 4.3

This guide walks you through creating a VPN gateway using NordVPN and routing your Qubes AppVMs through it securely. It includes firewall, DNS, MTU, and Qubes 4.3 compatibility fixes.

Tested on **Qubes OS 4.2** and **4.3** using **Fedora 41-43 & Debian 13** for both the VPN gateway and AppVMs.

---

## 🔧 Step 1: Create the VPN Gateway Qube

1. **Create a Standalone AppVM** based on `fedora-41-43 or Debian 13`.  
   Example name: `sys-vpn`.
2. Set its **NetVM** to `sys-firewall`.
3. Open **Qube Settings** for `sys-vpn`:

   * Check ✅ **"Provides network access to other qubes"**
   * Under **Services**, add:

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

## 🌐 Step 3: Set DNS and Apply Fixes

Edit the startup script:

```bash
sudo nano /rw/config/rc.local
```

### For Qubes 4.2, 4.3

Add the following: (Nordvpn DNS servers)

```bash
#!/bin/bash
echo -e "nameserver 103.86.96.100\nnameserver 103.86.99.100" > /etc/resolv.conf
```

### For Qubes 4.3 (includes vif group fix)

Qubes 4.3 switched from iptables to nftables. The NordVPN client overrides the network group on virtual interfaces (`vif*`) from group **2** (required by Qubes) to group **57841** (NordVPN's internal group). This causes Qubes' nftables antispoof and forwarding rules to silently drop all traffic from AppVMs connected to `sys-vpn`.

**Symptoms:**
- AppVMs using `sys-vpn` as their NetVM cannot reach the internet
- `sys-vpn` itself has full internet access
- Switching an AppVM's NetVM to `sys-firewall` and back to `sys-vpn` temporarily fixes it
- The fix breaks again when the AppVM is restarted

**Root cause:**
Qubes 4.3's nftables rules use `iifgroup 2` to identify traffic from downstream qubes. NordVPN sets all network interfaces (including `vif*`) to group 57841, so AppVM traffic bypasses the antispoof `goto` rule and hits a `drop` rule in the prerouting chain.

Add the following to `/rw/config/rc.local`:

```bash
#!/bin/bash

# Set DNS
echo -e "nameserver 103.86.96.100\nnameserver 103.86.99.100" > /etc/resolv.conf

# Fix for NordVPN overriding vif interface groups on Qubes 4.3
# NordVPN sets network interfaces to group 57841, but Qubes nftables
# rules require vif interfaces to be in group 2 for traffic forwarding
# and antispoof rules to work correctly.
(
while true; do
    for iface in /sys/class/net/vif*; do
        [ -e "$iface" ] || continue
        name=$(basename "$iface")
        current_group=$(ip -o link show "$name" 2>/dev/null | grep -o 'group [0-9]*' | awk '{print $2}')
        if [ "$current_group" != "2" ]; then
            ip link set "$name" group 2
        fi
    done
    sleep 2
done
) &
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

## 🔍 Troubleshooting

### AppVMs can't access internet through sys-vpn (Qubes 4.3)

If AppVMs connected to `sys-vpn` have no internet but `sys-vpn` itself works fine, verify the vif interface group in `sys-vpn`:

```bash
ip link show | grep vif
```

If the output shows `group 57841` instead of `group 2`, the NordVPN interface group fix is not applied. Ensure `/rw/config/rc.local` contains the background vif group fix script (see Step 3) and restart `sys-vpn`.

You can manually fix it immediately with:

```bash
sudo ip link set vif<NUMBER>.0 group 2
```

### Diagnosing dropped traffic

To confirm traffic is being dropped by nftables, run in `sys-vpn`:

```bash
sudo nft reset counters
# Then ping from the AppVM, then:
sudo nft list ruleset | grep 'counter packets' | grep -v 'packets 0 '
```

If you see hits on `ip saddr @downstream counter packets ... drop`, the vif group fix is not active.

### MTU issues

If pages load slowly or large downloads fail, ensure MTU is set to 1280 in both `sys-vpn` (via the dispatcher script) and in each AppVM (via `rc.local`).

---

## ✅ Done!

You now have a working NordVPN gateway (`sys-vpn`) on Qubes OS 4.2/4.3 with DNS, MTU, and interface group settings applied, and full traffic routing from selected AppVMs through a secured VPN tunnel.

**Network chain:** `AppVM → sys-vpn → sys-firewall → sys-net → Internet`

