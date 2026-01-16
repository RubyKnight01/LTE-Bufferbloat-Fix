📜 Automation & Configuration Scripts

This document contains the scripts required to:

Initialize the IFB virtual bridge for SQM

Ensure persistence across reboots and interface resets

Disable hardware acceleration on the GL-AX1800 so SQM/CAKE works correctly

Each section includes:

Purpose & file path

Instructions

Copy-ready scripts in code blocks

---------------------------------------

1️⃣ SQM Initialization Script
📍 File Path

'''bash
/etc/init.d/sqm-fix

🎯 Purpose

Creates a virtual IFB device for ingress (download) shaping and redirects WAN traffic through the CPU so SQM/CAKE can process it correctly.
This bypasses Qualcomm NSS hardware offloading.

📄 Script (copy exactly)

'''bash
#!/bin/sh /etc/rc.common

# This script creates a virtual IFB device for Ingress (Download) shaping
# and redirects eth0 traffic to it so SQM/CAKE can process it.

START=99

start() {
    echo "Initializing SQM Bridge for Qualcomm NSS bypass..."

    # 1. Create the virtual device using the SQM naming convention
    ip link add name ifb4eth0 type ifb 2>/dev/null
    ip link set dev ifb4eth0 up

    # 2. Attach the ingress qdisc to the physical WAN port (eth0)
    tc qdisc add dev eth0 handle ffff: ingress 2>/dev/null

    # 3. Redirect all traffic from eth0 to the virtual device ifb4eth0
    tc filter add dev eth0 parent ffff: protocol all u32 match u32 0 0 \
        action mirred egress redirect dev ifb4eth0 2>/dev/null

    # 4. Restart SQM so it detects the new interface
    /etc/init.d/sqm restart

    echo "SQM Bridge successfully linked to ifb4eth0."
}

stop() {
    echo "Tearing down SQM Bridge..."
    ip link delete ifb4eth0 2>/dev/null
}

