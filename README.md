Key Features

CAKE (Common Applications Kept Enhanced): Used for its superior "triple-isolate" logic, which ensures fair bandwidth distribution even when multiple devices are streaming/gaming.

Adaptive Persistence: Custom scripts in init.d and hotplug.d ensure the virtual bridge (ifb4eth0) survives reboots and LTE connection drops.

Overhead Compensation: Configured to handle the variable nature of LTE packet encapsulation.

=============================================================

Troubleshooting Log (Homelab Notes)

During implementation, several hurdles were cleared:

Issue: tc -s qdisc showed 0 bytes even when downloading.

Fix: Identified that Shortcut Forwarding Engine (SFE) was active; disabled via UCI and firewall settings.

Issue: Interface ifb0 disappeared on reboot.

Fix: Switched to a Hotplug script that listens for the wan interface "up" event to trigger the bridge creation.

Issue: SQM service naming conflict.

Fix: Discovered through kernel logs that SQM expected the naming convention ifb4eth0, and updated manual scripts to match.

=============================================================

Verification & Metrics

To verify the shaper is active, run the following command during a speed test:

    tc -s qdisc show dev ifb4eth0

Success Criteria:

qdisc cake is visible as the active discipline.

Sent [X] bytes is actively increasing.

backlog shows small values, indicating CAKE is actively managing the queue.

Verify NSS Offloading is disabled. If it returns nothing or shows the modules are not in use, it confirms the "hack" is active:

    lsmod | grep nss

============================================================

How to use:

Upload scripts to /etc/init.d/ and /etc/hotplug.d/iface/.

Ensure script permissions are executable: 

    chmod +x /etc/init.d/sqm-fix.

Disable hardware acceleration in the OpenWrt firewall settings.

Enable the service: 

    /etc/init.d/sqm-fix enable.
