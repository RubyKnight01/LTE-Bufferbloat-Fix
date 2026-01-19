# LTE Bufferbloat Fix for OpenWrt (GL.iNet GL-AX1800 for this example/guide)

**Fix high latency on LTE connections by forcing traffic through CPU-based SQM/CAKE shaping.**

Tested on GL-AX1800 running OpenWrt with LTE/cellular WAN.

---

## The Problem

Qualcomm-based routers offload LTE traffic to hardware (NSS/SFE), bypassing the Linux network stack. This makes SQM invisible to traffic, causing:
- High latency under load
- Poor real-time performance (calls, gaming, remote work)
- Speed tests look fine, but actual experience is terrible

## The Solution

This fix forces WAN traffic through the CPU using:
- IFB (Intermediate Functional Block) interface for ingress shaping
- Disabled hardware acceleration (NSS/SFE)
- Persistent scripts that survive reboots and LTE reconnects

**Trade-off:** Lower raw throughput in exchange for stable, low latency.

---

## Quick Start

### Prerequisites
- OpenWrt router 
- LTE/cellular WAN connection
- SSH access to router
- Basic command line knowledge

### Step 1: Calculate Your SQM Speeds

Run 3-5 speed tests at different times. Use your **lowest consistent speeds** and set SQM to **85-90%** of those values.

**Example:** If you get 30 Mbps down / 12 Mbps up → Set SQM to 25 Mbps / 10 Mbps

```bash
# Set speeds (values in kbit/s)
uci set sqm.eth0.download='25000'  # 25 Mbps
uci set sqm.eth0.upload='10000'    # 10 Mbps
uci commit sqm
/etc/init.d/sqm restart
```

### Step 2: Install Init Scripts

**SSH into your router:**
```bash
ssh root@your.router.ip
```

**Create the init script files:**
```bash
vi /etc/init.d/sqm-fix
vi /etc/init.d/99-sqm-fix
```

**Copy script contents from [SCRIPTS.md](SCRIPTS.md)**

Press `i` to edit, paste content, then save with `ESC` → `:wq` → `ENTER`

**Make scripts executable and enable on boot:**
```bash
chmod +x /etc/init.d/sqm-fix
chmod +x /etc/init.d/99-sqm-fix
/etc/init.d/sqm-fix enable
/etc/init.d/99-sqm-fix enable
```

**Reboot to apply:**
```bash
reboot
```

### Step 3: Verify It's Working

After reboot, check if IFB interface exists:
```bash
ip link show ifb4eth0
```

Check SQM status:
```bash
/etc/init.d/sqm status
```

Run a bufferbloat test at [waveform.com/tools/bufferbloat](https://www.waveform.com/tools/bufferbloat)

---

## Repository Contents

| File | Description |
|------|-------------|
| `README.md` | This file - setup guide and overview |
| `SCRIPTS.md` | Complete init scripts with detailed explanations |
| `example.png` | Before/after bufferbloat test results |
| `example2.png` | Additional visual reference |
| `example3.png` | Configuration screenshots |

---

## Important Notes

⚠️ **This intentionally reduces throughput for latency control** - unavoidable trade-off for working SQM on LTE

⚠️ **Re-enabling hardware offloading breaks SQM** - don't enable NSS/SFE

⚠️ **Rollback:** If something breaks, remove scripts and reboot to restore defaults

✅ **Best for:** Remote work, video calls, gaming, or any latency-sensitive use case

---

## Tested Environment

- **Router:** GL-AX1800
- **Firmware:** OpenWrt
- **WAN:** LTE / Cellular
- **QoS:** SQM with CAKE qdisc (ingress + egress)

Other Qualcomm-based OpenWrt routers should work with minimal changes.

---

## Troubleshooting

**SQM not working after setup?**
- Check if hardware offloading is disabled
- Verify IFB interface exists: `ip link show ifb4eth0`
- Check logs: `logread | grep sqm`

**Speeds too slow?**
- Your SQM limits may be too conservative
- Increase to 90-95% of your lowest test results

**LTE reconnects break SQM?**
- Ensure `99-sqm-fix` hotplug script is installed
- Check if it's executable: `ls -l /etc/init.d/99-sqm-fix`

**Need to revert everything?**
```bash
/etc/init.d/sqm-fix disable
rm /etc/init.d/sqm-fix
rm /etc/init.d/99-sqm-fix
reboot
```

---

## Contributing

Found a bug? Have improvements? Open an issue or PR.

This is a homelab project - community input helps improve it for everyone.

---

## Acknowledgments

Was done with the help of AI (not gonna lie) but everybit of troubleshooting was done to fix issues that came up until the fix worked as intended.
