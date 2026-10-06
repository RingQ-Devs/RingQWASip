# WhatsApp SIP Gateway — Complete Technical Reference
**RingQ PBX + Meta WhatsApp Business Calling**
*Production Reference — graylunaplc.ringq.ai*

---

## Table of Contents
1. [How It All Works — Big Picture](#1-how-it-all-works--big-picture)
2. [Architecture](#2-architecture)
3. [Ports & Transport](#3-ports--transport)
4. [Configuration A — wasip.ringq.io Portal](#4-configuration-a--wasipringqio-portal)
5. [Configuration B — /wasip Admin UI on RingQ PBX](#5-configuration-b--wasip-admin-ui-on-ringq-pbx)
6. [Inbound Call Flow](#6-inbound-call-flow)
7. [Outbound Call Flow](#7-outbound-call-flow)
8. [Dialplan Configuration](#8-dialplan-configuration)
9. [Database Tables](#9-database-tables)
10. [ACL — IP Whitelisting](#10-acl--ip-whitelisting)
11. [SIP Gateway (watrunk)](#11-sip-gateway-watrunk)
12. [SDES / SRTP — Audio Encryption](#12-sdes--srtp--audio-encryption)
13. [Troubleshooting Guide](#13-troubleshooting-guide)
14. [Key Lessons Learned](#14-key-lessons-learned)
15. [Fresh PBX Deployment Checklist](#15-fresh-pbx-deployment-checklist)
16. [Complete API Reference](#16-complete-api-reference)

---

## 1. How It All Works — Big Picture

WhatsApp Business Calling uses **SIP over TLS** to bridge between Meta's infrastructure and your RingQ PBX. There are three sides to this system:

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                       │
│  CUSTOMER SIDE          META SIDE             RingQ SIDE              │
│                                                                       │
│  ┌──────────┐          ┌──────────┐          ┌──────────────────┐   │
│  │WhatsApp  │◄────────►│wa.meta.vc│◄──TLS───►│  RingQ PBX       │   │
│  │App       │ WhatsApp │  :5061   │  SIP     │  (graylunaplc)   │   │
│  └──────────┘          └──────────┘          └────────┬─────────┘   │
│                              ▲                        │             │
│                              │                        │             │
│                    ┌─────────┴────────┐      ┌────────▼─────────┐  │
│                    │ Meta Graph API   │      │  Agent Softphone  │  │
│                    │ graph.facebook   │      │  (Browser/WSS)    │  │
│                    │ .com/v21.0/      │      │  Extension 1021   │  │
│                    └─────────▲────────┘      └──────────────────┘  │
│                              │                                       │
│                    ┌─────────┴────────┐                             │
│                    │ wasip.ringq.io   │                             │
│                    │ API Management   │                             │
│                    │ (.env credentials│                             │
│                    └──────────────────┘                             │
└─────────────────────────────────────────────────────────────────────┘
```

### The Three Connections

**Connection 1: Customer ↔ Meta**
The customer uses the WhatsApp app on their phone. Meta handles the WhatsApp protocol entirely. Your RingQ PBX never interacts with WhatsApp directly — only with Meta's SIP gateway.

**Connection 2: Meta ↔ RingQ PBX**
Meta's SIP gateway (`wa.meta.vc:5061`) and your RingQ PBX talk standard SIP over TLS. This is how calls are actually delivered. Meta sends `INVITE` to your PBX IP on port 5061.

**Connection 3: wasip.ringq.io ↔ Meta Graph API**
A separate management server holds your Meta credentials and sends REST API calls to configure Meta's settings (which PBX to route calls to, encryption type, etc.).

### What Meta Stores (Only 3 Values)
```json
{
  "hostname": "139.59.98.108",   ← Your RingQ PBX IP
  "port": 5061,
  "srtp_key_exchange_protocol": "SDES"
}
```
Meta sends ALL calls for your WA number to this IP:port. That's it. Meta knows nothing about extensions, queues, or agents.

### What RingQ PBX Controls
Everything after the call arrives at your PBX is handled internally:
- Which extension/queue/IVR to ring
- Caller ID presentation
- Hold music
- Recording
- Queue routing

Changing routing **never requires a Meta API call** — just update the RingQ dialplan.

### No Gateway Registration Required
The watrunk SIP gateway has `register=false`. RingQ PBX does NOT register with Meta.
- **Inbound**: Meta pushes calls directly to your IP (no registration needed)
- **Outbound**: Uses per-call inline SIP credentials (no registration needed)

---

## 2. Architecture

### Components

| Component | Server | Purpose |
|---|---|---|
| **RingQ PBX** | graylunaplc.ringq.ai (139.59.98.108) | SIP engine, call routing, agent phones |
| **RingQ PHP** | /var/www/ringq on PBX | Dialplan XML provider (mod_xml_curl) |
| **wasip Admin UI** | /var/www/wasip on PBX | Self-hosted admin portal for gateway |
| **wa.meta.vc** | Meta infrastructure | Meta's SIP gateway |
| **wasip.ringq.io** | Separate management server | Holds Meta API credentials (.env) |
| **PostgreSQL** | 127.0.0.1:5432 on PBX | All RingQ + gateway configuration |

### How mod_xml_curl Works (Important)
RingQ PBX **does not store dialplans in memory**. Every time a call arrives:
1. RingQ PBX calls RingQ PHP: `GET /index.php?section=dialplan&context=public...`
2. RingQ PHP queries PostgreSQL (`v_dialplans`, `v_dialplan_details`)
3. Returns XML dialplan on-the-fly
4. RingQ PBX executes it

This is why `show dialplan` always returns blank — correct, expected behavior.

---

## 3. Ports & Transport

### RingQ PBX Listening Ports

| Port | Protocol | Profile | Purpose |
|---|---|---|---|
| **5061** | **TLS** | **internal** | **Meta WhatsApp SIP — inbound calls** |
| 5060 | UDP/TCP | internal | Internal SIP (extensions) |
| 7443 | WSS | internal | Browser softphone (WebRTC) |
| 5080 | UDP/TCP | external | External SIP trunks |
| 5081 | TLS | external | External TLS trunks |

### Firewall Rules Required

```bash
# INBOUND — Allow Meta IPs on SIP + RTP
# (11 Meta CIDR ranges — see Section 10)
iptables -A INPUT -s 157.240.0.0/16 -p tcp --dport 5061 -j ACCEPT
iptables -A INPUT -s 66.220.144.0/20 -p tcp --dport 5061 -j ACCEPT
# ... all 11 ranges

# RTP media (audio packets)
iptables -A INPUT -p udp --dport 16384:32768 -j ACCEPT

# OUTBOUND — RingQ PBX to Meta (for outbound calls)
iptables -A OUTPUT -d wa.meta.vc -p tcp --dport 5061 -j ACCEPT
```

### TLS Certificate
```
Location: /etc/freeswitch/tls/agent.pem
Required: Valid for your PBX hostname
Check:    openssl x509 -in /etc/freeswitch/tls/agent.pem -noout -subject -dates
```
Meta accepts self-signed certificates. No CA validation required from Meta's side.

---

## 4. Configuration A — wasip.ringq.io Portal

`wasip.ringq.io` is a **separate management server** (not the RingQ PBX). It holds your Meta API credentials in a `.env` file and is used to make direct Meta Graph API calls.

### When to Use wasip.ringq.io
- Initial Meta account setup
- Emergency credential changes
- SDES re-application (if audio breaks)
- Verifying current Meta settings
- Getting/checking the Meta SIP password

### Credentials File Location
```bash
# On wasip.ringq.io
cat /root/wasip-gateway-production/.env

# Required variables:
WHATSAPP_PHONE_NUMBER_ID=1214061478463577
WHATSAPP_ACCESS_TOKEN=EAAxxxxxxxxxxxxxxx
WHATSAPP_APP_SECRET=xxxxxxxxxxxxxxxx
```

### Step-by-Step Meta Configuration

```bash
# SSH into wasip.ringq.io, then:
cd /root/wasip-gateway-production && source .env

# ─── STEP 1: Enable WhatsApp Calling ─────────────────────────────
curl -s -X POST "https://graph.facebook.com/v21.0/$WHATSAPP_PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $WHATSAPP_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"calling":{"status":"ENABLED","call_icon_visibility":"DEFAULT"}}' \
  | python3 -m json.tool
# Expected: {"success": true}

# ─── STEP 2: Enable SIP → point to your RingQ PBX ────────────────
# IMPORTANT: Use IP address, NOT hostname (see Section 14, Lesson 2)
curl -s -X POST "https://graph.facebook.com/v21.0/$WHATSAPP_PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $WHATSAPP_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"calling":{"status":"ENABLED","sip":{"status":"ENABLED","servers":[{"hostname":"139.59.98.108","port":5061}]}}}' \
  | python3 -m json.tool
# Expected: {"success": true}

# ─── STEP 3: Set SDES — MUST BE LAST, SEPARATE CALL ─────────────
# If combined with Step 2, it resets to DTLS → no audio
curl -s -X POST "https://graph.facebook.com/v21.0/$WHATSAPP_PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $WHATSAPP_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"calling":{"srtp_key_exchange_protocol":"SDES"}}' \
  | python3 -m json.tool
# Expected: {"success": true}
```

### Verify Current Meta Settings
```bash
cd /root/wasip-gateway-production && source .env

curl -s "https://graph.facebook.com/v21.0/$WHATSAPP_PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $WHATSAPP_ACCESS_TOKEN" | python3 -m json.tool
```
Expected output:
```json
{
  "calling": {
    "status": "ENABLED",
    "srtp_key_exchange_protocol": "SDES",
    "sip": {
      "status": "ENABLED",
      "servers": [{"hostname": "139.59.98.108", "port": 5061}]
    }
  }
}
```

### Get Meta SIP Password
```bash
curl -s "https://graph.facebook.com/v21.0/$WHATSAPP_PHONE_NUMBER_ID/settings?include_sip_credentials=true" \
  -H "Authorization: Bearer $WHATSAPP_ACCESS_TOKEN" | python3 -m json.tool | grep sip_user_password
```

### Change PBX IP (When Migrating Servers)
```bash
# Just re-run Step 2 with new IP, then Step 3 for SDES
curl -s -X POST "https://graph.facebook.com/v21.0/$WHATSAPP_PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $WHATSAPP_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"calling":{"status":"ENABLED","sip":{"status":"ENABLED","servers":[{"hostname":"NEW_IP","port":5061}]}}}' \
  | python3 -m json.tool

sleep 5

curl -s -X POST "https://graph.facebook.com/v21.0/$WHATSAPP_PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $WHATSAPP_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"calling":{"srtp_key_exchange_protocol":"SDES"}}' | python3 -m json.tool
```

### Disable WhatsApp Calling (Emergency)
```bash
curl -s -X POST "https://graph.facebook.com/v21.0/$WHATSAPP_PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $WHATSAPP_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"calling":{"sip":{"status":"DISABLED"}}}' | python3 -m json.tool
```

---

## 5. Configuration B — /wasip Admin UI on RingQ PBX

The `/wasip` portal is installed on the RingQ PBX itself. It manages the RingQ-side configuration: DB entries, dialplans, ACL, gateway, and reloads.

### Access
```
URL:    https://graylunaplc.ringq.ai/wasip?token=YOUR_TOKEN
Token:  SELECT access_token FROM q_tokens ORDER BY insert_date DESC LIMIT 1;
```

### Files on PBX
```
/var/www/wasip/index.php     — Application (PHP 7.4 + Phalcon)
/etc/ringq/config.conf       — DB credentials (read automatically, no hardcoding)
/etc/nginx/ringq             — Add location block here
```

### Nginx Setup (add before `location ~ \.php$`)
```nginx
location ^~ /wasip {
    fastcgi_pass  unix:/var/run/php/php7.4-fpm.sock;
    fastcgi_index index.php;
    include       fastcgi_params;
    fastcgi_param SCRIPT_FILENAME /var/www/wasip/index.php;
    fastcgi_param REQUEST_URI     $request_uri;
    fastcgi_read_timeout 3m;
}
```
After editing: `nginx -t && systemctl reload nginx`

### Health Check (no token needed)
```
https://graylunaplc.ringq.ai/wasip/health
```

---

### Overview Tab
Displays current gateway state (real-time from DB):

| Card | Source | Notes |
|---|---|---|
| Status badge | wa_gateway_config.gateway_status | Active / Deactivated / Failed |
| WA Number | wa_gateway_config.wa_number | Hidden when deactivated |
| Extension | v_dialplan_details (live) | Real-time — reflects RingQ DID changes immediately |
| Gateway in DB | v_gateways WHERE gateway='watrunk' | Checks actual DB |
| Dialplan in DB | v_dialplans WHERE dialplan_number=waNumber | Checks actual DB |

---

### Configure Tab
For **first-time setup** or **re-running configuration**.

**Fields:**
- **Access Token** — Meta permanent System User token (saved encrypted)
- **App Secret** — From Meta App Settings → Basic → Show
- **Phone Number ID** — Long numeric ID from WhatsApp API Setup (NOT the phone number)
- **Business Account ID** — WABA ID (optional)
- **WA Business Number** — Digits only (e.g. 6567729965)
- **PBX Domain** — Auto from browser URL. **Enter your PBX IP** (e.g. 139.59.98.108) not hostname
- **Route to Extension** — Dropdown showing all Extensions / Queues / IVR from DB

**What Configure Gateway does (13 steps):**
1. Validates all inputs
2. Saves credentials AES-256 encrypted to `wa_gateway_config`
3. Connects to Meta API (verifies phone number)
4. Calls Meta API: Enable calling
5. Calls Meta API: Enable SIP → this PBX
6. Calls Meta API: Set SDES (separate, last call)
7. Gets Meta SIP password
8. DELETEs all existing watrunk rows + INSERTs fresh watrunk in `v_gateways`
9. Adds Meta IP ranges to providers ACL
10. Updates SIP profile settings (OPUS codec + SRTP)
11. Creates entry in `q_dids`
12. Creates DID dialplan in public context (inline XML)
13. Reloads RingQ PBX (clears cache + reloadxml + reloadacl + profile rescan)

When already active, button shows "↺ Re-run Configuration" (secondary style) with active banner.

---

### Manage Tab
Shown only when gateway is **active** or **configured**.

**Reconfigure / Change PBX URL**
Use when:
- Moving to a new server
- Changing the target extension/queue/IVR
- PBX IP has changed

What it does:
1. Calls Meta API: Update SIP hostname to current browser URL/IP
2. Calls Meta API: Re-apply SDES
3. Gets new SIP password from Meta
4. Updates watrunk password in `v_gateways`
5. Rebuilds DID dialplan with new extension
6. Reloads RingQ PBX

**Deactivate Gateway**
Completely removes WhatsApp calling. What it does:
1. Calls Meta API: Disable SIP
2. Deletes watrunk from `v_gateways`
3. Deletes entry from `q_dids` (by WA number)
4. Deletes DID dialplan from `v_dialplans` (by WA number in public context)
5. Deletes dialplan details from `v_dialplan_details`
6. Removes Meta IP ranges from ACL
7. Updates `wa_gateway_config.gateway_status = 'deactivated'`
8. Reloads RingQ PBX
9. Page auto-refreshes after 2.5 seconds

Credentials remain encrypted in `wa_gateway_config` for easy reconfiguration later.

---

### Real-time Extension Display
The portal reads the routing target directly from `v_dialplan_details` — not from `wa_gateway_config`. If someone changes the DID routing in RingQ DID Manager, the portal shows the update automatically on next load and syncs `wa_gateway_config.extension_number`.

---

## 6. Inbound Call Flow

```
Step 1: Customer opens WhatsApp, taps call on your business number +6567729965
             ↓
Step 2: Meta SIP Gateway (wa.meta.vc) processes the call
             ↓
Step 3: Meta sends SIP INVITE to 139.59.98.108:5061
        ┌──────────────────────────────────────────────────┐
        │ INVITE sip:+6567729965@139.59.98.108:5061 SIP/2.0│
        │ From: "Rajthilak" <sip:+917397574188@wa.meta.vc> │
        │ To: <sip:+6567729965@139.59.98.108:5061>         │
        │ X-FB-External-Domain: wa.meta.vc                 │
        │ x-wa-meta-wacid: wacid.xxxxx                     │
        │ Content-Type: application/sdp                    │
        │ (SDP with a=crypto:1 AES_CM_128_HMAC_SHA1_80...) │
        └──────────────────────────────────────────────────┘
             ↓
Step 4: RingQ PBX internal profile receives on port 5061 (TLS)
             ↓
Step 5: ACL check — Meta IP (e.g. 66.220.149.16) checked against providers ACL
        → ALLOWED (Meta IPs in whitelist)
             ↓
Step 6: Dialplan lookup (mod_xml_curl)
        RingQ PHP queries: public context, destination_number = +6567729965
        Returns DID dialplan for 6567729965
             ↓
Step 7: Dialplan executes:
        set effective_caller_id_number = ${sip_from_user}  → +917397574188
        set effective_caller_id_name   = ${sip_from_display} → Rajthilak
        export call_direction = inbound
        set domain_uuid = ffd1d1d2-... (inline)
        set domain_name = graylunaplc.ringq.ai (inline)
        transfer 1021 XML graylunaplc.ringq.ai
             ↓
Step 8: Extension 1021 dialplan — rings agent's browser softphone via WSS
             ↓
Step 9: Agent answers → SDES SRTP negotiation → two-way OPUS audio
```

### What the SDP Looks Like (SDES = Working)
```
Meta offer SDP:
  m=audio 3480 UDP/TLS/RTP/SAVPF 111 126
  a=crypto:1 AES_CM_128_HMAC_SHA1_80 inline:Sy8ADAtyZ+...  ← SDES key

RingQ PBX answer SDP:
  m=audio 25972 UDP/TLS/RTP/SAVPF 111 126
  a=crypto:1 AES_CM_128_HMAC_SHA1_80 inline:responseKey...  ← SDES response
```

### What the SDP Looks Like (DTLS = Broken, no audio)
```
  a=fingerprint:sha-256 BB:12:82:66:...  ← DTLS (wrong)
  a=setup:actpass                         ← Both sides try to be client → conflict
```

---

## 7. Outbound Call Flow

```
Step 1: Agent dials 917397574188 from RingQ browser softphone
             ↓
Step 2: RingQ PBX processes call in context graylunaplc.ringq.ai
             ↓
Step 3: Dialplan order processing:
        Order 10:  global user_exists check → ${user_exists}=false (external number)
        Order 82:  outbound-pin → no match (different prefix patterns)
        Order 83:  WhatsApp Outbound → MATCH! ^(\+?[1-9][0-9]{8,14})$
             ↓
Step 4: WhatsApp Outbound sets per-call channel variables:
        sip_auth_username         = 6567729965
        sip_auth_password         = pEwPqO3d92WC...
        sip_from_user             = 6567729965
        sip_from_domain           = graylunaplc.ringq.ai
        origination_caller_id_number = 6567729965
        effective_caller_id_number = 6567729965
        rtp_secure_media          = mandatory
        sip_allow_reinvite        = false
             ↓
Step 5: Bridge: sofia/internal/917397574188@wa.meta.vc:5061;transport=tls
        ┌────────────────────────────────────────────────────────────┐
        │ INVITE sip:917397574188@wa.meta.vc:5061 SIP/2.0           │
        │ From: sip:6567729965@graylunaplc.ringq.ai  ← WA business  │
        │ To: sip:917397574188@wa.meta.vc                           │
        │ Authorization: Digest username="6567729965", realm="..."   │
        │ (SDP with a=crypto:1 SDES key)                           │
        └────────────────────────────────────────────────────────────┘
             ↓
Step 6: Meta validates credentials → rings +917397574188 on WhatsApp
        Customer sees: incoming WhatsApp call from +6567729965
             ↓
Step 7: Customer answers → two-way OPUS audio
```

### Direct Test (SSH/Putty)
```bash
fs_cli -x "originate {\
sip_auth_username=6567729965,\
sip_auth_password=pEwPqO3d92WCvcaI7FzmpOynBd0A97k3,\
sip_from_domain=graylunaplc.ringq.ai,\
origination_caller_id_number=6567729965,\
sip_from_user=6567729965,\
rtp_secure_media=mandatory}\
sofia/internal/917397574188@wa.meta.vc:5061;transport=tls 1021"
```

### Key: Inline Auth (No Gateway Registration)
Outbound works because RingQ PBX sends SIP credentials **per-call** in the INVITE's Authorization header. Meta validates them against the stored SIP password. No persistent gateway registration needed.

---

## 8. Dialplan Configuration

### Dialplan Order (context: graylunaplc.ringq.ai)

```
Order  0-10: Global (timezone, hold music, call-id export, user_exists check)
Order 82:    outbound-pin (specific authorized prefixes)
Order 83:    WhatsApp Outbound ← MUST BE HERE (before prevent-outbound)
Order 84:    outbound-rules (combined external gateway rules)
Order 87:    WARule (DISABLED — was routing via watrunk gateway)
Order 88:    prevent-outbound ← blocks external calls for unregistered users
Order 100+:  Queue, IVR, extension routing
Order 9999:  unknown-destination → "please contact administrator"
```

### Why Order 83 is Non-Negotiable
`prevent-outbound` (order 88) blocks any external number when `${user_exists}=false`.
External numbers (like WhatsApp customer numbers) always have `user_exists=false`.
WhatsApp Outbound at order 83 intercepts BEFORE the block fires.

### Inbound DID Dialplan (public context)
```sql
-- In v_dialplans:
dialplan_name    = '6567729965'    -- the WA business number
dialplan_context = 'public'
dialplan_order   = 0
app_uuid         = extension_uuid  -- UUID of the target extension

-- In v_dialplan_details (inline XML actions):
condition: destination_number = ^((\+)?6567729965)$
action:    set effective_caller_id_number=${sip_from_user}
action:    set effective_caller_id_name=${sip_from_display}
action:    export call_direction=inbound inline=true
action:    set domain_uuid=ffd1d1d2-... inline=true
action:    set domain_name=graylunaplc.ringq.ai inline=true
action:    export hold_music=local_stream://default inline=true
action:    transfer 1021 XML graylunaplc.ringq.ai
```

### Outbound Dialplan (domain context)
```sql
-- In v_dialplans:
dialplan_name    = 'WhatsApp Outbound'
dialplan_context = 'graylunaplc.ringq.ai'
dialplan_order   = 83   ← critical

-- In v_dialplan_details:
condition: destination_number = ^(\+?[1-9][0-9]{8,14})$
action:    set effective_caller_id_number=6567729965
action:    set sip_auth_username=6567729965
action:    set sip_auth_password=pEwPqO3d92WC...
action:    set sip_from_user=6567729965
action:    set sip_from_domain=graylunaplc.ringq.ai
action:    set origination_caller_id_number=6567729965
action:    set rtp_secure_media=mandatory
action:    set sip_allow_reinvite=false
action:    bridge sofia/internal/$1@wa.meta.vc:5061;transport=tls
```

### Reload After Changes
```bash
rm -rf /var/cache/ringq/*
fs_cli -x "reloadxml"
```

---

## 9. Database Tables

### wa_gateway_config (Custom)
Tracks portal configuration state. Created by `/wasip` portal.

| Column | Purpose |
|---|---|
| wa_number | WhatsApp business number (primary key logic) |
| phone_number_id | Meta Phone Number ID |
| pbx_domain | PBX hostname/IP |
| extension_number | Current routing target (auto-synced from dialplan) |
| dialplan_uuid | UUID of inbound DID dialplan |
| access_token_enc | Meta token (AES-256 encrypted) |
| meta_sip_pass_enc | Meta SIP password (AES-256 encrypted) |
| gateway_status | unconfigured / configured / active / deactivated / failed |

### v_gateways (watrunk)
SIP trunk for Meta WhatsApp.

| Column | Value |
|---|---|
| gateway | watrunk |
| username / auth_username | 6567729965 (WA business number) |
| password | Meta SIP password |
| realm | wa.meta.vc:5061 |
| proxy / register_proxy | wa.meta.vc:5061 |
| register | **false** (no registration) |
| register_transport | tls |
| profile | **internal** (has port 5061) |
| context | graylunaplc.ringq.ai |
| codec_prefs | OPUS |

### v_dialplans + v_dialplan_details
See Section 8. Dialplans are read dynamically by mod_xml_curl on every call.

### q_dids
RingQ DID management.

| Column | Value |
|---|---|
| did_number | 6567729965 |
| type | 3 (SIP gateway DID) |
| name | Target extension name |
| extension | 1021 |
| sip_trunk_name | watrunk |

### v_sip_profile_settings
Applied to both `internal` and `external` profiles.

| Setting | Value |
|---|---|
| inbound-codec-prefs | OPUS,PCMU,PCMA,G722,G729 |
| outbound-codec-prefs | OPUS,PCMU,PCMA,G722,G729 |
| rtp-secure-media | optional |

### q_tokens
Portal authentication tokens (same as RingQ fax dashboard).
```sql
SELECT access_token FROM q_tokens ORDER BY insert_date DESC LIMIT 1;
```

### q_outbound_rules
RingQ softphone outbound routing. WARule entry is **disabled** (we use dialplan instead).

---

## 10. ACL — IP Whitelisting

The `providers` ACL whitelist allows Meta's SIP traffic through.

### Meta IP Ranges (11 blocks)
```
66.220.144.0/20    66.220.128.0/19    69.63.176.0/20
69.171.224.0/19    173.252.64.0/18    204.15.20.0/22
74.119.76.0/22     157.240.0.0/16     179.60.192.0/22
31.13.24.0/21      31.13.64.0/18
```

### Verify ACL
```bash
fs_cli -x "reloadacl"

# Check count:
psql -h 127.0.0.1 -U ringq -d ringq -c \
  "SELECT COUNT(*) FROM v_access_control_nodes WHERE node_description='Meta WhatsApp SIP';"
# Expect: 11
```

### Add Missing IPs
```sql
DO $$
DECLARE v_acl uuid;
BEGIN
  SELECT access_control_uuid INTO v_acl FROM v_access_controls
  WHERE access_control_name='providers' LIMIT 1;
  INSERT INTO v_access_control_nodes
    (access_control_node_uuid,access_control_uuid,node_type,node_cidr,node_description,insert_date)
  SELECT gen_random_uuid(),v_acl,'allow',cidr,'Meta WhatsApp SIP',NOW()
  FROM (VALUES
    ('66.220.144.0/20'),('66.220.128.0/19'),('69.63.176.0/20'),
    ('69.171.224.0/19'),('173.252.64.0/18'),('204.15.20.0/22'),
    ('74.119.76.0/22'),('157.240.0.0/16'),('179.60.192.0/22'),
    ('31.13.24.0/21'),('31.13.64.0/18')
  ) t(cidr)
  WHERE NOT EXISTS (
    SELECT 1 FROM v_access_control_nodes
    WHERE access_control_uuid=v_acl AND node_cidr=t.cidr
  );
END;
$$;
```
Then: `fs_cli -x "reloadacl"`

---

## 11. SIP Gateway (watrunk)

### Registration: Disabled by Design
```sql
SELECT gateway, register, enabled FROM v_gateways WHERE gateway='watrunk';
-- register = false   ← intentional
-- enabled  = true
```

### Why Registration is Disabled
1. Meta returns `403 Forbidden` when RingQ PBX tries to register
2. Root cause: Meta validates source IP against configured hostname
   - If hostname=domain → DNS lookup → may not match actual packet source IP
   - If hostname=IP → direct match → always works
3. Both inbound and outbound work WITHOUT registration
4. Inline auth credentials handle authentication per-call

### Sofia Status (Expected Output)
```bash
fs_cli -x "sofia status" | grep "wa\.meta"
# internal::58af33af-...  gateway  sip:6567729965@wa.meta.vc:5061  NOREG
# NOREG = register=false (expected, not an error)
```

### Known Display Issue
```bash
fs_cli -x "sofia status gateway watrunk"
# Returns: Invalid Gateway!   ← known RingQ display bug, NOT a real error
# Use: sofia status | grep wa.meta   instead
```

### Prevent Duplicate Gateways
Each "Configure Gateway" click in the portal does DELETE + INSERT (not UPDATE).
This prevents ghost gateways accumulating in memory.
```bash
# After reconfiguring, verify only 1 watrunk:
psql -h 127.0.0.1 -U ringq -d ringq -c \
  "SELECT COUNT(*) FROM v_gateways WHERE gateway='watrunk';"
# Expect: 1
```

---

## 12. SDES / SRTP — Audio Encryption

### Why SDES Not DTLS
| Method | Result |
|---|---|
| **SDES** (Security Descriptions) | ✅ Audio works — keys in SDP crypto lines |
| **DTLS** (Datagram TLS) | ❌ No audio — role conflict (both sides try to be client) |

Meta's SIP gateway uses SDES. RingQ PBX defaults to DTLS. SDES must be explicitly configured.

### Verify SDES on Active Call
```bash
fs_cli -x "show channels" | awk -F',' '{print $21}'
# srtp:sdes:AES_CM_128_HMAC_SHA1_80  ← GOOD
# srtp:dtls:...                       ← BAD (no audio)
# (empty)                             ← needs investigation
```

### Re-apply SDES (from wasip.ringq.io)
```bash
cd /root/wasip-gateway-production && source .env
curl -s -X POST "https://graph.facebook.com/v21.0/$WHATSAPP_PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $WHATSAPP_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"calling":{"srtp_key_exchange_protocol":"SDES"}}' | python3 -m json.tool
```

### SDES Must Always Be Set Last
When you update Meta SIP settings (Step 2), Meta **resets SRTP to DTLS**.
Step 3 (SDES) must always follow as a separate call. Never combine them.

---

## 13. Troubleshooting Guide

### Problem: Calls not arriving at RingQ PBX

```bash
# 1. Verify Meta is pointing to correct IP
cd /root/wasip-gateway-production && source .env
curl -s "https://graph.facebook.com/v21.0/$WHATSAPP_PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $WHATSAPP_ACCESS_TOKEN" | python3 -m json.tool | grep hostname
# Must show: "hostname": "139.59.98.108"

# 2. Verify port 5061 is listening
ss -tlnp | grep 5061

# 3. Check ACL (Meta IP count)
psql -h 127.0.0.1 -U ringq -d ringq -t -A -c \
  "SELECT COUNT(*) FROM v_access_control_nodes WHERE node_description='Meta WhatsApp SIP';"
# Expect: 11
```

---

### Problem: Call arrives but extension doesn't ring

```bash
# Check at least 1 softphone is registered
fs_cli -x "show registrations"

# Watch live dialplan processing during a call
tail -f /var/log/freeswitch/freeswitch.log | grep -iE "6567|transfer|bridge|hangup|cause"
```

---

### Problem: No audio (call connects, silence)

SDES has been reset to DTLS. Fix immediately:
```bash
# From wasip.ringq.io
curl -s -X POST ".../settings" -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"calling":{"srtp_key_exchange_protocol":"SDES"}}' | python3 -m json.tool

# Verify on next call
fs_cli -x "show channels" | grep "srtp:sdes"
```

---

### Problem: Outbound says "please contact administrator"

`prevent-outbound` (order 88) is blocking before WhatsApp Outbound runs.

```sql
-- Check order:
SELECT dialplan_name, dialplan_order, dialplan_enabled
FROM v_dialplans
WHERE dialplan_name = 'WhatsApp Outbound';
-- Must show: order 83, enabled true

-- Fix if wrong:
UPDATE v_dialplans SET dialplan_order = 83 WHERE dialplan_name = 'WhatsApp Outbound';
```
Then: `rm -rf /var/cache/ringq/* && fs_cli -x "reloadxml"`

---

### Problem: Outbound fails with NORMAL_TEMPORARY_FAILURE (480)

Meta returning 480 — IP not matching or SDES issue.

```bash
# Check outbound IP matches Meta config
curl -s ifconfig.me
# Must match hostname in Meta settings

# If hostname is domain name, change to IP:
curl -s -X POST ".../settings" -d '{"calling":{"sip":{"status":"ENABLED","servers":[{"hostname":"YOUR_IP","port":5061}]}}}'
sleep 5
curl -s -X POST ".../settings" -d '{"calling":{"srtp_key_exchange_protocol":"SDES"}}'
```

---

### Problem: Portal shows 500 error

```bash
# Check PHP syntax
php7.4 -l /var/www/wasip/index.php

# Check Phalcon installed for PHP 7.4
php7.4 -m | grep phalcon

# Check correct PHP-FPM socket
ls -la /var/run/php/php7.4-fpm.sock

# Check nginx config
nginx -t
```

---

### Problem: Duplicate watrunk gateways in sofia status

Multiple configure clicks created multiple gateways. Clear ghost gateways:
```bash
rm -rf /var/cache/ringq/*
fs_cli -x "reloadxml"
sleep 3
fs_cli -x "sofia profile internal restart"  # full restart, not rescan
sleep 20
fs_cli -x "sofia status" | grep "wa\.meta"
# Should show exactly 1 entry
```

---

### Complete Status Check (run all at once)
```bash
DB_PASS=$(grep "^database.0.password" /etc/ringq/config.conf | cut -d= -f2 | tr -d ' ')
echo "=== RingQ PBX Sofia Status ===" && fs_cli -x "sofia status" | grep -E "wa\.meta|RUNNING"
echo "=== Softphone Registrations ===" && fs_cli -x "show registrations" | head -3
echo "=== ACL Meta IPs ===" && PGPASSWORD="$DB_PASS" psql -h 127.0.0.1 -U ringq -d ringq -t -A -c \
  "SELECT COUNT(*)||' Meta IPs in ACL' FROM v_access_control_nodes WHERE node_description='Meta WhatsApp SIP';"
echo "=== Dialplan Order ===" && PGPASSWORD="$DB_PASS" psql -h 127.0.0.1 -U ringq -d ringq -t -A -c \
  "SELECT dialplan_name||' | order='||dialplan_order||' | enabled='||dialplan_enabled FROM v_dialplans WHERE dialplan_name IN ('WhatsApp Outbound','WARule') ORDER BY dialplan_order;"
echo "=== watrunk Gateway ===" && PGPASSWORD="$DB_PASS" psql -h 127.0.0.1 -U ringq -d ringq -t -A -c \
  "SELECT gateway||' register='||register||' enabled='||enabled FROM v_gateways WHERE gateway='watrunk';"
echo "=== Gateway Config ===" && PGPASSWORD="$DB_PASS" psql -h 127.0.0.1 -U ringq -d ringq -c \
  "SELECT wa_number, pbx_domain, extension_number, gateway_status, last_configured FROM wa_gateway_config;"
```

---

## 16. Complete API Reference

---

### A. Meta Graph API Endpoints

**Base URL:** `https://graph.facebook.com/v21.0/{PHONE_NUMBER_ID}`
**Auth Header:** `Authorization: Bearer {ACCESS_TOKEN}`
**Content-Type:** `application/json`

---

#### GET — Get Current SIP Settings
```
GET /settings
```
```bash
curl -s "https://graph.facebook.com/v21.0/$PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $ACCESS_TOKEN" | python3 -m json.tool
```
**Response:**
```json
{
  "calling": {
    "status": "ENABLED",
    "call_icon_visibility": "DEFAULT",
    "callback_permission_status": "ENABLED",
    "srtp_key_exchange_protocol": "SDES",
    "sip": {
      "status": "ENABLED",
      "servers": [{"app_id": 1019084797593892, "hostname": "139.59.98.108", "port": 5061}]
    }
  }
}
```

---

#### GET — Get SIP Credentials (Password)
```
GET /settings?include_sip_credentials=true
```
```bash
curl -s "https://graph.facebook.com/v21.0/$PHONE_NUMBER_ID/settings?include_sip_credentials=true" \
  -H "Authorization: Bearer $ACCESS_TOKEN" | python3 -m json.tool
```
**Response adds inside each server object:**
```json
"sip_user_password": "pEwPqO3d92WCvcaI7FzmpOynBd0A97k3"
```
Use this password in `v_gateways.password` and in outbound channel variable `sip_auth_password`.

---

#### GET — Verify Phone Number
```
GET /{PHONE_NUMBER_ID}?fields=id,display_phone_number,verified_name
```
```bash
curl -s "https://graph.facebook.com/v21.0/$PHONE_NUMBER_ID?fields=id,display_phone_number,verified_name" \
  -H "Authorization: Bearer $ACCESS_TOKEN" | python3 -m json.tool
```
**Response:**
```json
{"id": "1214061478463577", "display_phone_number": "+65 6772 9965", "verified_name": "Your Business"}
```
Used by portal Configure step to verify credentials before proceeding.

---

#### POST — Enable WhatsApp Calling (Step 1)
```
POST /settings
Body: {"calling":{"status":"ENABLED","call_icon_visibility":"DEFAULT"}}
```
```bash
curl -s -X POST "https://graph.facebook.com/v21.0/$PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"calling":{"status":"ENABLED","call_icon_visibility":"DEFAULT"}}' \
  | python3 -m json.tool
```
**Response:** `{"success": true}`

---

#### POST — Enable SIP / Set PBX Server (Step 2)
```
POST /settings
Body: {"calling":{"status":"ENABLED","sip":{"status":"ENABLED","servers":[{"hostname":"IP","port":5061}]}}}
```
```bash
curl -s -X POST "https://graph.facebook.com/v21.0/$PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"calling":{"status":"ENABLED","sip":{"status":"ENABLED","servers":[{"hostname":"139.59.98.108","port":5061}]}}}' \
  | python3 -m json.tool
```
**Response:** `{"success": true}`
> ⚠️ This resets `srtp_key_exchange_protocol` to DTLS. Always call SDES endpoint after this.

---

#### POST — Set SDES Encryption (Step 3 — MUST BE LAST)
```
POST /settings
Body: {"calling":{"srtp_key_exchange_protocol":"SDES"}}
```
```bash
curl -s -X POST "https://graph.facebook.com/v21.0/$PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"calling":{"srtp_key_exchange_protocol":"SDES"}}' \
  | python3 -m json.tool
```
**Response:** `{"success": true}`
> ⚠️ Never combine with Step 2. Always a separate call, always last.

---

#### POST — Disable SIP (Emergency / Deactivate)
```
POST /settings
Body: {"calling":{"sip":{"status":"DISABLED"}}}
```
```bash
curl -s -X POST "https://graph.facebook.com/v21.0/$PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"calling":{"sip":{"status":"DISABLED"}}}' \
  | python3 -m json.tool
```
**Response:** `{"success": true}`
All WhatsApp calls stop immediately. Callers hear "unavailable".

---

#### POST — Disable Calling Entirely
```
POST /settings
Body: {"calling":{"status":"DISABLED"}}
```
```bash
curl -s -X POST "https://graph.facebook.com/v21.0/$PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"calling":{"status":"DISABLED"}}' \
  | python3 -m json.tool
```

---

### B. /wasip Portal Internal API Endpoints

**Base URL:** `https://graylunaplc.ringq.ai`
**Auth:** `Authorization: Bearer {TOKEN}` header OR `?token=TOKEN` query param
**Token source:** `SELECT access_token FROM q_tokens ORDER BY insert_date DESC LIMIT 1;`

All endpoints except `/wasip/health` require a valid token.

---

#### GET /wasip/health
Health check — no authentication required.
```
GET https://graylunaplc.ringq.ai/wasip/health
```
```bash
curl https://graylunaplc.ringq.ai/wasip/health
```
**Response:**
```json
{
  "ok": true,
  "php": "7.4.33",
  "phalcon": "4.1.2",
  "db": "ok",
  "host": "ST-Testing-GRAYLUNAPIC"
}
```
Use for deployment verification and monitoring.

---

#### GET /wasip/load
Loads saved configuration + dropdown data for the UI.
Returns real-time routing from `v_dialplan_details` (not just `wa_gateway_config`).
Also auto-syncs `wa_gateway_config.extension_number` if dialplan routing has changed.
```
GET https://graylunaplc.ringq.ai/wasip/load
Authorization: Bearer {TOKEN}
```
**Response:**
```json
{
  "waNumber": "6567729965",
  "phoneNumberId": "1214061478463577",
  "businessAccountId": "3668511866622780",
  "pbxDomain": "139.59.98.108",
  "pbxSipPort": 5061,
  "extensionNumber": "1021",
  "extensionUuid": "48728013-f170-41d4-a287-ea9005d1af4c",
  "hasToken": true,
  "hasSecret": true,
  "hasPassword": true,
  "gatewayStatus": "active",
  "lastConfigured": "2026-09-10 09:14:07",
  "domainName": "graylunaplc.ringq.ai",
  "extensions": [
    {"extension": "1021", "extension_uuid": "...", "effective_caller_id_name": "Paramesh RingQ"},
    {"extension": "1022", "extension_uuid": "...", "effective_caller_id_name": "Thilak Manager"}
  ],
  "queues": [
    {"extension": "9001", "extension_uuid": "...", "effective_caller_id_name": "WhatsApp Queue"}
  ],
  "ivrs": []
}
```

---

#### GET /wasip/status
Real-time gateway status. Called every 30 seconds by the UI.
Extension number is queried live from `v_dialplan_details` (not from cache).
```
GET https://graylunaplc.ringq.ai/wasip/status
Authorization: Bearer {TOKEN}
```
**Response:**
```json
{
  "configured": true,
  "active": true,
  "gatewayStatus": "active",
  "gatewayInDB": true,
  "dialplanInDB": true,
  "waNumber": "6567729965",
  "extensionNumber": "1021",
  "pbxDomain": "139.59.98.108",
  "lastConfigured": "2026-09-10 09:14:07"
}
```
**gatewayStatus values:**

| Value | Meaning |
|---|---|
| `unconfigured` | Portal never run on this PBX |
| `configuring` | Configure button in progress |
| `configured` | API calls done, awaiting verification |
| `active` | Fully operational |
| `deactivated` | Intentionally disabled |
| `failed` | Configuration error |

---

#### POST /wasip/configure
Full gateway setup — calls Meta API, creates DB entries, reloads RingQ PBX.
Takes ~30 seconds.
```
POST https://graylunaplc.ringq.ai/wasip/configure
Authorization: Bearer {TOKEN}
Content-Type: application/json
```
**Request body:**
```json
{
  "accessToken": "EAAxxxxxxx",
  "appSecret": "abc123def456",
  "phoneNumberId": "1214061478463577",
  "businessAcctId": "3668511866622780",
  "waNumber": "6567729965",
  "pbxDomain": "139.59.98.108",
  "pbxSipPort": 5061,
  "extensionNumber": "1021",
  "extensionUuid": "48728013-f170-41d4-a287-ea9005d1af4c"
}
```
> Leave `accessToken` and `appSecret` empty to keep previously saved values.

**Response:**
```json
{
  "success": true,
  "registered": true,
  "steps": [
    {"s": "ok", "m": "Validation passed"},
    {"s": "ok", "m": "Credentials saved (encrypted)"},
    {"s": "ok", "m": "Meta API connected — phone: +65 6772 9965"},
    {"s": "ok", "m": "WhatsApp Calling enabled"},
    {"s": "ok", "m": "SIP mode enabled → 139.59.98.108:5061"},
    {"s": "ok", "m": "SDES key exchange set (required for audio)"},
    {"s": "ok", "m": "Meta SIP password received and saved"},
    {"s": "ok", "m": "RingQ gateway (watrunk) created/updated"},
    {"s": "ok", "m": "Providers ACL updated with Meta IP ranges"},
    {"s": "ok", "m": "SIP profile settings updated (OPUS + SRTP)"},
    {"s": "ok", "m": "DID created in q_dids + dialplan created (+6567729965 → extension 1021)"},
    {"s": "ok", "m": "RingQ reloaded"},
    {"s": "ok", "m": "Gateway active ✓ — Call +6567729965 from WhatsApp to test"}
  ]
}
```
Step `s` values: `ok` = success, `err` = failure, `warn` = warning.

---

#### POST /wasip/reconfigure
Updates Meta SIP hostname to current PBX + rebuilds dialplan.
Use when: changing servers, changing target extension, PBX IP changed.
```
POST https://graylunaplc.ringq.ai/wasip/reconfigure
Authorization: Bearer {TOKEN}
Content-Type: application/json
```
**Request body:**
```json
{
  "extensionNumber": "1022",
  "extensionUuid": "c6dd298d-3845-d0b6-a4aa-xxxxxxxxxxxx"
}
```
Both fields optional — omit to keep current extension.

**Response:**
```json
{
  "success": true,
  "steps": [
    {"s": "ok", "m": "Reconfiguring for: 139.59.98.108 (was: 139.59.98.108)"},
    {"s": "ok", "m": "Meta SIP server updated → 139.59.98.108:5061"},
    {"s": "ok", "m": "SDES confirmed"},
    {"s": "ok", "m": "SIP password updated"},
    {"s": "ok", "m": "Gateway updated with new PBX domain"},
    {"s": "ok", "m": "DID rebuilt in q_dids + dialplan → 1022 @ graylunaplc.ringq.ai"},
    {"s": "ok", "m": "RingQ reloaded — reconfiguration complete"}
  ]
}
```

---

#### POST /wasip/deactivate
Removes all WhatsApp configuration. Calls from WhatsApp stop immediately.
Credentials remain encrypted in DB for future reconfiguration.
```
POST https://graylunaplc.ringq.ai/wasip/deactivate
Authorization: Bearer {TOKEN}
Content-Type: application/json
Body: {} (empty)
```
**Response:**
```json
{
  "success": true,
  "steps": [
    {"s": "ok", "m": "SIP disabled on Meta — calls will stop"},
    {"s": "ok", "m": "watrunk gateway removed"},
    {"s": "ok", "m": "DID, dialplan, and dialplan details removed"},
    {"s": "ok", "m": "Meta IP ranges removed from providers ACL"},
    {"s": "ok", "m": "Configuration marked as deactivated"},
    {"s": "ok", "m": "RingQ reloaded — WhatsApp gateway fully deactivated"}
  ]
}
```
**What gets deleted:**
- `v_gateways` WHERE gateway=watrunk
- `q_dids` WHERE did_number=wa_number
- `v_dialplan_details` WHERE dialplan matches WA number
- `v_dialplans` WHERE dialplan_number=wa_number (public context)
- `v_access_control_nodes` WHERE node_description='Meta WhatsApp SIP'

**What is NOT deleted:**
- `wa_gateway_config` row (kept, status set to 'deactivated')
- Encrypted credentials (kept for easy reconfiguration)

---

### C. API Error Responses

All /wasip endpoints return HTTP 401 for invalid token:
```json
{"error": "Invalid token. Check q_tokens table."}
```

All /wasip endpoints return HTTP 500 for PHP errors:
```json
{"error": "PHP[2]: Division by zero in /var/www/wasip/index.php:123"}
```

Configure/Reconfigure/Deactivate return `success: false` with last completed steps on partial failure:
```json
{
  "success": false,
  "error": "Meta API: Invalid OAuth access token.",
  "steps": [
    {"s": "ok",  "m": "Validation passed"},
    {"s": "ok",  "m": "Credentials saved (encrypted)"},
    {"s": "err", "m": "Meta API: Invalid OAuth access token."}
  ]
}
```

---

### D. Quick API Test — Verify Everything

```bash
TOKEN="69494f7772e453975266ca7a98fe808f"
PBX="https://graylunaplc.ringq.ai"

echo "=== Health (no auth) ==="
curl -s "$PBX/wasip/health" | python3 -m json.tool

echo "=== Status ==="
curl -s -H "Authorization: Bearer $TOKEN" "$PBX/wasip/status" | python3 -m json.tool

echo "=== Load (extensions list) ==="
curl -s -H "Authorization: Bearer $TOKEN" "$PBX/wasip/load" | python3 -m json.tool | head -30
```


---

## 14. Key Lessons Learned

### 1. SDES must be 3rd separate API call
Combining SDES with SIP enable resets encryption to DTLS → no audio.
Always: Enable Calling → Enable SIP → Set SDES (separate call, last).

### 2. Use IP address in Meta, not hostname
`hostname=graylunaplc.ringq.ai` → Meta validates via DNS → sometimes mismatch → 403/480
`hostname=139.59.98.108` → direct IP match → always works

### 3. No gateway registration needed
Both inbound and outbound work with `register=false`.
Inline per-call auth replaces persistent registration.

### 4. Dialplan order 83 is critical
WhatsApp Outbound MUST be below 88 (prevent-outbound). Order 83 works reliably.

### 5. DELETE + INSERT prevents ghost gateways
Always DELETE existing watrunk rows before INSERT when configuring.
Multiple rows cause competing registrations and 403 errors from Meta.

### 6. `show dialplan` always blank — correct
mod_xml_curl serves dialplans dynamically. They're never in RingQ memory.

### 7. `sofia status gateway watrunk` → "Invalid Gateway!" — ignore
RingQ display bug. Use `sofia status | grep wa.meta` instead.

### 8. Domain from HTTP_HOST not gethostname()
PHP `gethostname()` returns Linux hostname (e.g. "debian"), not the domain.
Always use `$_SERVER['HTTP_HOST']` for domain detection.

### 9. q_dids deletion by did_number not app_uuid
`v_dialplans.app_uuid` = extension UUID (not q_dids.did_uuid).
Delete from q_dids using `WHERE did_number = wa_number`.

### 10. Reload sequence matters
```bash
rm -rf /var/cache/ringq/*   # clear PHP cache
fs_cli -x "reloadxml"       # reload dialplans
fs_cli -x "reloadacl"       # reload ACL (if changed)
fs_cli -x "sofia profile internal rescan"  # reload gateways
# Use 'restart' instead of 'rescan' to clear ghost gateways
```

---

## 15. Fresh PBX Deployment Checklist

### Prerequisites
- [ ] RingQ PBX installed and running
- [ ] TLS certificate configured (`/etc/freeswitch/tls/agent.pem`)
- [ ] Port 5061 (TLS) open inbound from Meta IPs
- [ ] RTP ports 16384-32768 open inbound
- [ ] Port 7443 open (WebRTC softphone)
- [ ] PHP 7.4 + Phalcon installed
- [ ] wasip/index.php at `/var/www/wasip/`
- [ ] Nginx location block added for `/wasip`

### Step 1 — Configure Meta (from wasip.ringq.io)
```bash
cd /root/wasip-gateway-production && source .env

# Enable calling
curl -s -X POST "https://graph.facebook.com/v21.0/$WHATSAPP_PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $WHATSAPP_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"calling":{"status":"ENABLED","call_icon_visibility":"DEFAULT"}}' | python3 -m json.tool

# Enable SIP with YOUR PBX IP
curl -s -X POST "https://graph.facebook.com/v21.0/$WHATSAPP_PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $WHATSAPP_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"calling":{"status":"ENABLED","sip":{"status":"ENABLED","servers":[{"hostname":"YOUR_PBX_IP","port":5061}]}}}' | python3 -m json.tool

# Set SDES last
curl -s -X POST "https://graph.facebook.com/v21.0/$WHATSAPP_PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $WHATSAPP_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"calling":{"srtp_key_exchange_protocol":"SDES"}}' | python3 -m json.tool
```

### Step 2 — Configure RingQ PBX (via /wasip portal)
1. Open `https://YOUR-PBX-IP/wasip?token=YOUR_TOKEN`
2. Click Configure tab
3. Fill in all Meta credentials
4. **PBX Domain field: enter YOUR PBX IP** (not hostname)
5. Select extension/queue from dropdown
6. Click Configure Gateway → wait for all steps green
7. Click Manage tab → verify it appeared (gateway is active)

### Step 3 — Verify
```bash
# On PBX:
fs_cli -x "show registrations"           # agents online?
fs_cli -x "sofia status" | grep wa.meta  # gateway entry exists?

# From wasip.ringq.io:
curl -s "https://graph.facebook.com/v21.0/$WHATSAPP_PHONE_NUMBER_ID/settings" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool | grep -E "hostname|srtp"
# hostname = YOUR_IP, srtp = SDES
```

### Step 4 — Test
- **Inbound**: Call your WA number from WhatsApp → extension rings
- **Outbound**: From softphone dial customer number → their WhatsApp rings

---

### Migrating to New PBX
1. Deploy wasip on new PBX
2. From wasip.ringq.io: run Steps 1 above with new PBX IP
3. Open wasip on NEW PBX
4. Click Configure → fill credentials → Configure Gateway
5. Old PBX stops receiving calls immediately (Meta points to new IP)

### Changing Routing (Extension → Queue → IVR)
- Via portal: Manage tab → change dropdown → Reconfigure (no Meta API needed)
- Via RingQ DID Manager: change directly → portal reads it automatically
- Via SQL: update `v_dialplan_details` transfer action + reload XML

---

*Complete reference for WhatsApp SIP Gateway on RingQ PBX*
*Production: graylunaplc.ringq.ai (139.59.98.108)*
*WA Business: +6567729965*
