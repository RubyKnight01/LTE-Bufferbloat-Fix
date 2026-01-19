#### 📡 LTE Bufferbloat Fix for OpenWrt (GL-AX1800)

This repository provides a reliable, CPU-based SQM setup for LTE connections on OpenWrt devices affected by hardware offloading (NSS / SFE), specifically tested on the GL-AX1800.

The goal is simple:
👉 Fix bufferbloat on LTE connections so latency stays low under load.



#### ❓ The Problem (Why This Exists)

On Qualcomm-based routers, LTE traffic is often:

Offloaded to hardware (NSS / SFE)

Routed around the Linux networking stack

Invisible to SQM / CAKE

This causes:

High latency under load

Speed tests that look fine but real-world performance that feels terrible



#### ✅ What This Fix Does

This setup forces all WAN traffic through the CPU, allowing SQM to work correctly by:

Creating an IFB (Intermediate Functional Block) interface for ingress shaping

Redirecting WAN traffic to that IFB device

Making the setup persistent across reboots and LTE reconnects

Disabling hardware acceleration that bypasses SQM

Result:
Stable latency, working CAKE shaping, and predictable LTE performance.



#### 🧠 Who This Is For

This repo is for you if:

You’re using OpenWrt on a Qualcomm-based router

You rely on LTE / cellular WAN

SQM appears enabled but bufferbloat remains

You’re comfortable running a few shell commands

This is not a beginner networking tutorial — but everything is documented step-by-step.



#### 🧩 What’s in This Repository

README.md	👉 High-level explanation and usage

SCRIPTS.md 👉 Copy-ready scripts with full explanations

example.png 👉 Reference / visual context



#### 🚀 High-Level Setup Overview

You will:

Disable hardware acceleration (NSS / SFE)

Install an init script to create the IFB bridge

Add a hotplug script so SQM survives LTE reconnects

Enable the service so it runs on boot


## FIRST 👉👉👉 For SQM to work, set the connection speed slightly below its maximum capacity. This prevents the LTE modem's buffer from filling up.


### How to calculate your speeds:
1. Run 3-5 speed tests at different times of the day.
2. Find your **lowest** consistent Download and Upload speeds.
3. Set your SQM limits to **85-90%** of those values.

**Example:** If you usually get 30Mbps down, set your SQM to `25000` (25Mbps).

### Apply Speeds via CLI:
Run these commands to set your target speeds (in kbit/s):

```bash
# Set Download to 25Mbps and Upload to 10Mbps
uci set sqm.eth0.download='25000'
uci set sqm.eth0.upload='10000'
uci commit sqm
/etc/init.d/sqm restart
```

## 👉👉 Create the SQM Init Script File 👈👈

Before applying any fixes, you need to create the init script file that OpenWrt will execute.

1️⃣ SSH into your router

```
ssh root@yourrouterip
```

2️⃣ Create the init script file

Use a text editor such as vi or nano:

```
vi /etc/init.d/sqm-fix

vi /etc/init.d/99-sqm-fix
```

3️⃣ Paste the script contents into their respective files

Copy the entire SQM Initialization Script from SCRIPTS.md📄

, then save and exit. (ESC > :wq > ENTER)

👉 All commands and scripts are documented in detail here: ##### 📄 SCRIPTS.md

4️⃣ Make the script executable
```
chmod +x /etc/init.d/sqm-fix
```

5️⃣ Enable the script to run on boot
```
/etc/init.d/sqm-fix enable
```

At this point, the init script is installed and ready to run.




#### ⚠️ Important Notes

This setup intentionally reduces raw throughput in exchange for latency control

That trade-off is unavoidable on LTE if you want working SQM

If you re-enable hardware offloading, SQM will stop working

If something breaks, a reboot + removing the scripts restores default behavior.



#### ✅ Tested Environment

Router: GL-AX1800

Firmware: OpenWrt

WAN: LTE / Cellular

SQM: CAKE (ingress + egress)

Other Qualcomm-based OpenWrt routers may work with minimal or no changes.



#### 📌 Final Thoughts

This is not a “tweak” — it’s a correctness fix.

If you depend on LTE for:

Remote work

Video calls

Gaming

General responsiveness under load

…then forcing traffic through the CPU is the only reliable way SQM can do its job.
