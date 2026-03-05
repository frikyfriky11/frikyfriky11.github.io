+++
date = 2026-03-05T19:25:08+01:00
title = "Fixing ERR_ECH_FALLBACK_CERTIFICATE_INVALID with Split-Horizon DNS and Cloudflare"
description = "How a modern browser privacy feature called ECH clashes with split-horizon DNS setups — and how to fix it cleanly via the Cloudflare API."
authors = ["Stefano Previato"]
tags = ["DNS", "Cloudflare", "homelab", "networking", "Traefik", "UniFi"]
categories = ["Homelab"]
+++

## TL;DR

If you run a split-horizon DNS setup (local DNS overrides + Cloudflare tunnels for external access) and started seeing `ERR_ECH_FALLBACK_CERTIFICATE_INVALID` in Chrome, here's what's happening and how to fix it:

- Cloudflare automatically publishes **ECH configuration** inside DNS type 65 (HTTPS) records for all proxied domains
- Chrome fetches these records and tries to use ECH when connecting — even if your local DNS is sending it to an internal server that knows nothing about ECH
- The mismatch causes the error
- **The fix**: disable ECH for your zones via the Cloudflare API (works on all plans, including free):

```bash
curl -s -X PATCH "https://api.cloudflare.com/client/v4/zones/{YOUR_ZONE_ID}/settings/ech" \
  -H "X-Auth-Email: you@example.com" \
  -H "X-Auth-Key: YOUR_GLOBAL_API_KEY" \
  -H "Content-Type: application/json" \
  --data '{"value":"off"}'
```

Run this once per zone and you're done.

---

## The Setup: Why Split-Horizon DNS?

My home network runs a reverse proxy — [Traefik](https://traefik.io/) — inside a Docker container. All self-hosted services (dashboards, media servers, password managers, and so on) sit behind it, dispatched by hostname. Something like this:

```
# External request from anywhere
https://files.myhomelab.dev
    → Cloudflare (proxied) → Cloudflare Tunnel → Traefik → internal service

# Internal request from home network
https://files.myhomelab.dev
    → Local DNS → Traefik directly (e.g. 192.168.1.20) → internal service
```

This pattern is called **split-horizon DNS** (or split-brain DNS). The same hostname resolves to different addresses depending on where you're asking from. When you're at home, your local DNS resolver intercepts the query and returns the internal IP directly — bypassing Cloudflare entirely. When you're on the road, public DNS resolves it normally through Cloudflare.

The benefits are real: internal traffic never leaves your network, latency is minimal, and you don't pay Cloudflare bandwidth costs for streaming a local video to yourself. The router (a UniFi Dream Machine Pro SE in my case) runs dnsmasq under the hood, and you can add local DNS overrides through the UniFi dashboard — no extra infrastructure needed.

It works beautifully. Until it doesn't.

---

## The Error: ERR_ECH_FALLBACK_CERTIFICATE_INVALID

One day, after a Chrome update, some internal services started throwing this error:

```
ERR_ECH_FALLBACK_CERTIFICATE_INVALID
```

No explanation, no useful detail in the browser. Just a red lock and a dead page. Other browsers (older ones, or Firefox with ECH disabled) were fine. The services were running. The certificates were valid. And yet — Chrome refused to connect.

---

## Understanding the Pieces

### DNS Type 65: The HTTPS Resource Record

Most people are familiar with DNS A records (IPv4 address), AAAA records (IPv6), and CNAME records (aliases). But there's a newer record type: **HTTPS** (type 65), standardised in [RFC 9460](https://www.rfc-editor.org/rfc/rfc9460).

This record is designed to carry metadata about how to connect to a service over HTTPS. It can advertise supported protocols (like HTTP/2 or HTTP/3), provide IP hints for faster connections, and — most importantly for our story — publish **ECH configuration**.

You can query it directly:

```bash
dig TYPE65 files.myhomelab.dev @1.1.1.1
```

On a Cloudflare-proxied domain with ECH enabled, you'll see something like this in the answer:

```
files.myhomelab.dev. 300 IN HTTPS 1 . alpn="h2,h3" \
  ipv4hint=104.21.57.4,172.67.139.92 \
  ech=AEX+DQA... (long base64 blob) \
  ipv6hint=2606:4700::...
```

That `ech=` field is an ECH configuration structure. It contains a **public key** and metadata that clients can use to initiate an Encrypted Client Hello.

### What is ECH?

**Encrypted Client Hello (ECH)** is a TLS 1.3 extension that solves a long-standing privacy leak. In standard TLS, the very first message a client sends — the `ClientHello` — includes a field called the **Server Name Indication (SNI)**. The SNI tells the server which certificate to use (important for virtual hosting), but it travels in plaintext. Anyone observing the network traffic — your ISP, a corporate firewall, a government-level observer — can see exactly which hostname you're connecting to, even though the rest of the connection is encrypted.

ECH fixes this. The client fetches the server's ECH public key from the DNS HTTPS record, then uses it to encrypt the real (inner) `ClientHello`. An observer only sees an outer `ClientHello` addressed to a generic cover domain (in Cloudflare's case, `cloudflare-ech.com`) — not your actual service hostname.

It's a meaningful privacy improvement, and Cloudflare has been rolling it out across all their infrastructure.

### The Collision

Here's where split-horizon DNS and ECH collide.

When Chrome navigates to `https://files.myhomelab.dev`, it fires off **two DNS queries in parallel**:

1. An **A record** query — to get the IP address
2. An **HTTPS record (type 65)** query — to get connection metadata, including ECH config

In a split-horizon setup, your local DNS resolver handles the A record query and correctly returns the internal IP (say, `192.168.1.20`). But the HTTPS record query is a different story. Your local DNS has no type 65 records configured for your domain — the UniFi dashboard has no concept of them. So the query falls through to your upstream public resolver (1.1.1.1 or 8.8.8.8), which happily returns Cloudflare's HTTPS record — complete with the ECH public key.

Now Chrome has:

- **IP**: `192.168.1.20` (your local Traefik instance)
- **ECH config**: Cloudflare's public key

Chrome connects to `192.168.1.20` and attempts to use ECH, encrypting the `ClientHello` with Cloudflare's public key. Your Traefik instance receives the connection — and has absolutely no idea what to do with it. Traefik doesn't hold Cloudflare's private key. It can't decrypt the `ClientHello`.

What happens next: Traefik falls back and presents its normal TLS certificate (issued by Let's Encrypt, or self-signed). Chrome checks whether this certificate is legitimate in the ECH fallback context — specifically, whether it matches what Cloudflare would present as the outer server. It doesn't. Cloudflare and your homelab Traefik instance are very different entities.

Result: `ERR_ECH_FALLBACK_CERTIFICATE_INVALID`.

---

## Why This Appeared Suddenly

ECH support in Chrome has been rolling out gradually. The HTTPS DNS record (type 65) query was added to Chrome's DNS resolution path as part of this rollout. If your setup worked fine before and then suddenly broke after a Chrome update, this is exactly why — Chrome started querying type 65, found Cloudflare's ECH config, and tried to use it.

It only affects clients that support ECH (Chrome, Edge, modern Firefox) and only for domains where Cloudflare is publishing an ECH config. Safari and older browsers are unaffected, which is why the error looks inconsistent across devices.

---

## Possible Solutions

Once you understand the problem, several solutions come to mind — with varying trade-offs.

### Option 1: Disable ECH in the browser

Chrome has an internal flag to disable ECH: `chrome://flags/#encrypted-client-hello`. Set it to disabled.

This works, but it's a per-device change. In a household with multiple computers, phones, and tablets, you'd need to touch every single device. It also opts you out of a privacy feature that's beneficial everywhere else. Not great.

### Option 2: Block type 65 queries at the DNS level

If the browser never receives a type 65 HTTPS record with an ECH config, it never attempts ECH. You can configure your local DNS resolver to intercept and nullify type 65 queries.

**With dnsmasq** (the DNS server running on UniFi equipment), you can use the `local=` directive to make dnsmasq authoritative for your domains. When dnsmasq is authoritative, it answers from its own records and never forwards to upstream. Since dnsmasq has no HTTPS records, type 65 queries for those domains return NXDOMAIN.

```conf
local=/myhomelab.dev/
local=/personal-cloud.site/
```

**With AdGuard Home**, there's a checkbox under DNS settings: _Block HTTPS records_. One toggle, network-wide, no config files to edit.

The downside of this approach: it affects all devices on your network globally, opting everyone out of ECH even for external websites. It also requires changes to your network infrastructure that may need to survive router reboots and firmware updates — which can be fiddly depending on your equipment.

In my case on the UniFi Dream Machine Pro SE, dnsmasq's configuration is fully managed by UniFi's internal daemon (`ubios-udapi-ser`), which regenerates the config from its own database on every DNS change or restart. Making custom directives stick permanently requires creative approaches (boot scripts, binary wrappers) that feel fragile alongside routine network administration. Doable, but not clean.

### Option 3: Disable ECH at the Cloudflare level (the right answer)

The cleanest fix is to tell Cloudflare to stop publishing ECH configuration in the HTTPS records for your zones. No browser changes, no DNS hacks, no infrastructure modifications. Fix it at the source.

---

## The Fix: Disabling ECH via the Cloudflare API

Cloudflare's dashboard may or may not show a toggle for ECH depending on your plan and current UI version (it's been inconsistently available). However, the **API works on all plans**, including free.

### What you need

- Your **Zone ID**: find it in the Cloudflare dashboard for your domain, right sidebar
- Your **Global API Key** (or an API token with `Zone:Edit` permissions): _My Profile → API Tokens_

### Check current status

```bash
curl -s -X GET \
  "https://api.cloudflare.com/client/v4/zones/{ZONE_ID}/settings/ech" \
  -H "X-Auth-Email: you@example.com" \
  -H "X-Auth-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json"
```

You'll get back something like:

```json
{
  "result": {
    "id": "ech",
    "value": "on",
    "editable": true
  },
  "success": true
}
```

### Disable ECH

```bash
curl -s -X PATCH \
  "https://api.cloudflare.com/client/v4/zones/{ZONE_ID}/settings/ech" \
  -H "X-Auth-Email: you@example.com" \
  -H "X-Auth-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  --data '{"value":"off"}'
```

If you have multiple domains, run this once per zone. Each domain in Cloudflare is its own zone with its own zone ID.

### Verify it worked

After a few minutes (DNS TTL needs to expire), query the HTTPS record from a public resolver:

```bash
dig TYPE65 files.myhomelab.dev @1.1.1.1
```

With ECH disabled, the `ech=` field should be gone from the answer:

```
files.myhomelab.dev. 300 IN HTTPS 1 . alpn="h2" \
  ipv4hint=104.21.57.4,172.67.139.92 \
  ipv6hint=2606:4700::...
```

No `ech=` means no ECH config is being advertised. Chrome will not attempt ECH for this domain, and your split-horizon setup works exactly as before.

---

## A Note on Cloudflare Free Plan Limits

ECH is one of several TLS features that Cloudflare controls at the infrastructure level. On the free plan, the dashboard gives you limited visibility into these settings — you may not see a toggle for ECH at all, or it may appear but do nothing. This is a known inconsistency that has been reported in the Cloudflare community forums.

The API, however, is a different story. The `ech` setting is exposed via the Zones Settings API endpoint and is editable regardless of plan. If you're on the free tier and can't find the toggle in the UI, go straight to the API — it's the reliable path.

A broader observation: Cloudflare's free plan is remarkably capable, but features around TLS behaviour, performance tuning, and security policies are progressively unlocked at higher tiers (Pro, Business, Enterprise). ECH disabling via API appears to be one of the exceptions where the free tier has full API access despite limited UI exposure.

---

## How to Test Whether Your Domain Is Affected

Before doing anything, it's worth confirming whether your domain actually serves ECH config. Three quick methods:

**1. DNS query (most direct)**

```bash
dig TYPE65 yourdomain.com @1.1.1.1
```

Look for an `ech=` field in the HTTPS record answer. If it's there, your domain advertises ECH.

**2. Chrome DevTools**

Open Chrome, navigate to your domain, open DevTools → Security tab. If ECH was negotiated (or attempted and failed), Chrome reports it there.

**3. Check from inside your local network vs outside**

If the error only appears on your home network but the same URL works fine from a mobile connection (4G, no local DNS), that's a strong signal the issue is split-horizon DNS + ECH — not a certificate problem or server misconfiguration.

---

## Conclusion

ECH is a genuinely good privacy feature. Encrypting the SNI closes one of the last observable metadata leaks in HTTPS connections, and Cloudflare deserves credit for rolling it out at scale. But it introduces a subtle assumption: the server you're connecting to at the IP level is the same entity that published the ECH configuration in DNS.

Split-horizon DNS breaks that assumption by design. The whole point is that internally you bypass Cloudflare entirely — so the ECH keys Cloudflare published are useless to your local Traefik instance.

The fix is simple once you know where to look. Disable ECH at the Cloudflare zone level via the API, and you eliminate the root cause cleanly. Your split-horizon setup continues to work exactly as designed: local traffic stays local, external traffic goes through Cloudflare, and the browser stops complaining.

The broader lesson: as browsers adopt new privacy and security features (ECH, HTTPS records, HTTP/3 via QUIC), they interact in increasingly non-obvious ways with non-standard network configurations like homelabs. Staying on top of what type 65 records are actually being published for your domains — and what metadata they carry — is becoming a routine part of self-hosted infrastructure hygiene.
