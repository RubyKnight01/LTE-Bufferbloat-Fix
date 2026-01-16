Automation & Configuration Scripts

This document contains the scripts necessary to initialize the virtual bridge, ensure persistence across reboots, and disable hardware acceleration on the GL-AX1800.

1. SQM Initialization Script

File Path:

    /etc/init.d/sqm-fix

Purpose: Handles the manual creation of the IFB virtual bridge and forces the redirection of WAN traffic to the CPU for shaping.
Bash

    #!/bin/sh /etc/rc.common

# This script creates a virtual IFB device for Ingress (Download) shaping
# and redirects eth0 traffic to it so SQM/CAKE can process it.

### 1. SQM Initialization Script
**File Path:** `/etc/init.d/sqm-fix`

```bash
#!/bin/sh /etc/rc.common

START=99

start() {
    echo "Initializing SQM Bridge for Qualcomm NSS bypass..."

    # 1. Create the virtual device using the SQM naming convention
    ip link add name ifb4eth0 type ifb 2>/dev/null
    ip link set dev ifb4eth0 up

    # 2. Attach the Ingress Qdisc hook to the physical WAN port (eth0)
    tc qdisc add dev eth0 handle ffff: ingress 2>/dev/null

    # 3. Redirect all traffic from eth0 to the virtual device ifb4eth0
    tc filter add dev eth0 parent ffff: protocol all u32 match u32 0 0 \
    action mirred egress redirect dev ifb4eth0 2>/dev/null

    # 4. Restart the SQM service to pick up the new interface
    /etc/init.d/sqm restart

    echo "SQM Bridge successfully linked to ifb4eth0."
}

stop() {
    echo "Tearing down SQM Bridge..."
    ip link delete ifb4eth0 2>/dev/null
}---



2. Hotplug Persistence Script

File Path: 

    /etc/hotplug.d/iface/99-sqm-fix

Purpose: Ensures the shaper is "event-driven." It re-applies the bridge automatically if the LTE connection resets or the interface toggles.
Bash

    #!/bin/sh

    # Hotplug script to ensure SQM persists after interface resets.
    # Triggered when the WAN interface (eth0) comes online.

    [ "$ACTION" = "ifup" ] && [ "$INTERFACE" = "wan" ] && /etc/init.d/sqm-fix start


3. Hardware Acceleration Kill-Switch

Execution: Run via CLI (Manual Setup)

Purpose: Disables the Qualcomm NSS and Shortcut Forwarding Engine (SFE). This is mandatory to prevent packets from "skipping" the CPU and ignoring SQM rules.
Bash

    #!/bin/sh

    # Disable NSS and Firewall Offloading
    # This forces traffic through the Linux networking stack
    uci set network.globals.nss_offload='0' 2>/dev/null
    uci set firewall.@defaults[0].flow_offloading='0'
    uci set firewall.@defaults[0].fullcone_nat='0'

    # Commit changes to the router's configuration
    uci commit network
    uci commit firewall

    # Restart services to apply CPU-path routing
    /etc/init.d/network restart
    /etc/init.d/firewall restart

    # Disable the proprietary Shortcut Forwarding Engine if present
    /etc/init.d/shortcut-fe stop
    /etc/init.d/shortcut-fe disable

Implementation Notes

Permissions: After creating these files on your router, you must make them executable: 

    chmod +x /etc/init.d/sqm-fix

Activation: To enable the service to start on boot: 

    /etc/init.d/sqm-fix enable
