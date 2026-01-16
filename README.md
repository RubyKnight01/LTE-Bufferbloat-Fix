📡 LTE Bufferbloat Fix for OpenWrt (GL-AX1800)

This repository provides a reliable, CPU-based SQM setup for LTE connections on OpenWrt devices affected by hardware offloading (NSS / SFE), specifically tested on the GL-AX1800.

The goal is simple:
👉 Fix bufferbloat on LTE connections so latency stays low under load.

❓ The Problem (Why This Exists)

On Qualcomm-based routers, LTE traffic is often:

Offloaded to hardware (NSS / SFE)

Routed around the Linux networking stack

Invisible to SQM / CAKE

This causes:

High latency under load

SQM appearing “enabled” but doing nothing

Speed tests that look fine but real-world performance that feels terrible

✅ What This Fix Does

This setup forces all WAN traffic through the CPU, allowing SQM to work correctly by:

Creating an IFB (Intermediate Functional Block) interface for ingress shaping

Redirecting WAN traffic to that IFB device

Making the setup persistent across reboots and LTE reconnects

Disabling hardware acceleration that bypasses SQM

Result:
Stable latency, working CAKE shaping, and predictable LTE performance.

🧠 Who This Is For

This repo is for you if:

You’re using OpenWrt on a Qualcomm-based router

You rely on LTE / cellular WAN

SQM appears enabled but bufferbloat remains

You’re comfortable running a few shell commands

This is not a beginner networking tutorial — but everything is documented step-by-step.

🧩 What’s in This Repository
File	Description
README.md	High-level explanation and usage
SCRIPTS.md	Copy-ready scripts with full explanations
example.png	Reference / visual context
🚀 High-Level Setup Overview

You will:

Disable hardware acceleration (NSS / SFE)

Install an init script to create the IFB bridge

Add a hotplug script so SQM survives LTE reconnects

Enable the service so it runs on boot

👉 All commands and scripts are documented in detail here:
📄 SCRIPTS.md

⚠️ Important Notes

This setup intentionally reduces raw throughput in exchange for latency control

That trade-off is unavoidable on LTE if you want working SQM

If you re-enable hardware offloading, SQM will stop working

If something breaks, a reboot + removing the scripts restores default behavior.

✅ Tested Environment

Router: GL-AX1800

Firmware: OpenWrt

WAN: LTE / Cellular

SQM: CAKE (ingress + egress)

Other Qualcomm-based OpenWrt routers may work with minimal or no changes.

📌 Final Thoughts

This is not a “tweak” — it’s a correctness fix.

If you depend on LTE for:

Remote work

Video calls

Gaming

General responsiveness under load

…then forcing traffic through the CPU is the only reliable way SQM can do its job.
