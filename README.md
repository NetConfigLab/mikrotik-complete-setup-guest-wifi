# MikroTik Complete Setup & Secure Isolated Guest Wi-Fi (RouterOS v6)

A production-ready base network template and secure isolated Guest Wi-Fi implementation for MikroTik RouterOS v6 devices (tested on physical hardware: MikroTik hAP lite RB941-2nD).

This repository documents the complete configuration workflow demonstrated across our hands-on YouTube video series:
- Part 1 (Base RouterOS Setup from Scratch): https://youtu.be/4zbpMH4VU0U
- Part 2 (Enterprise Wi-Fi & Isolated Guest Network): https://youtu.be/KsGy6qdhLZI

---

## Architecture Overview

```text
WAN (ISP Internet)
       │
[ether1 (DHCP Client)]
       │
┌──────┴───────────────────────────┐
│       MikroTik RouterOS v6       │
│        Identity: NetComLab       │
└──────┬───────────────────────────┘
       │
       ├─────────────────────────────────────────┐
       │                                         │
[bridge-local]                            [bridge-guest]
Subnet: 192.168.10.1/24                   Subnet: 192.168.20.1/24
- Ports: ether2, ether3, ether4           - Interface: wlan-guest (Virtual AP)
- Wireless: wlan1 (NetConfigLab-CORP)     - SSID: NetConfigLab-Guest
- DHCP: 192.168.10.2 - 192.168.10.254     - DHCP: 192.168.20.2 - 192.168.20.254
- Access: Full Local & Internet           - Security: L2 Client Isolation (default-forward=no)
- Firewall: Drop to LAN & Drop to Admin Ports

```
## Key Features


1.Clean Baseline Configuration: Built from scratch with no default configuration.
2.WAN & Masquerade NAT: Automatic IP pickup via DHCP on ether1 with outbound NAT Masquerade.
3.Core Local Network (Subnet 10): Switched wired ports (ether2-ether4) and primary secure Wi-Fi AP.
4.Isolated Guest Network (Subnet 20): Virtual AP (VAP) tied to an independent bridge.
5.Layer 2 Client Isolation: default-forward=no prevents wireless guests from communicating with one another.
6.Layer 3 Firewall Containment: Forward and Input filter rules block guest traffic to the corporate network and management ports (WinBox 8291, WebFig 80/443, SSH 22).


## Full RouterOS v6 CLI Script
Run the following commands in the MikroTik Terminal (WinBox -> Terminal or SSH):

```routeros

# ============================================================
# 1. SYSTEM IDENTITY & ROUTER NAMING
# ============================================================
/system identity set name="NetComLab"

# ============================================================
# 2. WAN CONFIGURATION (ISP DHCP CLIENT)
# ============================================================
/ip dhcp-client add interface=ether1 disabled=no comment="WAN Internet Uplink"

# ============================================================
# 3. CORE LAN BRIDGE & PHYSICAL PORT BINDING
# ============================================================
/interface bridge add name=bridge-local comment="Core LAN Bridge"
/interface bridge port
add bridge=bridge-local interface=ether2
add bridge=bridge-local interface=ether3
add bridge=bridge-local interface=ether4

# ============================================================
# 4. CORE LAN IP ADDRESSING & DHCP SERVER (192.168.10.0/24)
# ============================================================
/ip address add address=192.168.10.1/24 interface=bridge-local network=192.168.10.0
/ip pool add name=dhcp_pool_lan ranges=192.168.10.2-192.168.10.254
/ip dhcp-server add address-pool=dhcp_pool_lan interface=bridge-local lease-time=1d name=dhcp-lan disabled=no
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.10.1,8.8.8.8,1.1.1.1

# ============================================================
# 5. WIRELESS SECURITY PROFILES (WPA2-PSK AES-CCM)
# ============================================================
/interface wireless security-profiles
add name=Core-Secure-Prof mode=dynamic-keys authentication-types=wpa2-psk unicast-ciphers=aes-ccm group-ciphers=aes-ccm wpa2-pre-shared-key="CHANGE_TO_SECURE_LAN_PASS"
add name=Prof-Guest mode=dynamic-keys authentication-types=wpa2-psk unicast-ciphers=aes-ccm group-ciphers=aes-ccm wpa2-pre-shared-key="CHANGE_TO_SECURE_GUEST_PASS"

# ============================================================
# 6. PRIMARY WIRELESS AP CONFIGURATION
# ============================================================
/interface wireless
set [ find default-name=wlan1 ] ssid="NetConfigLab-CORP" mode=ap-bridge band=2ghz-b/g/n channel-width=20mhz frequency=auto wireless-protocol=802.11 security-profile=Core-Secure-Prof disabled=no
/interface bridge port add bridge=bridge-local interface=wlan1

# ============================================================
# 7. VIRTUAL AP (GUEST WIFI) WITH LAYER 2 ISOLATION
# ============================================================
/interface wireless
add name=wlan-guest master-interface=wlan1 ssid="NetConfigLab-Guest" security-profile=Prof-Guest default-forward=no disabled=no

# Dedicated Guest Bridge
/interface bridge add name=bridge-guest comment="Isolated Guest Bridge"
/interface bridge port add bridge=bridge-guest interface=wlan-guest

# ============================================================
# 8. GUEST IP ADDRESSING & DHCP SERVER (192.168.20.0/24)
# ============================================================
/ip address add address=192.168.20.1/24 interface=bridge-guest network=192.168.20.0
/ip pool add name=dhcp_pool_guest ranges=192.168.20.2-192.168.20.254
/ip dhcp-server add address-pool=dhcp_pool_guest interface=bridge-guest lease-time=1h name=dhcp-guest disabled=no
/ip dhcp-server network add address=192.168.20.0/24 gateway=192.168.20.1 dns-server=192.168.20.1,8.8.8.8,1.1.1.1

# Allow DNS requests on router for local resolver
/ip dns set allow-remote-requests=yes

# ============================================================
# 9. FIREWALL NAT (INTERNET ACCESS)
# ============================================================
/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade comment="Default Masquerade NAT"

# ============================================================
# 10. FIREWALL FILTER RULES (GUEST ISOLATION & HARDENING)
# ============================================================
/ip firewall filter
# Drop guest access to core LAN resources
add chain=forward in-interface=bridge-guest out-interface=bridge-local action=drop comment="Block Guest to LAN"

# Drop guest access to router management services
add chain=input in-interface=bridge-guest protocol=tcp dst-port=21,22,23,80,443,8291,8728,8729 action=drop comment="Block Guest to Router Management"

```


## Verification & Testing Checklist
- [x] WAN Check: Router acquires an IP address on ether1 and can ping external hosts (8.8.8.8).
- [x] Core LAN Check: Wired devices on ether2-ether4 and wireless clients on NetConfigLab-CORP receive 192.168.10.x IPs with full internet access.
- [x] Guest Isolation (L2): Connected devices on NetConfigLab-Guest cannot discover or reach each other.
- [x] Guest Isolation (L3): Devices on 192.168.20.x cannot ping 192.168.10.x and cannot access WinBox/WebFig at 192.168.20.1.

## Support & Community
Maintained by NetConfig Lab.

- **YouTube:** https://www.youtube.com/@NetConfigLab
- **Reddit:** https://reddit.com/u/NetConfigLab
