# HomeNET — OpenWrt Setup Manual
**Router:** Cudy WR3000 v1 &nbsp;·&nbsp; **ISP:** Exord Online 70 Mbps (PPPoE via ONU) &nbsp;·&nbsp; **OS:** OpenWrt 24.10.x

---

## Does CAKE SQM Actually Fix Gaming Ping During Torrents / Downloads?

**Yes — and this is the whole point of setting it up.**

Without SQM, when your torrent or a family member's download saturates the connection, every packet from every device (including your game) queues up behind the bulk traffic at your ISP's equipment. You have zero control there. Ping spikes to 300–500ms. That's **bufferbloat**.

CAKE SQM moves the bottleneck from your ISP's buffers to *inside your router*, where it actively manages the queue. It uses fair per-flow queuing — your gaming packets (small, time-sensitive) never wait behind torrent packets (large, bulk). The result: **sustained 10–15ms ping even while someone else maxes out the connection.** It works for:

- Torrents running in background on your PC ✅
- Sister downloading a large file ✅
- Mom/Dad streaming 4K video simultaneously ✅

The only requirement: CAKE must be set to ~90% of your real WAN speed, and hardware offloading must be **off**. Both are covered in this guide.

---

## ⚠️ Hardware Check — Do This Before Anything Else

> **This guide is for Cudy WR3000 v1 only.**
> - **v1** = MediaTek MT7981B SoC + MT7915 radio → ✅ Fully supported by OpenWrt
> - **v2** = TRIDuctor SoC → ❌ NOT supported — flashing will brick it permanently
>
> Check the label on the bottom. If your serial number starts with `2543` or later, use **OpenWrt 24.10.5+** (different NAND chip revision).

---

## Network Architecture

This is what you're building. Read it once so every step makes sense.

```
INTERNET (Exord Online, 70 Mbps)
    │
    │ Fiber → ONU (Bridge Mode) → PPPoE
    │
[Cudy WR3000 — OpenWrt]
    │
    ├── VLAN 1  →  192.168.1.0/24   LAN      (your devices, family)
    ├── VLAN 10 →  192.168.10.0/24  GUEST    (internet-only, isolated)
    └── VLAN 20 →  192.168.20.0/24  IOT      (internet-only, isolated)
                                               └── Xiaomi TV Box S3

WiFi SSIDs:
    📶 "HomeNET"         → VLAN 1  · 2.4 + 5 GHz (same SSID, band steering)
    📶 "HomeNET-Family"  → VLAN 1  · 2.4 + 5 GHz (separate password for family)
    📶 "HomeNET-Guest"   → VLAN 10 · 2.4 GHz only · isolated
    📶 "HomeNET-IoT"     → VLAN 20 · 2.4 GHz only · isolated · hidden

Traffic rules:
    LAN   ↔  WAN   ✅ Full access
    GUEST →  WAN   ✅ Internet only   |   GUEST → LAN ❌ Blocked
    IoT   →  WAN   ✅ Internet only   |   IoT   → LAN ❌ Blocked
    LAN   →  GUEST ❌ Blocked         |   LAN   → IoT ❌ Blocked

QoS tiers (CAKE layer_cake):
    CS6  →  Your gaming PC — highest priority
    CS4  →  Your phone
    CS0  →  Family devices — best effort (default)
    CS1  →  Torrent / bulk traffic — lowest, never starves gaming

DNS:  Clients → dnsmasq → https-dns-proxy → Cloudflare DoH (Quad9 backup)
Adblock: adblock-fast (DNS sinkhole, ~5MB RAM, blocklists via cron)
Roaming: DAWN daemon + 802.11k/r/v (clients auto-switch 2.4↔5 GHz)
```

### IP Address Plan

| Network | Subnet | Gateway | DHCP Range |
|---------|--------|---------|------------|
| LAN | 192.168.1.0/24 | 192.168.1.1 | .50 – .200 |
| Guest | 192.168.10.0/24 | 192.168.10.1 | .50 – .150 |
| IoT | 192.168.20.0/24 | 192.168.20.1 | .50 – .100 |

**Reserved static IPs (fill in your MACs later):**

| Device | Network | IP | MAC |
|--------|---------|----|----|---|
| Your gaming PC | LAN | 192.168.1.10 | `aa:bb:cc:dd:ee:ff` |
| Your phone | LAN | 192.168.1.11 | `aa:bb:cc:dd:ee:ff` |
| Xiaomi TV Box S3 | IoT | 192.168.20.10 | `aa:bb:cc:dd:ee:ff` |

---

## Packages to Install (Full List)

You'll install these across the setup steps below. Listed here for reference:

```bash
# WiFi (replaces stripped default)
wpad-wolfssl

# LuCI security
luci-ssl

# DNS + Adblock
https-dns-proxy  luci-app-https-dns-proxy
adblock-fast     luci-app-adblock-fast
banip            luci-app-banip

# QoS
sqm-scripts      luci-app-sqm       kmod-sched-cake
luci-app-qosmate
qosify

# Band steering
dawn

# Guest QR code
qrencode

# Quality of life
watchcat         luci-app-watchcat   # connection watchdog
vnstat2          luci-app-vnstat2    # bandwidth usage graphs
ddns-scripts     luci-app-ddns       # dynamic DNS
wireguard-tools  kmod-wireguard  luci-app-wireguard  # remote VPN access
luci-app-wol                         # wake gaming PC remotely
luci-app-attendedsysupgrade          # easy firmware updates
```

**Check flash space after installing everything:**
```bash
df -h /
# Root partition should have at least 10MB free.
# If tight, skip vnstat2 and luci-app-wireguard (manage WireGuard via SSH instead).
```

---

## Setup Steps

Follow in order. Do not skip ahead.

---

### Step 0 — Flash OpenWrt

1. Download **OpenWrt 24.10.x** for Cudy WR3000 v1:
   `https://downloads.openwrt.org/releases/24.10.1/targets/mediatek/filogic/`
   File: `openwrt-24.10.x-mediatek-filogic-cudy_wr3000-v1-squashfs-factory.bin`

2. Connect your PC to the router via **Ethernet cable** (not WiFi — WiFi is off by default after flash).

3. Access Cudy's stock firmware at `http://192.168.10.1` → find Firmware Upgrade → upload the file.

4. Wait ~3 minutes for the reboot. OpenWrt is now at `http://192.168.1.1`.

---

### Step 1 — Initial Security

```bash
# SSH into the router (no password yet on fresh flash)
ssh root@192.168.1.1

# Set root password
passwd

# Update package list and install HTTPS LuCI
opkg update
opkg install luci-ssl
```

From now on, access the web UI at `https://192.168.1.1` (accept the self-signed cert warning).

In LuCI → **System → Administration**:
- Set SSH to listen on `lan` interface only (not WAN)
- Disable SSH password auth once you've added your SSH public key (optional)

---

### Step 2 — Replace wpad (Required for Band Steering)

The default `wpad-basic-wolfssl` package strips out 802.11k/r/v support. Replace it **before** configuring WiFi:

```bash
opkg remove wpad-basic-wolfssl
opkg install wpad-wolfssl
```

> If `wpad-wolfssl` shows as not found, try `wpad-openssl`. Either works.

---

### Step 3 — WAN: PPPoE over ONU

Your setup: **Fiber → ONU → Router (PPPoE)**

**First: Set your ONU to Bridge Mode.**
Log into your ONU's admin page (usually `192.168.1.1` or `192.168.100.1` — check label). Find the WAN connection and set it to Bridge or IPoE bridge mode. This makes the ONU a transparent pass-through so your OpenWrt router handles the PPPoE session directly. If you can't find this setting, call Exord — they can enable it remotely.

**Then configure WAN on the router:**

```bash
uci set network.wan.proto='pppoe'
uci set network.wan.username='your_exord_username'
uci set network.wan.password='your_exord_password'
uci set network.wan.mtu='1492'         # PPPoE subtracts 8 bytes from 1500
uci set network.wan.keepalive='4 20'   # LCP echo: 4 retries, 20s interval — detects dead sessions fast
uci commit network
service network restart
```

**Enable MSS clamping** (prevents broken page loads on PPPoE — essential):

In LuCI → **Network → Firewall → Zones** → edit the **wan** zone → check **"MSS clamping"** → Save & Apply.

Or via UCI:
```bash
uci set firewall.@zone[1].mtu_fix='1'
uci commit firewall
service firewall restart
```

**Verify connection:**
```bash
ping -c 4 8.8.8.8     # should work
ip addr show pppoe-wan # should show your public IP
```

> **If ONU can't be set to bridge mode:** Your ONU will handle PPPoE and give your router a LAN IP (double-NAT). Gaming will still work fine but you'll lose port forwarding to your PC — and WireGuard remote access also won't work without additional port forwarding on the ONU. Ask Exord to enable bridge mode — it's a standard request.

**IPv6 — disable if Exord doesn't provide it:**
Check if you have an IPv6 address after connecting: `ip -6 addr show pppoe-wan`. If you see a global address, IPv6 is working and OpenWrt handles it automatically. If not (most common in BD), disable it to prevent leaked IPv6 traffic bypassing your VLAN rules:
```bash
uci set network.wan6.disabled='1'
uci commit network
service network restart
```
This only disables the IPv6 WAN — internal LAN still works normally.

---

### Step 4 — VLANs (DSA Bridge VLANs)

> OpenWrt 21.02+ uses DSA (Distributed Switch Architecture). Do NOT use the old `config switch` syntax.

Edit `/etc/config/network`. Find the existing `lan` interface section and replace the entire LAN block with this:

```uci
# Bridge device with VLAN filtering
config device
    option name 'br-lan'
    option type 'bridge'
    option vlan_filtering '1'
    list ports 'lan1'
    list ports 'lan2'
    list ports 'lan3'
    list ports 'lan4'

# VLAN 1 — Main LAN (untagged on all physical ports)
config bridge-vlan
    option device 'br-lan'
    option vlan '1'
    list ports 'lan1'
    list ports 'lan2'
    list ports 'lan3'
    list ports 'lan4'

# VLAN 10 — Guest (WiFi only, no physical port needed)
config bridge-vlan
    option device 'br-lan'
    option vlan '10'

# VLAN 20 — IoT (WiFi only)
config bridge-vlan
    option device 'br-lan'
    option vlan '20'

# LAN interface
config interface 'lan'
    option device 'br-lan.1'
    option proto 'static'
    option ipaddr '192.168.1.1'
    option netmask '255.255.255.0'

# Guest interface
config interface 'guest'
    option device 'br-lan.10'
    option proto 'static'
    option ipaddr '192.168.10.1'
    option netmask '255.255.255.0'

# IoT interface
config interface 'iot'
    option device 'br-lan.20'
    option proto 'static'
    option ipaddr '192.168.20.1'
    option netmask '255.255.255.0'
```

Apply:
```bash
service network restart
```

---

### Step 5 — DHCP for All Networks

Edit `/etc/config/dhcp`. Add guest and IoT pools, and static leases for your devices:

```uci
# LAN (update existing or verify it matches)
config dhcp 'lan'
    option interface 'lan'
    option start '50'
    option limit '150'
    option leasetime '24h'

# Guest — 8h leases so disconnected guests don't linger forever
config dhcp 'guest'
    option interface 'guest'
    option start '50'
    option limit '100'
    option leasetime '8h'

# IoT
config dhcp 'iot'
    option interface 'iot'
    option start '50'
    option limit '50'
    option leasetime '24h'

# Static leases — your devices always get same IP
config host
    option name 'gaming-pc'
    option mac 'aa:bb:cc:dd:ee:ff'    # replace with real MAC
    option ip '192.168.1.10'
    option leasetime 'infinite'

config host
    option name 'your-phone'
    option mac 'aa:bb:cc:dd:ee:ff'    # replace with real MAC
    option ip '192.168.1.11'
    option leasetime 'infinite'

# Xiaomi TV Box S3 — static lease on IoT VLAN
config host
    option name 'xiaomi-tv-box'
    option mac 'aa:bb:cc:dd:ee:ff'    # replace with real MAC (Settings → About → Status on the TV box)
    option ip '192.168.20.10'
    option leasetime 'infinite'
```

How to find MACs:
- PC (Linux): `ip link show` → look for your Ethernet adapter
- Phone: Settings → About Phone → Wi-Fi MAC address
- TV Box: Settings → About → Status → Wi-Fi MAC address

> **⚠️ MAC Randomization — Static leases will silently fail without this fix.**
> Android 10+ and iOS 14+ randomize the MAC address per-network by default. Your phone will get a random IP instead of the static one you configured. Fix this **on each device per-SSID**:
>
> **Android:** Settings → Wi-Fi → long-press `HomeNET` → Modify → Advanced → Privacy → Use device MAC
>
> **iOS:** Settings → Wi-Fi → tap ⓘ next to `HomeNET` → toggle **Private Wi-Fi Address OFF**
>
> Do this for every device that has a static lease. Your gaming PC (wired Ethernet) is not affected — randomization only applies to WiFi.

```bash
service dnsmasq restart
```

---

### Step 6 — Firewall Zones

> **Critical:** `flow_offloading` must be `0` for QoS to work. Hardware offloading bypasses the packet scheduler entirely. This is set here.

Edit `/etc/config/firewall`. Keep your existing `lan` and `wan` zones. Add the guest and IoT zones, and update `defaults`:

```uci
# Update defaults block
config defaults
    option synflood_protect '1'
    option input 'DROP'
    option output 'ACCEPT'
    option forward 'DROP'
    option flow_offloading '0'
    option flow_offloading_hw '0'

# Guest zone — internet only, no LAN access
config zone
    option name 'guest'
    option input 'REJECT'
    option output 'ACCEPT'
    option forward 'REJECT'
    list network 'guest'

config forwarding
    option src 'guest'
    option dest 'wan'

config rule
    option name 'guest-dhcp-dns'
    option src 'guest'
    option dest_port '53 67 68'
    option proto 'tcp udp'
    option target 'ACCEPT'

# IoT zone — internet only, no LAN access
config zone
    option name 'iot'
    option input 'REJECT'
    option output 'ACCEPT'
    option forward 'REJECT'
    list network 'iot'

config forwarding
    option src 'iot'
    option dest 'wan'

config rule
    option name 'iot-dhcp-dns'
    option src 'iot'
    option dest_port '53 67 68'
    option proto 'tcp udp'
    option target 'ACCEPT'

# Force all DNS queries to go through the router
# (prevents bypassing adblock / DoH via hardcoded 8.8.8.8 etc.)
config redirect
    option name 'force-dns-lan'
    option src 'lan'
    option src_dport '53'
    option proto 'tcp udp'
    option target 'DNAT'
    option dest_ip '192.168.1.1'

config redirect
    option name 'force-dns-guest'
    option src 'guest'
    option src_dport '53'
    option proto 'tcp udp'
    option target 'DNAT'
    option dest_ip '192.168.10.1'

# Force IoT DNS through router — Xiaomi TV Box uses hardcoded DNS (8.8.8.8) by default
config redirect
    option name 'force-dns-iot'
    option src 'iot'
    option src_dport '53'
    option proto 'tcp udp'
    option target 'DNAT'
    option dest_ip '192.168.20.1'
```

```bash
service firewall restart
```

---

### Step 7 — Wireless Configuration

Edit `/etc/config/wireless`. The key design:
- `HomeNET` uses the **same SSID on both radios** → enables seamless band steering
- `HomeNET-Family` is a second password on the same LAN VLAN → family uses this
- Guest and IoT are separate SSIDs on separate VLANs

```uci
# ── 2.4 GHz Radio ───────────────────────────────────────────────────────────
config wifi-device 'radio0'
    option type 'mac80211'
    option band '2g'
    option htmode 'HT40'
    option channel 'auto'
    option country 'BD'
    option txpower '20'       # Lower than 5GHz to nudge clients toward 5GHz
    option channel 'auto'     # 2.4GHz auto is fine — no DFS on 2.4GHz
    option disabled '0'

# ── 5 GHz Radio ─────────────────────────────────────────────────────────────
config wifi-device 'radio1'
    option type 'mac80211'
    option band '5g'
    option htmode 'VHT80'
    option channel 'auto'
    option country 'BD'
    option txpower '23'
    option channel '36'            # Use fixed non-DFS channel (36/40/44/48 are safe)
    option disabled '0'

# ── Main SSID — 2.4 GHz ─────────────────────────────────────────────────────
config wifi-iface 'main_24'
    option device 'radio0'
    option mode 'ap'
    option ssid 'HomeNET'
    option encryption 'sae-mixed'        # WPA2 + WPA3 mixed — backward compatible
    option key 'YourStrongMainPassword'
    option network 'lan'
    # 802.11r — Fast roaming between radios
    option ieee80211r '1'
    option ft_psk_generate_local '1'
    option mobility_domain 'ab12'        # 4 hex chars, must match 5GHz below
    option ft_over_ds '1'
    # 802.11k — Clients discover better APs faster
    option ieee80211k '1'
    option rrm_neighbor_report '1'
    option rrm_beacon_report '1'
    # 802.11v — Router can suggest clients roam
    option ieee80211v '1'
    option bss_transition '1'
    option wnm_sleep_mode '1'

# ── Main SSID — 5 GHz ───────────────────────────────────────────────────────
config wifi-iface 'main_5'
    option device 'radio1'
    option mode 'ap'
    option ssid 'HomeNET'                # IDENTICAL to 2.4GHz — required for roaming
    option encryption 'sae-mixed'
    option key 'YourStrongMainPassword'  # IDENTICAL to 2.4GHz
    option network 'lan'
    option ieee80211r '1'
    option ft_psk_generate_local '1'
    option mobility_domain 'ab12'        # MUST match 2.4GHz
    option ft_over_ds '1'
    option ieee80211k '1'
    option rrm_neighbor_report '1'
    option rrm_beacon_report '1'
    option ieee80211v '1'
    option bss_transition '1'
    option wnm_sleep_mode '1'

# ── Family SSID — 2.4 GHz ───────────────────────────────────────────────────
# Same LAN VLAN as main, but separate password you can change without affecting your devices
config wifi-iface 'family_24'
    option device 'radio0'
    option mode 'ap'
    option ssid 'HomeNET-Family'
    option encryption 'psk2+ccmp'
    option key 'FamilyPassword2025'
    option network 'lan'

# ── Family SSID — 5 GHz ─────────────────────────────────────────────────────
config wifi-iface 'family_5'
    option device 'radio1'
    option mode 'ap'
    option ssid 'HomeNET-Family'
    option encryption 'psk2+ccmp'
    option key 'FamilyPassword2025'
    option network 'lan'

# ── Guest SSID — 2.4 GHz only ───────────────────────────────────────────────
config wifi-iface 'guest_24'
    option device 'radio0'
    option mode 'ap'
    option ssid 'HomeNET-Guest'
    option encryption 'psk2+ccmp'
    option key 'GuestPass2025'
    option network 'guest'
    option isolate '1'                   # prevents guest devices from seeing each other

# ── IoT SSID — 2.4 GHz only, hidden ─────────────────────────────────────────
config wifi-iface 'iot_24'
    option device 'radio0'
    option mode 'ap'
    option ssid 'HomeNET-IoT'
    option encryption 'psk2+ccmp'
    option key 'IoTPassword2025'
    option network 'iot'
    option isolate '1'
    option hidden '1'                    # hidden — connect manually on TV box once
```

```bash
wifi
# Wait 10 seconds, then verify all SSIDs are visible on a phone
```

**Password strategy recap:**
- `HomeNET` → Your secret password. Only connect your personal devices. Never share.
- `HomeNET-Family` → Mom, Dad, Sister connect here. You can change this anytime without affecting your own devices.
- `HomeNET-Guest` → Visitors. Isolated from LAN. Router generates QR code for this (Step 12).
- `HomeNET-IoT` → Hidden SSID. Connect Xiaomi TV Box here once manually; it stays connected.

**Connecting the Xiaomi TV Box S3 to HomeNET-IoT:**
On the TV box: Settings → WiFi → Add Network → type `HomeNET-IoT` manually (it's hidden) → enter password → connect. It'll get `192.168.20.10` after you add the static lease in Step 5.

> **Casting from phone to TV box (Google Cast / screen mirror):** Chromecast requires the casting device and TV box to be on the same subnet. Since the TV box is on IoT (192.168.20.x) and your phone is on LAN (192.168.1.x), casting **won't work by default**. You have two options:
>
> **Option A — Just move it to LAN (simpler).** Connect the TV box to `HomeNET-Family` instead, change its static lease to `192.168.1.12`, and casting works immediately. Acceptable risk — it's a TV box, not a door lock. Most people choose this.
>
> **Option B — Keep on IoT + install Avahi mDNS repeater (proper isolation).** Avahi bridges mDNS/Chromecast discovery across subnets:
> ```bash
> opkg install avahi-daemon
> # Edit /etc/avahi/avahi-daemon.conf:
> # allow-interfaces=br-lan.1,br-lan.20
> service avahi-daemon enable && service avahi-daemon start
> ```
> Also add a firewall rule to allow LAN → IoT on port 8009 (Chromecast control port):
> ```bash
> uci add firewall rule
> uci set firewall.@rule[-1].name='cast-to-iot'
> uci set firewall.@rule[-1].src='lan'
> uci set firewall.@rule[-1].dest='iot'
> uci set firewall.@rule[-1].dest_ip='192.168.20.10'
> uci set firewall.@rule[-1].dest_port='8009 8010'
> uci set firewall.@rule[-1].proto='tcp'
> uci set firewall.@rule[-1].target='ACCEPT'
> uci commit firewall && service firewall restart
> ```

---

### Step 8 — DNS-over-HTTPS

All DNS queries from every device will be encrypted via DoH instead of going out in plaintext.

```bash
opkg install https-dns-proxy luci-app-https-dns-proxy
```

```bash
# Remove defaults and configure Cloudflare (primary) + Quad9 (backup)
while uci -q delete https-dns-proxy.@https-dns-proxy[0]; do :; done

uci add https-dns-proxy https-dns-proxy
uci set https-dns-proxy.@https-dns-proxy[-1].bootstrap_dns='1.1.1.1,1.0.0.1'
uci set https-dns-proxy.@https-dns-proxy[-1].resolver_url='https://cloudflare-dns.com/dns-query'
uci set https-dns-proxy.@https-dns-proxy[-1].listen_addr='127.0.0.1'
uci set https-dns-proxy.@https-dns-proxy[-1].listen_port='5053'

uci add https-dns-proxy https-dns-proxy
uci set https-dns-proxy.@https-dns-proxy[-1].bootstrap_dns='9.9.9.9,149.112.112.112'
uci set https-dns-proxy.@https-dns-proxy[-1].resolver_url='https://dns.quad9.net/dns-query'
uci set https-dns-proxy.@https-dns-proxy[-1].listen_addr='127.0.0.1'
uci set https-dns-proxy.@https-dns-proxy[-1].listen_port='5054'

uci commit https-dns-proxy
service https-dns-proxy restart
service dnsmasq restart
```

Test:
```bash
nslookup google.com 127.0.0.1   # should resolve
```

---

### Step 9 — Adblocking

```bash
opkg install adblock-fast luci-app-adblock-fast
```

```bash
uci set adblock-fast.config.enabled='1'
uci set adblock-fast.config.dns='dnsmasq.servers'
uci commit adblock-fast
service adblock-fast enable
service adblock-fast start
```

In LuCI → **Services → Adblock-Fast → Blocklists**, enable:
- `hagezi:multi` — comprehensive, well-maintained
- `oisd:big` — curated, low false positives

If an important site is blocked: LuCI → Adblock-Fast → Allowlist → add the domain.

**Optional — IP-level threat blocking:**
```bash
opkg install banip luci-app-banip
# Enable in LuCI → Services → banIP
# Turn on: "Threats" and "Ads+Trackers" feeds
```

---

### Step 10 — QoS / SQM (Eliminates Gaming Ping Spikes)

```bash
opkg install sqm-scripts luci-app-sqm kmod-sched-cake
```

**Find your WAN interface name:**
```bash
uci get network.wan.device
# Common results: eth0, eth1, pppoe-wan
```

**Configure CAKE SQM** — edit `/etc/config/sqm`:

```uci
config queue 'wan_sqm'
    option enabled '1'
    option interface 'pppoe-wan'     # replace with your WAN interface from above
    option download '63000'          # 90% of 70 Mbps = 63,000 kbps
    option upload '4500'             # 90% of your actual upload — run a speed test first
    option script 'layer_cake.qos'   # DSCP-aware — required for per-device priority
    option qdisc 'cake'
    option linklayer 'ethernet'
    option overhead '40'         # 40 for PPPoE over fiber (not 44 — that's plain Ethernet)
    option qdisc_advanced '0'
```

```bash
uci set sqm.wan_sqm=queue
uci set sqm.wan_sqm.enabled=1
uci set sqm.wan_sqm.interface='pppoe-wan'
uci set sqm.wan_sqm.download=63000
uci set sqm.wan_sqm.upload=4500
uci set sqm.wan_sqm.script='layer_cake.qos'
uci set sqm.wan_sqm.qdisc='cake'
uci set sqm.wan_sqm.linklayer='ethernet'
uci set sqm.wan_sqm.overhead=40
uci commit sqm
service sqm restart
```

**Test:** Go to `https://www.waveform.com/tools/bufferbloat` — target grade **A or A+**. If you get B or lower, reduce download/upload values by another 5%. If A+ but speed feels capped, try increasing values to 95% of your measured speed.

---

### Step 11 — Per-Device Priority & On-Demand Control

```bash
opkg install luci-app-qosmate
service rpcd restart
```

**Configure in LuCI → Network → QoSmate**, or create rules manually.

`layer_cake.qos` maps DSCP values to priority tiers:

| DSCP | Tier | Who |
|------|------|-----|
| CS6 | Highest | Your gaming PC |
| CS4 | High | Your phone |
| CS0 | Default | Family devices |
| CS1 | Bulk | Torrents, large downloads |

**Priority toggle script** — save as `/usr/local/bin/priority`:

```bash
#!/bin/sh
# Usage:
#   priority on          → Your PC gets highest priority (default)
#   priority off         → Equal for everyone (give sister full speed)
#   priority give IP     → Give any device high priority temporarily

ACTION="${1:-on}"
IP="${2:-192.168.1.10}"

case "$ACTION" in
  on)
    nft add rule inet fw4 mangle_prerouting \
      ip saddr 192.168.1.10 ip dscp set cs6 comment "gaming-pc" 2>/dev/null
    nft add rule inet fw4 mangle_prerouting \
      ip saddr 192.168.1.11 ip dscp set cs4 comment "your-phone" 2>/dev/null
    echo "Priority ON — gaming PC = CS6, your phone = CS4"
    ;;
  off)
    nft flush chain inet fw4 mangle_prerouting 2>/dev/null
    echo "Priority OFF — all devices equal bandwidth"
    ;;
  give)
    nft add rule inet fw4 mangle_prerouting \
      ip saddr "$IP" ip dscp set cs4 comment "temp-$IP" 2>/dev/null
    echo "High priority given to $IP (CS4)"
    ;;
  *)
    echo "Usage: priority on | off | give <IP>"
    ;;
esac
```

```bash
chmod +x /usr/local/bin/priority
priority on    # set default
```

**Make priority persist across reboots** — nftables rules added by the script are lost on reboot. Add it to startup:
```bash
# Append to /etc/rc.local (before the 'exit 0' line)
sed -i '/^exit 0/i /usr/local/bin/priority on' /etc/rc.local
# Verify:
cat /etc/rc.local
```

**Daily use:**
```bash
priority on               # your PC gets priority (gaming mode)
priority off              # equal for all (sister needs full speed for something)
priority give 192.168.1.55   # give sister's laptop high priority temporarily
```

---

### Step 12 — Torrent Traffic Deprioritization (BDIX)

Your BDIX torrents hit 10–11 Mbps. Without marking, they compete with game traffic. `layer_cake.qos` already handles fair queuing, but explicitly marking torrents as bulk (CS1) ensures they're always in the lowest tier.

**Two-layer approach: qosify + µTP**

```bash
opkg install qosify
```

Edit `/etc/config/qosify`:

```uci
config interface 'wan'
    option name 'wan'
    option bandwidth_up '4500k'
    option bandwidth_down '63000k'
    option dscp_default_tcp 'CS0'
    option dscp_default_udp 'CS0'
    # Auto-detect sustained bulk flows and mark them CS1
    option dscp_bulk 'CS1'
    option bulk_trigger_pkt_len '500'
    option bulk_trigger_timeout '1'
    option prio_max_avg_pkt_len '400'   # small packets (gaming) stay CS0+

# Mark common BitTorrent ports as bulk — safer than marking by IP
# (marking the whole PC IP as CS1 would also mark your game traffic as bulk!)
config map 'torrent_ports'
    option type 'port'
    option addr '6881-6889'
    option dscp 'CS1'

# If you use a custom port in qBittorrent (recommended — go to Options → Connection → Listening port)
# add it here too:
# config map 'torrent_custom_port'
#     option type 'port'
#     option addr '54321'
#     option dscp 'CS1'
```

```bash
service qosify enable
service qosify start
```

**In qBittorrent → Options → BitTorrent:**
- Enable µTP: ✅ (congestion-aware — backs off automatically when latency rises)
- Set upload limit: 4 Mbps max (keeps headroom for gaming ping)
- Protocol Encryption: Prefer Encrypted ✅

---

### Step 13 — Band Steering (DAWN)

DAWN monitors client signal quality and sends BSS Transition requests to nudge clients to the better radio. The 802.11k/r/v configured in Step 7 is what makes this work.

```bash
opkg install dawn
```

Edit `/etc/config/dawn`:

```uci
config network
    option broadcast_ip '192.168.1.255'
    option broadcast_port '1026'
    option tcp_port '1025'
    option use_broadcast '1'

config metric
    option ht_support '10'
    option vht_support '20'
    option he_support '30'
    option rssi '-70'           # suggest roam if signal drops below -70 dBm
    option rssi_val '-80'       # never kick below this floor
    option min_kick_count '3'   # must trigger 3× before acting (reduces false kicks)

config behaviour
    option kicking '1'          # set to 0 if clients randomly disconnect
    option set_hostapd_nr '1'   # auto-populate neighbor reports
    option eval_probe_req '1'
    option eval_auth_req '1'
    option eval_assoc_req '1'
```

```bash
service dawn enable
service dawn start
```

**If clients randomly disconnect** after enabling DAWN, set `option kicking '0'` — advisory mode still helps 802.11v-capable phones (iPhone, modern Android) roam correctly without forced kicks.

The 2.4 GHz radio is configured at lower TX power (20 dBm vs 23 dBm for 5 GHz) in Step 7 — this naturally makes 5 GHz appear stronger to nearby clients, so they prefer it without DAWN even needing to nudge them.

---

### Step 14 — Guest WiFi QR Code

The router generates a QR code for the Guest SSID. Anyone on your LAN can open it and show it to a guest. The QR only reveals the Guest password — guests land on an isolated VLAN and can't reach your LAN or see any other credentials.

```bash
opkg install qrencode
```

Save this as `/usr/local/bin/gen-guest-qr`:

```bash
#!/bin/sh
SSID=$(uci get wireless.guest_24.ssid)
PASS=$(uci get wireless.guest_24.key)
qrencode -o /www/guest-qr.png -s 8 -m 2 "WIFI:S:${SSID};T:WPA;P:${PASS};;"
echo "QR code updated: http://192.168.1.1/guest-qr.png"
echo "SSID: $SSID  |  Password: $PASS"
```

```bash
chmod +x /usr/local/bin/gen-guest-qr
gen-guest-qr
```

Access from any device on LAN: `http://192.168.1.1/guest-qr.png` — open this, show the screen to your guest, they scan with phone camera.

**Auto-rotate guest password monthly** (so old guests lose access automatically):

```bash
cat > /usr/local/bin/rotate-guest-pass << 'EOF'
#!/bin/sh
NEW=$(cat /dev/urandom | tr -dc 'A-Za-z0-9' | head -c 12)
uci set wireless.guest_24.key="$NEW"
uci commit wireless
wifi
sleep 8
gen-guest-qr
logger "Guest WiFi password rotated"
EOF
chmod +x /usr/local/bin/rotate-guest-pass

# Monthly rotation — 1st of each month at 3am
echo '0 3 1 * * /usr/local/bin/rotate-guest-pass' >> /etc/crontabs/root
service cron restart
```

---

### Step 15 — Backup

After everything is working, **back up immediately before touching anything else.**

LuCI → **System → Backup / Flash Firmware → Generate Archive**

Save `backup-OpenWrt-YYYY-MM-DD.tar.gz` to your PC. This captures all `/etc/config/` files. If you ever need to reset, flash OpenWrt fresh and restore this archive.

---

## Quality of Life Improvements

These are optional but highly recommended. Install any that fit your needs.

---

### Connection Watchdog (Auto-Reconnect)

If your PPPoE connection drops (ISP hiccup), `watchcat` reboots the interface or the whole router automatically.

```bash
opkg install watchcat luci-app-watchcat
```

In LuCI → **Services → Watchcat → Add rule:**
- Mode: `Ping Reboot` or `Restart Interface`
- Ping host: `8.8.8.8`
- Check every: `60` seconds
- Cycles before action: `3`

---

### Bandwidth Usage Graphs (vnStat)

See per-day/month bandwidth usage — useful for keeping an eye on your ISP limit.

```bash
opkg install vnstat2 luci-app-vnstat2
service vnstat enable
service vnstat start
```

View in LuCI → **Status → Traffic** — shows daily/monthly totals per interface.

---

### Dynamic DNS (DDNS)

Gives your home network a stable hostname (like `home.yourdomain.com`) even though your ISP changes your public IP. Required if you want WireGuard remote access.

```bash
opkg install ddns-scripts luci-app-ddns ddns-scripts-cloudflare
```

In LuCI → **Services → Dynamic DNS → Add**:
- Service: Cloudflare (or No-IP if you don't have a domain)
- Hostname: your subdomain
- API Token: Cloudflare token with `Zone:DNS:Edit` permission

---

### WireGuard VPN (Access Home Network Remotely)

Connect to your home network from anywhere — use your gaming PC remotely, access your NAS, or route phone traffic through your home adblock when on mobile data.

```bash
opkg install wireguard-tools kmod-wireguard luci-app-wireguard
```

**Quick setup:**
```bash
# Generate server keys
wg genkey | tee /etc/wireguard/server.key | wg pubkey > /etc/wireguard/server.pub
SERVER_PRIV=$(cat /etc/wireguard/server.key)
SERVER_PUB=$(cat /etc/wireguard/server.pub)

# Generate phone client keys
wg genkey | tee /etc/wireguard/phone.key | wg pubkey > /etc/wireguard/phone.pub
PHONE_PRIV=$(cat /etc/wireguard/phone.key)
PHONE_PUB=$(cat /etc/wireguard/phone.pub)
```

In LuCI → **Network → Interfaces → Add new interface**:
- Name: `wg0`, Protocol: WireGuard VPN
- Private key: paste `$SERVER_PRIV`
- Listen port: `51820`
- IP addresses: `10.0.0.1/24`

Add peer (your phone):
- Public key: paste `$PHONE_PUB`
- Allowed IPs: `10.0.0.2/32`
- Route allowed IPs: checked

In LuCI → **Network → Firewall → Add zone**:
- Name: `vpn`, Network: `wg0`
- Input/Forward: `ACCEPT`, Masquerading: ✅
- Allow forwarding to: `lan`, `wan`

Add firewall rule to accept WireGuard port:
```bash
uci add firewall rule
uci set firewall.@rule[-1].name='wireguard-udp'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].dest_port='51820'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].target='ACCEPT'
uci commit firewall
service firewall restart
```

Client config for your phone (`/etc/wireguard/phone-client.conf`):
```ini
[Interface]
PrivateKey = <PHONE_PRIV key>
Address = 10.0.0.2/32
DNS = 192.168.1.1           # use home adblock + DoH remotely

[Peer]
PublicKey = <SERVER_PUB key>
Endpoint = your-ddns.domain.com:51820
AllowedIPs = 0.0.0.0/0     # route all traffic through home (full tunnel)
PersistentKeepalive = 25
```

Generate QR code for phone:
```bash
qrencode -t ansiutf8 < /etc/wireguard/phone-client.conf
```
Scan with the WireGuard app on your phone.

---

### Wake-on-LAN (Wake Gaming PC Remotely)

Turn on your PC from anywhere via WireGuard.

```bash
opkg install luci-app-wol
```

**On your gaming PC (Ubuntu/Windows):**
- Enable Wake-on-LAN in BIOS/UEFI
- Ubuntu: `sudo ethtool -s enp3s0 wol g` (add to startup)
- Windows: Device Manager → Network Adapter → Properties → Power Management → Allow this device to wake the computer

**In LuCI → Services → Wake on LAN** — select your PC and click Wake. Works from anywhere when connected to WireGuard.

---

### Easy Firmware Updates (Attended Sysupgrade)

Keeps your package list intact when upgrading OpenWrt firmware.

```bash
opkg install luci-app-attendedsysupgrade
```

In LuCI → **System → Attended Sysupgrade** — checks for new OpenWrt versions and rebuilds firmware with all your currently installed packages. No need to manually reinstall everything after update.

---

### Port Forwarding for Online Games

Some games need open NAT. If your ONU is in bridge mode (Step 3), this works perfectly.

LuCI → **Network → Firewall → Port Forwards → Add:**

| Game | Protocol | WAN Port | LAN IP | LAN Port |
|------|----------|----------|--------|----------|
| Steam | TCP/UDP | 27015-27036 | 192.168.1.10 | 27015-27036 |
| Minecraft | TCP | 25565 | 192.168.1.10 | 25565 |

Or via UCI:
```bash
uci add firewall redirect
uci set firewall.@redirect[-1].name='steam-game'
uci set firewall.@redirect[-1].src='wan'
uci set firewall.@redirect[-1].src_dport='27015-27036'
uci set firewall.@redirect[-1].dest='lan'
uci set firewall.@redirect[-1].dest_ip='192.168.1.10'
uci set firewall.@redirect[-1].dest_port='27015-27036'
uci set firewall.@redirect[-1].proto='tcp udp'
uci set firewall.@redirect[-1].target='DNAT'
uci commit firewall
service firewall restart
```

---

### Scheduled Weekly Reboot

Clears memory fragmentation, resets any stuck connections, keeps the router fresh.

```bash
# Reboot every Sunday at 4am (when no one is likely online)
echo '0 4 * * 0 reboot' >> /etc/crontabs/root
service cron restart
```

---

## Troubleshooting Reference

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| WiFi disappears after reboot | MT7915 PCI probe race | Full power cycle (unplug cord), not software reboot |
| Games still lag during downloads | SQM not active | Check `service sqm status`, verify `flow_offloading='0'` in firewall |
| Wrong WAN interface in SQM | Interface name mismatch | Run `uci get network.wan.device` and update sqm config |
| Some websites fail to load | PPPoE MTU issue | Verify MSS clamping is enabled on WAN zone in LuCI |
| Guests can reach LAN devices | Firewall zone error | Verify guest zone has `input 'REJECT'`, forwarding is to wan only |
| Band steering not working | wpad-basic still installed | `opkg list-installed \| grep wpad` — must show `wpad-wolfssl` |
| DAWN causes random disconnects | RSSI threshold too aggressive | Set `option kicking '0'` in DAWN config |
| DoH not working | Service not running | `service https-dns-proxy status`, then `service https-dns-proxy restart` |
| Guest QR not accessible | Web server path issue | `ls /www/guest-qr.png` — regenerate with `gen-guest-qr` |
| Adblock breaking a site | False positive block | Add domain to Adblock-Fast allowlist in LuCI |
| High CPU during heavy traffic | SQM overhead | Normal for CAKE — WR3000's MT7981B handles 70Mbps fine |
| Phone/device not getting static IP | MAC randomization enabled | Disable per-SSID on device: Android → Wi-Fi → Modify → Privacy → Use device MAC |
| Priority rules gone after reboot | nftables rules not persistent | Verify `/etc/rc.local` has `priority on` line |
| 5GHz WiFi takes 60s to start | DFS radar scan on channel | Set `option channel '36'` in radio1 config (Step 7) |
| IoT device (TV Box) showing ads | Hardcoded DNS bypassing adblock | Verify `force-dns-iot` rule exists in firewall (Step 6) |
| PPPoE session hangs without disconnect | No LCP keepalive | Add `option keepalive '4 20'` to WAN interface (Step 3) |
| Adblock not working on guest/IoT | Misconception | Adblock works via dnsmasq which serves ALL zones — it applies everywhere |
| WireGuard not reachable from internet | Double-NAT (ONU not in bridge mode) | Enable bridge mode on ONU or set port forward for UDP 51820 on ONU admin page |
| opkg not found after firmware upgrade | Future OpenWrt uses apk | OpenWrt 25.x+ transitions to `apk` package manager — same commands, different binary |

---

## File Map

```
/etc/config/
├── network           → VLANs, interfaces, IP addresses, PPPoE WAN
├── wireless          → SSIDs, passwords, VLAN binding, 802.11k/r/v
├── firewall          → Zones, forwarding rules, DNS redirect, port forwards
├── dhcp              → DHCP pools, static leases
├── sqm               → CAKE QoS settings
├── https-dns-proxy   → DoH resolvers
├── adblock-fast      → Blocklists config
├── dawn              → Band steering thresholds
├── qosify            → DSCP/bulk traffic marking
├── wireguard/        → WireGuard keys and peer configs
└── ddns              → Dynamic DNS config

/usr/local/bin/
├── priority          → Toggle device priority on/off/give
├── gen-guest-qr      → Generate guest WiFi QR code
└── rotate-guest-pass → Rotate guest password + regenerate QR

/www/
└── guest-qr.png      → Guest QR code (http://192.168.1.1/guest-qr.png)
```

---

## Daily Cheatsheet

```bash
# Priority modes
priority on                      # gaming mode — your PC highest priority
priority off                     # equal bandwidth for all
priority give 192.168.1.55       # give a specific device high priority

# Guest management
gen-guest-qr                     # refresh QR after password change
rotate-guest-pass                # change guest password + update QR
# Show QR: open http://192.168.1.1/guest-qr.png on any LAN device

# Monitoring
cat /tmp/dhcp.leases             # see all connected devices + IPs
service adblock-fast status      # adblock running?
service sqm status               # QoS running?
service dawn status              # band steering running?

# After any config change
service network restart          # network + VLANs
service wireless restart         # WiFi (or just: wifi)
service firewall restart         # firewall rules
service dnsmasq restart          # DNS + DHCP

# Adblock
service adblock-fast reload      # force blocklist update
# Add a site to allowlist: LuCI → Services → Adblock-Fast → Allowlist

# Bufferbloat test
# → https://www.waveform.com/tools/bufferbloat (target A or A+)

# Check WireGuard peers
wg show                          # see connected VPN clients
```

---

*Cudy WR3000 v1 · OpenWrt 24.10.x · Exord Online · PPPoE via ONU*
*Stack: DSA VLANs · CAKE SQM · layer_cake QoS · DAWN band steering · DoH · adblock-fast · WireGuard*
