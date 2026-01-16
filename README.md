Key Features

CAKE (Common Applications Kept Enhanced): Used for its superior "triple-isolate" logic, which ensures fair bandwidth distribution even when multiple devices are streaming/gaming.

Adaptive Persistence: Custom scripts in init.d and hotplug.d ensure the virtual bridge (ifb4eth0) survives reboots and LTE connection drops.

Overhead Compensation: Configured to handle the variable nature of LTE packet encapsulation.

================================================================

🛠️ Prerequisites & Environment

This guide is designed for OpenWrt users who are comfortable using the Command Line Interface (CLI) via SSH.

1. Hardware & OS

Device: GL.iNet GL-AX1800 (Flint) or similar Qualcomm-based routers.

Firmware: OpenWrt 21.02 or newer (Snapshot builds confirmed working).

Access: SSH access (using Terminal, PuTTY, or PowerShell).


2. Required Packages

The SQM scripts will not function unless the base SQM and CAKE packages are installed. If they are missing from your build, run the following commands while your router has internet access:
Bash

opkg update
opkg install lucid-app-sqm sqm-scripts sqm-scripts-extra tc-full kmod-sched-cake kmod-ifb


3. Command Line First

While some of these settings can be seen in the LuCI web interface, this specific fix requires the CLI. The proprietary hardware acceleration on many GL.iNet/Qualcomm devices often hides the "Disable" toggle in the GUI, making the manual UCI commands provided in this repo mandatory.
🚀 Quick Start (CLI Only)

Install the packages listed above.

Disable Acceleration using the disable-nss-acceleration.sh script.

Configure your speeds (based on 90% of your LTE baseline).

Deploy the persistence scripts to ensure your shaper survives a reboot.

=======================================================================================

⚙️ Configuration & Speed Tuning

Before deploying the scripts, you must define your bandwidth targets. Unlike fixed-line broadband, LTE requires a more strategic approach to speed setting.


1. Identify Your "True" Baseline

Run several speed tests at different times of the day (morning, afternoon, and peak evening hours).

The Golden Rule: Set your SQM speeds to 85-90% of your lowest consistent result.

If you usually get 30Mbps but it drops to 25Mbps during peak hours, use 22Mbps as your base.


2. Check Your ISP's Fair Usage Policy (FUP)

Many LTE providers implement "Data Caps" or "Deprioritization" after a certain GB threshold.

Throttling: If your provider drops your speed to 1Mbps after 100GB, SQM will actually cause more lag because the shaper is trying to allow 22Mbps through a 1Mbps pipe.

Recommendation: Monitor your data usage. If you hit an FUP limit, you must manually update the SQM config to match the throttled speed.


3. Applying Your Speeds
To set your specific download and upload speeds (in Kilobits), run the following commands on your router:
Bash

Example for 22Mbps Down and 8Mbps Up
uci set sqm.eth0.download='22000'
uci set sqm.eth0.upload='8000'
uci commit sqm
/etc/init.d/sqm restart

Optimization Tips for LTE
Scenario	Adjustment
High Jitter/Ping Spikes	Lower the download speed by another 5-10%.
Bufferbloat "C" or lower	Ensure linklayer is set to none in /etc/config/sqm for LTE.
Video Buffering	Slightly increase the download limit, but monitor the av_delay in tc -s qdisc.

===================================================================================

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
