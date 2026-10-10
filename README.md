# ASUS Wi-Fi 7 router for PC VR: minimal-stutter setup

A practical setup guide for a fresh ASUS Wi-Fi 7 router (tested on the **TUF-BE9400**, Qualcomm platform, stock ASUSWRT 3.0.0.6.102). The goal: a plain router that does routing, DHCP and Wi-Fi, with nothing in the background waking up and stealing airtime while you stream VR to a Quest 3.

Who this is for: you stream PC VR (Virtual Desktop, Steam Link) over Wi-Fi and want as few micro-stutters as possible. You don't need roaming, mesh, cloud features or traffic analytics.

Everything here works on stock firmware: the web UI plus SSH, without Merlin. The SSH part is optional, but it removes periodic stalls the UI can't turn off.

---

## Typical results

Synthetic VR-like stream: PC (wired) → Quest 3, 250 Mbit/s, 90 fps. A **late frame** is one that fully arrived more than one frame interval (11.1 ms) after it was sent.

| Setup | Late frames | Median latency | p99 | Worst |
|---|---|---|---|---|
| Out of the box (Wi-Fi 7 mode, all services on) | ~0.1 %, plus periodic 10–40 ms stalls | 5.9 ms | 7.9 ms | 25–40 ms |
| **This guide** (6 GHz) | **~0 %** | 5.2 ms | 6.6 ms | ~10 ms |
| **This guide + DSCP marking on the PC** | **~0 %** | **4.7 ms** | **5.6 ms** | ~9 ms |

At 450 Mbit/s the tuned setup stays under 0.05 % late frames with p99 ≈ 8.5 ms. No packet loss in either case.

For comparison: a simple older Wi-Fi 6 router on 5 GHz matches this only while its channel is quiet. When neighbouring networks on the same channel get busy, it gets 10–50× worse. 6 GHz usually has no neighbours at all.

---

## Step 1. Web UI

Names differ slightly between firmware versions.

**Wireless → General**
- **Smart Connect:** off. Give each band its own network name, so the headset stays on one band.
- **Wi-Fi 7 / 802.11be:** off on every band. Quest 3 is a Wi-Fi 6E client and gains nothing from it, while the Wi-Fi 7 code path adds ~0.6 ms of latency.
- **6 GHz:** fixed channel (a PSC channel, e.g. 5, 21, 37…), width **80 MHz**. Quest 3 can't use 160 MHz, and a wider channel only spreads the transmit power thinner.
- **5 GHz:** fixed channel with the fewest neighbours, 80 MHz.

**Wireless → Professional** (for the VR band at least)
- **Roaming assistant:** off.
- **OFDMA / MU-MIMO:** off. They only help with many simultaneous clients; with one VR client they slightly raise the worst-case latency.
- **Airtime fairness, Target Wake Time (TWT):** off.
- **WPS:** off.

**Everything else you don't use: off.**
- AiProtection, Adaptive QoS / Traffic Analyzer, Parental controls;
- AiMesh (don't add nodes);
- USB applications: media server (DLNA), Samba, FTP, cloud;
- remote access from the ASUS Router app, DDNS, VPN server;
- UPnP, if you don't need it.

---

## Step 2. SSH: turn off what the UI can't

Enable SSH: **Administration → System → Enable SSH (LAN only)**, key-based login preferred.

ASUSWRT respawns its background daemons. `init` starts them at boot, and `watchdog` checks every second and restarts many of them. Killing them is pointless, and only a few have a real off switch.

### 2a. Persistent flags (once)

```sh
nvram set no_asd=1               # asd: security daemon, not restarted
nvram set aae_disable_force=1    # mastiff: app remote access
nvram set disable_amas_lib=1     # AiMesh library (breaks AiMesh)
nvram set stop_amas_lldpd=1      # AiMesh node discovery (breaks AiMesh)

# Wi-Fi 7 off, including the hidden AiMesh backhaul profile apm1.
# If any network on a radio keeps Wi-Fi 7, the whole radio boots in Wi-Fi 7.
nvram set apm1_11be=0
nvram set wl0_11be=0; nvram set wl1_11be=0; nvram set wl2_11be=0

nvram commit
reboot
```

Check after the reboot. You want `HE` (Wi-Fi 6), not `EHT` (Wi-Fi 7):

```sh
for v in ath001 ath101 ath201; do iwpriv $v get_mode; done
```

If it still says `EHT`, list the per-network profiles and set `11be=0` on every enabled one:

```sh
nvram show | grep -E '^apm[0-9]+_(ssid|enable|11be)='
```

Saving the wireless page in the web UI can switch Wi-Fi 7 back on, so re-check after using it.

### 2b. After every reboot

These don't survive a reboot. Save the script below as `/jffs/scripts/after_boot.sh` and run `sh /jffs/scripts/after_boot.sh` about 1–2 minutes after the router comes up. It waits for the services to start if you run it too early.

```sh
#!/bin/sh
# Wait until watchdog has started everything, otherwise it starts services after we stop them
up=$(cut -d. -f1 /proc/uptime); [ "$up" -lt 90 ] && sleep $((90 - up))

# 1. Hidden service networks (AiMesh backhaul / onboarding) beacon even without mesh nodes
for i in ath0 ath1 ath2 ath002; do
  hostapd_cli -p /var/run/hostapd -i $i disable >/dev/null 2>&1
done

# 2. Daemons nothing restarts: just end them
for n in asd sysstate ahs wlc_nt wlc_monitor nbr_monitor erp_monitor bwdpi_check scs_tool mcsd rstats; do
  p=$(pidof $n) && { kill -TERM $p; kill -CONT $p; }
done

# 3. Daemons watchdog restarts without a flag: freeze them (SIGSTOP).
#    A frozen process still "exists", so watchdog leaves it alone.
#    net_log writes to flash every minute and runs a network check every 10 min.
for n in net_log roamast conn_diag networkmap cfg_server amas_lanctrl amas_portstatus amas_ssd_cd \
         awsiot nt_center nt_monitor wpsaide lld2d infosvr fsmd netool; do
  p=$(pidof $n) && kill -STOP $p
done

# 4. Check
for v in ath001 ath101 ath201; do iwpriv $v get_mode; done
for i in ath0 ath1 ath2 ath002; do
  printf '%s: ' $i; iwconfig $i | grep -oE 'Access Point: [^ ]+'
done
```

`Access Point: Not-Associated` means the service network is really off.

To undo the freezing without rebooting:

```sh
for n in net_log roamast conn_diag networkmap cfg_server amas_lanctrl amas_portstatus amas_ssd_cd awsiot nt_center nt_monitor wpsaide lld2d infosvr fsmd netool; do p=$(pidof $n) && kill -CONT $p; done
```

### Do NOT stop these

| Process | Why |
|---|---|
| `protect_srv` | SSH logins wait for it. Stop it and you lock yourself out until a reboot |
| `PS_pod` | Stopping it broke Wi-Fi until a reboot |
| `hostapd*`, `dnsmasq`, `wanduck`, `pppd`/`udhcpc`, `ntp` | This is the router |
| `httpd`, `dropbear` | Your way back in |
| `watchdog`, `hwdog`, `thermald`, `cnssdaemon` | Hardware watchdog, thermal control, radio firmware |

If you experiment beyond this list, stop **one process at a time** and check that SSH and Wi-Fi still work after each.

Freezing is a workaround: a frozen daemon holds its sockets, and something talking to it could hang. The web UI's client list and notifications stop updating while they are frozen. A reboot restores everything.

---

## Step 3. PC: give the VR stream priority (DSCP)

Wi-Fi has four priority queues (WMM). Packets marked DSCP 40 go to the **Video** queue and win channel access over normal traffic, including the headset's own background uploads.

Windows ignores DSCP that an application sets itself, so use a QoS policy (admin PowerShell):

```powershell
New-NetQosPolicy -Name "VD-video" -AppPathNameMatchCondition "VirtualDesktop.Streamer.exe" -DSCPAction 40 -NetworkProfile All
# Needed on PCs that are not in a domain, otherwise the policy may be ignored:
New-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\QoS" -Force | Out-Null
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\QoS" -Name "Do not use NLA" -Value "1" -Type String
```

Remove it with `Remove-NetQosPolicy -Name "VD-video" -Confirm:$false`.

Virtual Desktop streams video over TCP, so don't restrict the policy to UDP. For another streamer, point the policy at the process that sends the video.

Check on the router that it works. The **Video** counter should grow while you stream (`ath201` is the 6 GHz network; use `ath101` for 5 GHz):

```sh
apstats -v -i ath201 | sed -n '/Tx Data Packets per AC/,/Voice/p'
```

The effect: −0.5 ms median and −1 ms p99, at both 250 and 450 Mbit/s.

Also on the PC:
- Avoid heavy background activity while streaming (opening apps, a live Wireshark window). A busy PC produces stalls that look exactly like network stalls.
- On CPUs with E-cores, keeping the streamer on P-cores can help (Process Lasso or affinity settings).

---

## Step 4. Placement and the headset

- **Same room, antennas visible to where you play.** 6 GHz is blocked by walls and bodies more than 5 GHz. A router hidden behind furniture works, but raising it above head height gives margin for when you turn your back to it. Aim for a signal of −50 dBm or better at the headset.
- **Quest 3 regional 6 GHz bug.** The headset picks its country from neighbouring networks. For some countries Meta's table has no 6 GHz, so the headset drops 6 GHz after a while or right after connecting. Fix for a while: Wi-Fi off → reboot the headset → Wi-Fi on. Nothing on the router side helps. If it keeps happening, use the 5 GHz network.
- **Headset asleep = Wi-Fi power save.** When testing with the headset off your head, the display turns off. The router then queues frames, and latency jumps to 100–300 ms. That is not a router problem.
- **Small periodic blips are normal.** The headset uploads data in the background about once a minute, which can cost a frame or two. DSCP marking (step 3) is the remedy.

---

## What didn't help

| Tried | Result |
|---|---|
| ANI (adaptive noise immunity) off | Worse |
| MU-EDCA off | Periodic degraded throughput |
| Fixed MCS (no rate adaptation) | Worse worst case |
| Frameburst on, beamforming off | No change |
| Freezing `cnssdaemon` | No measurable change; not worth the risk |
| Fighting `watchdog` | Not needed: it uses ~2 % of one core, never touches the radio, and leaves frozen daemons alone |
| 160 MHz on 6 GHz | Quest 3 maxes out at 80 MHz |
