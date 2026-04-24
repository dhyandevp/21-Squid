# 🦑 21 Squid

> Network-wide HTTPS ad elimination via SSL interception. Deeper than DNS. Cleaner than anything.

---

## What is 21 Squid?

21 Squid is a self-hosted network gateway that intercepts, decrypts, and cleans HTTPS traffic at the payload level before it reaches any device on your network.

Standard DNS blockers only see domain names — they are blind to first-party ad content served from the same domain as the content you actually want. 21 Squid sees **inside** the encrypted stream, stripping embedded ads from YouTube, Spotify, and anywhere else they hide behind HTTPS.

---

## How it works

```
Internet → pfSense → Squid SSL Bump → SquidGuard → Your devices
                          ↑
               Decrypts · Strips ads · Re-encrypts
```

All HTTPS traffic is routed through a transparent proxy. A locally-generated Certificate Authority (21 Squid CA) is distributed to your devices, allowing the proxy to decrypt traffic, inspect the payload, drop ad content, and re-encrypt the stream. From the device's perspective, the connection looks normal.

---

## The Stack

| Component | Name in this project | Role |
|---|---|---|
| pfSense | The Gateway | Routes all network traffic |
| Squid proxy | The Interceptor | SSL bump — decrypts and re-encrypts HTTPS |
| SquidGuard | The Filter | Drops ad and telemetry payloads |
| pfBlockerNG | The DNS Layer | First line of defense — domain-level blocking |
| Local CA cert | 21 Squid CA | Makes SSL interception trusted by devices |

---

## Project Structure

```
21-squid/
├── README.md
├── certs/
│   └── 21squid-ca.crt          # Distribute this to all devices
├── config/
│   ├── squid.conf               # Squid proxy configuration
│   ├── squidguard.conf          # SquidGuard ACL rules
│   └── pfsense-nat-rules.xml    # pfSense NAT port forward rules
├── lists/
│   ├── ad_domains.list          # Ad domain blocklist
│   └── telemetry_domains.list   # Telemetry and tracking blocklist
└── bypass/
    └── pinned_bypass.acl        # Cert-pinned app passthrough rules
```

---

## Deployment Phases

### Phase 1 — Gateway setup

1. Install pfSense on dedicated hardware (AES-NI required for SSL bump performance)
2. Assign interfaces:
   - `LAN` — managed devices (intercepted)
   - `OPT1` — IoT / pinned-cert VLAN (bypass)
3. Disable hardware offloading (System > Advanced > Networking)
4. Block UDP/443 outbound on LAN — forces QUIC fallback to TCP so Squid can intercept
5. Install pfBlockerNG for DNS-layer filtering

### Phase 2 — Interceptor deployment

1. Install packages: `squid`, `squidguard`, `lightsquid`
2. Generate the **21 Squid CA** (System > Cert Manager > CAs)
   - RSA 4096 or EC P-384
   - SHA-256
   - Keep this cert safe — it signs everything
3. Enable SSL bump on Squid:
   - Transparent HTTPS proxy on port `3129`
   - SSL/MITM mode: Splice All, then carve out bypass ACLs
4. Configure NAT to redirect LAN port 443 → `127.0.0.1:3129`
5. Load ad and telemetry domain lists into SquidGuard

### Phase 3 — CA distribution

Push the `21squid-ca.crt` to every managed device:

**Windows (GPO)**
```powershell
certutil -addstore -f "ROOT" 21squid-ca.crt
```

**macOS**
```bash
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain 21squid-ca.crt
```

**Linux**
```bash
sudo cp 21squid-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

**Android** — Requires MDM (Intune/Jamf) or rooted device. User-installed CAs are ignored by apps on Android 7+.

**iOS** — Install via profile, then enable full trust under Settings > General > About > Certificate Trust Settings.

---

## Certificate Pinning Bypass

Some apps hardcode their own CA and will reject the 21 Squid CA entirely, causing connectivity failures. These are routed through the `OPT1` VLAN and passed through untouched — falling back to DNS-layer filtering only.

Known bypass candidates:

| App / Service | Reason |
|---|---|
| Spotify (app) | Transport cert pinning |
| Banking apps | Regulatory requirement |
| WhatsApp / Signal | E2E + transport pins |
| Smart TVs / Roku | Embedded CA stores |
| Ring / Nest / Ecobee | IoT hardcoded certs |

Bypass ACL in `squid.conf`:
```squid
acl PINNED_BYPASS ssl::server_name .spotify.com
acl PINNED_BYPASS ssl::server_name .whatsapp.net
acl PINNED_BYPASS ssl::server_name .icloud.com
ssl_bump splice PINNED_BYPASS
ssl_bump bump all
```

---

## What gets blocked

**Payload-level (Squid):**
- YouTube pre-roll and mid-roll ads
- Spotify injected audio ads (browser/web player)
- Embedded tracking pixels
- First-party telemetry endpoints

**DNS-level (pfBlockerNG):**
- Known ad networks and CDNs
- Third-party trackers
- Malware domains

---

## Known Limitations

- **Spotify desktop/mobile app** — cert pinning means audio ads fall back to DNS blocking only. Full elimination requires web player via browser.
- **YouTube mobile app** — SSL bump works on managed devices with CA installed. Works best on desktop browsers.
- **Android 7+** — Apps ignore user-installed CAs. MDM or root required for full interception.
- **HTTP/3 (QUIC)** — Must block UDP/443 or Squid is bypassed entirely.
- **Firefox** — Uses its own trust store. Must import 21 Squid CA separately via `about:preferences#privacy`.

---

## Hardware Recommendations

| Use case | Recommended hardware |
|---|---|
| Home network (< 50 devices) | Protectli VP2420, N100 mini PC |
| Small office (50–200 devices) | Protectli VP4650, Intel i5 NUC |
| Performance check | Run `openssl speed -evp aes-128-gcm` — AES-NI must be active |

---

## License

MIT — do whatever you want with it.

---

*21 Squid — deep packet, no ads, no mercy.*
