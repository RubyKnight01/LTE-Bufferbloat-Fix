# Configuration Scripts

Complete scripts and commands to fix bufferbloat on OpenWrt LTE connections by bypassing hardware offloading.

---

## Overview

These scripts accomplish three things:

1. **Create IFB bridge** - Virtual interface for ingress traffic shaping
2. **Persist across reboots** - Automatically reapply settings when LTE reconnects
3. **Disable hardware offloading** - Force traffic through CPU for SQM processing

---

## Script 1: SQM Initialization

Creates the IFB virtual bridge and redirects WAN traffic for CPU-based shaping.

### Installation

```bash
vi /etc/init.d/sqm-fix
```

Press `i` to enter insert mode, paste the script below, then save with `ESC` → `:wq` → `ENTER`

### Script Contents

```bash
#!/bin/sh /etc/rc.common
# Creates virtual IFB device for ingress (download) shaping
# Redirects eth0 traffic to CPU so SQM/CAKE can process it

START=99

start() {
    echo "Initializing SQM Bridge for Qualcomm NSS bypass..."
    
    # Create virtual device using SQM naming convention
    ip link add name ifb4eth0 type ifb 2>/dev/null
    ip link set dev ifb4eth0 up
    
    # Attach ingress qdisc hook to physical WAN port (eth0)
    tc qdisc add dev eth0 handle ffff: ingress 2>/dev/null
    
    # Redirect all traffic from eth0 to virtual device ifb4eth0
    tc filter add dev eth0 parent ffff: protocol all u32 match u32 0 0 \
        action mirred egress redirect dev ifb4eth0 2>/dev/null
    
    # Restart SQM service to detect new interface
    /etc/init.d/sqm restart
}
```

### Activation

```bash
chmod +x /etc/init.d/sqm-fix
/etc/init.d/sqm-fix enable
```

**What this does:**
- Makes the script executable
- Enables automatic startup on boot

---

## Script 2: Hotplug Persistence

Automatically reapplies the IFB bridge when LTE reconnects or interface resets.

### Installation

```bash
vi /etc/hotplug.d/iface/99-sqm-fix
```

Press `i` to enter insert mode, paste the script below, then save with `ESC` → `:wq` → `ENTER`

### Script Contents

```bash
#!/bin/sh
# Hotplug script to ensure SQM persists after interface resets
# Triggered when WAN interface (eth0) comes online

[ "$ACTION" = "ifup" ] && [ "$INTERFACE" = "wan" ] && /etc/init.d/sqm-fix start
```

### Activation

```bash
chmod +x /etc/hotplug.d/iface/99-sqm-fix
```

**What this does:**
- Monitors WAN interface state
- Automatically runs `sqm-fix` when interface comes up
- No manual intervention needed after LTE reconnects

---

## Script 3: Disable Hardware Offloading

**⚠️ CRITICAL:** This must be run manually **before** enabling the other scripts. Hardware offloading bypasses the CPU entirely, making SQM ineffective.

### Run Once via CLI

Copy and paste this entire block into your SSH session:

```bash
#!/bin/sh
# Disable NSS and Firewall Offloading
# Forces traffic through Linux networking stack

uci set network.globals.nss_offload='0' 2>/dev/null
uci set firewall.@defaults[0].flow_offloading='0'
uci set firewall.@defaults[0].fullcone_nat='0'

# Commit changes to router configuration
uci commit network
uci commit firewall

# Restart services to apply CPU-path routing
/etc/init.d/network restart
/etc/init.d/firewall restart

# Disable proprietary Shortcut Forwarding Engine if present
/etc/init.d/shortcut-fe stop
/etc/init.d/shortcut-fe disable
```

**What this does:**
- Disables NSS (Qualcomm Network Subsystem)
- Disables firewall flow offloading
- Disables Shortcut Forwarding Engine (SFE)
- Restarts network services to apply changes

---

## Complete Installation Checklist

Run these commands in order:

```bash
# 1. SSH into router
ssh root@your.router.ip

# 2. Disable hardware offloading (run once)
uci set network.globals.nss_offload='0' 2>/dev/null
uci set firewall.@defaults[0].flow_offloading='0'
uci set firewall.@defaults[0].fullcone_nat='0'
uci commit network
uci commit firewall
/etc/init.d/network restart
/etc/init.d/firewall restart
/etc/init.d/shortcut-fe stop 2>/dev/null
/etc/init.d/shortcut-fe disable 2>/dev/null

# 3. Create and install sqm-fix init script
vi /etc/init.d/sqm-fix
# (paste Script 1 contents, save with ESC :wq ENTER)
chmod +x /etc/init.d/sqm-fix
/etc/init.d/sqm-fix enable

# 4. Create and install hotplug persistence script
vi /etc/hotplug.d/iface/99-sqm-fix
# (paste Script 2 contents, save with ESC :wq ENTER)
chmod +x /etc/hotplug.d/iface/99-sqm-fix

# 5. Reboot to apply all changes
reboot
```

---

## Verification

After reboot, verify everything is working:

### Check if IFB interface exists
```bash
ip link show ifb4eth0
```
**Expected output:** Should show `ifb4eth0` interface in UP state

### Check if traffic redirection is active
```bash
tc filter show dev eth0 ingress
```
**Expected output:** Should show filter redirecting to `ifb4eth0`

### Check SQM status
```bash
/etc/init.d/sqm status
```
**Expected output:** `running`

### Check hardware offloading is disabled
```bash
uci get network.globals.nss_offload
uci get firewall.@defaults[0].flow_offloading
```
**Expected output:** Both should return `0`

---

## Troubleshooting

### IFB interface doesn't exist after reboot
```bash
# Manually run init script
/etc/init.d/sqm-fix start

# Check if script is enabled
ls -l /etc/rc.d/*sqm-fix
```

### Hardware offloading re-enabled after firmware update
Re-run Script 3 (Hardware Offloading Kill-Switch)

### SQM stops working after LTE reconnect
```bash
# Check if hotplug script exists and is executable
ls -l /etc/hotplug.d/iface/99-sqm-fix

# Manually trigger it
/etc/init.d/sqm-fix start
```

### Need to remove everything and start over
```bash
# Disable and remove scripts
/etc/init.d/sqm-fix stop
/etc/init.d/sqm-fix disable
rm /etc/init.d/sqm-fix
rm /etc/hotplug.d/iface/99-sqm-fix

# Re-enable hardware offloading (optional)
uci set network.globals.nss_offload='1'
uci set firewall.@defaults[0].flow_offloading='1'
uci commit network
uci commit firewall

# Reboot
reboot
```

---

## Technical Explanation

### Why IFB (Intermediate Functional Block)?

Linux's `tc` (traffic control) can only shape **egress** (outgoing) traffic directly. For **ingress** (incoming/download) shaping:

1. Create a virtual IFB interface
2. Redirect incoming WAN traffic to that IFB's egress
3. Apply shaping rules to the IFB's egress (which is really the WAN's ingress)

### Why Disable Hardware Offloading?

Qualcomm's NSS and SFE process packets in hardware before they reach the Linux network stack. This means:
- `tc` rules never see the packets
- SQM/CAKE cannot shape traffic
- Bufferbloat persists despite SQM being "enabled"

By forcing traffic through the CPU, packets flow through the normal Linux path where SQM can intercept and shape them.

### Performance Impact

**Trade-off:** CPU processing reduces maximum throughput by ~10-30% depending on connection speed and router CPU.

**Benefit:** Consistent low latency under load, which matters more than raw speed for:
- Video calls
- Gaming
- VoIP
- Remote desktop
- Real-time applications

---

## Additional Resources

- [OpenWrt SQM Documentation](https://openwrt.org/docs/guide-user/network/traffic-shaping/sqm)
- [Bufferbloat Testing](https://www.waveform.com/tools/bufferbloat)
- [tc-mirred man page](https://man7.org/linux/man-pages/man8/tc-mirred.8.html)
- [IFB Documentation](https://wiki.linuxfoundation.org/networking/ifb)
