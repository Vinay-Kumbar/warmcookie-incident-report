# Incident Report: WarmCookie Malware Infection

**Analyst:** Vinay
**Date:** July 29, 2026
**Environment:** LAFONTAINEBLEU domain (lafontainebleu.org)
**Source:** Packet capture provided for internal investigation

---

## Executive Summary

On the morning of August 15, 2024, a workstation on the LAFONTAINEBLEU network picked up an infection that traces back to a phishing email dressed up as a FedEx shipment notice. The user, going by the account "plucero," opened what looked like an invoice — a JavaScript file named `Invoice-876597035-003-8331775-8334138.js` — hosted on a lookalike domain, `quote.checkfedexexp.com`. That script pulled down a follow-up payload from `199.232.210.172`, and shortly after, the machine started checking in repeatedly with an external server at `72.5.43.29`, which is consistent with WarmCookie's known command-and-control behavior. When I tried to pull a copy of the delivered file for hash analysis, Windows Defender flagged it as malicious on the spot, which pretty much confirms what the traffic pattern was already suggesting.

Bottom line: one workstation is compromised, the infection started with a phishing lure, and there's an active C2 channel that needs to be cut off.

---

## Victim Details

| Field | Value |
|---|---|
| IP Address | 10.8.15.133 |
| Hostname | DESKTOP-H8ALZBV |
| MAC Address | 00:1c:bf:03:54:82 |
| Domain | lafontainebleu.org |
| Windows Username | plucero |

I pulled the hostname and IP from the DNS/mDNS chatter early in the capture — the machine was announcing itself as `DESKTOP-H8ALZBV.local` right after it grabbed a DHCP lease. The MAC address came straight off the Ethernet header in the same broadcast traffic. For the username, I filtered on Kerberos authentication requests and skipped past the machine account (`desktop-h8alzbv$`, which every domain-joined computer has automatically) to find the actual human login — `plucero` — in a separate AS-REQ a bit later in the capture.

*(Screenshot: MAC address and hostname from Ethernet/DHCP frames)*
*(Screenshot: Kerberos AS-REQ showing CNameString "plucero")*

---

## What Actually Happened

Going through the capture chronologically, the machine behaves completely normally at first — DHCP lease, ARP, the usual Active Directory noise (LDAP lookups, Kerberos tickets, SMB traffic to the domain controller). Nothing unusual for a good chunk of the capture.

Things change when the machine reaches out to `quote.checkfedexexp.com`. Following that HTTP stream, I could see it grabbed a file called `Invoice-876597035-003-8331775-8334138.js` — the "Invoice" naming is a pretty standard phishing trick, playing on the fact that people tend to open anything that looks like it's about money or a package they're expecting.

After that, the traffic pattern shifts. There's a series of HEAD and GET requests to `199.232.210.172`, hitting an endpoint called `/filestreamingservice/files/` with random UUID-style filenames. That's the malware fetching its actual payload — the JS file was just the delivery mechanism, not the final malware itself.

Not long after that, I started seeing repeated GET and POST requests to `72.5.43.29`, with almost nothing in the request path — just bare `GET /` and `POST /` calls, happening over and over at regular intervals. That's a beaconing pattern: the infected machine checking in with its command server, waiting for instructions. This lines up with the SIEM alert that kicked off this whole investigation, which flagged NetSupport Manager RAT activity on port 443 from a similar external IP.

*(Screenshot: HTTP requests showing traffic to 199.232.210.172 and 72.5.43.29)*

To try and get a hash of the actual payload for this report, I exported the HTTP objects from the capture and attempted to save the file tied to the `quote.checkfedexexp.com` request. Windows Defender caught it immediately and flagged it as a live threat before I could even open it. I didn't try to bypass that — there's no good reason to have working malware sitting on a machine outside of a proper isolated sandbox, and honestly, getting flagged in real time is stronger confirmation than a hash lookup would have been anyway.

*(Screenshot: Windows Security "Threats found" notification during file export)*

---

## Indicators of Compromise (IOCs)

**Malicious domain:**
- `quote.checkfedexexp.com` (phishing lure, FedEx impersonation)

**Malicious IP addresses:**
- `199.232.210.172` — payload staging/delivery
- `72.5.43.29` — command-and-control (C2) beaconing
- `104.21.55.70` — additional suspicious check-in traffic observed earlier in the capture

**Malicious file:**
- `Invoice-876597035-003-8331775-8334138.js` — initial delivery script, disguised as an invoice
- Payload confirmed malicious via real-time Windows Defender detection (file not retained for safety reasons)

---

## Recommendations

- Isolate DESKTOP-H8ALZBV from the network immediately and hand it off for a proper forensic image/wipe — beaconing to a live C2 means the attacker likely still has some level of access
- Block the three IPs and the phishing domain at the firewall/proxy level
- Reset plucero's domain credentials as a precaution, since anything typed on a compromised machine should be considered exposed
- Search mail logs for the original phishing email so it can be pulled from other inboxes before anyone else clicks it
- Worth a reminder to staff about invoice/shipping-themed phishing — this lure style works because it plays on urgency, and FedEx/DHL/invoice impersonation is common enough that it's worth calling out specifically in training

---

*This investigation was conducted using a publicly available training exercise pcap from malware-traffic-analysis.net, for educational purposes.*
