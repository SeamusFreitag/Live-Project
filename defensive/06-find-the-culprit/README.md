# Find the Culprit

Analysis of a provided packet capture from an infected Windows host. The capture documents an active malware infection communicating with its command-and-control server. The task is to identify the victim, the malware, and the root cause.

All analysis was done inside an isolated Kali VM on KVM, reverted to a clean snapshot beforehand. The network link was disabled before the archive was extracted. Carved files were hashed inside the VM and only the hashes were looked up on VirusTotal from the host. Nothing was extracted to the host and nothing was executed.

## Scoping the Capture

The file holds 1,961 packets spanning 2015-09-22 22:32:14 to 22:57:28 UTC, just over 25 minutes. Wireshark's capture properties dialog renders those times in the analyst machine's local zone, so the packet list time column was set to UTC and every timestamp below is UTC. The C2 server's own HTTP response headers later confirmed this independently.

Protocol Hierarchy first, before any filtering, to see what is in the file.

![Protocol hierarchy](./media/defensive06-protocol-hierarchy.png)

*Protocol Hierarchy across all 1,961 packets. HTTP over plain TCP, DNS, and workgroup broadcast traffic.*

The mix is narrow: HTTP, DNS, NBNS, DHCP and some multicast. No TLS, which means the C2 runs in the clear over HTTP and every request and response is readable. The 16 malformed packets under JPEG File Interchange Format are noted here and explained later.

## Identifying the Host

DHCP, NBNS and the Ethernet header between them give the full victim profile.

![DHCP traffic](./media/defensive06-dhcp-lease.png)

*DHCP Request at 22:32:14 and periodic renewals from 10.54.112.205.*

![NBNS registrations](./media/defensive06-nbns-hostname-citibank.png)

*NetBIOS name registrations for PENDJIEK-PC and workgroup WORKGROUP.*

The host registers as PENDJIEK-PC in WORKGROUP. A `kerberos.CNameString` filter returns nothing, so there is no Active Directory domain here and no domain account to attribute activity to.

![Ethernet header](./media/defensive06-victim-mac.png)

*Ethernet source address 00:50:8b:01:db:2f, OUI Hewlett-Packard.*

For the operating system, the User-Agent strings in the capture belong to Windows' own connectivity check and to the malware. The malware's own requests carry a full Internet Explorer 8 string including real .NET CLR version tokens, which is consistent with requests issued through the system's own HTTP stack rather than a forged header.

![Frame 66 headers](./media/defensive06-frame66-useragent-host.png)

*Frame 66. Windows NT 6.1 in the User-Agent and Host: classicalbitu[.]com.*

Windows NT 6.1 is Windows 7. The mshome.net DNS suffix and the NetBIOS registrations corroborate Windows independently.

| Field | Value |
|---|---|
| IP | 10.54.112.205 |
| MAC | 00:50:8b:01:db:2f (Hewlett-Packard) |
| Host name | PENDJIEK-PC |
| Domain | none, workgroup WORKGROUP |
| OS | Windows 7 |

## What the Host Talked To

Filtering on HTTP requests shows where the traffic actually went.

![All HTTP requests](./media/defensive06-http-requests-overview.png)

*Every HTTP request in the capture. Nearly all directed at one host.*

Setting aside the repeating traffic leaves three requests in the whole file.

![Non-C2 HTTP requests](./media/defensive06-non-c2-http-requests.png)

*The only HTTP requests not directed at 193.23.181[.]155.*

One is Windows' NCSI connectivity check at 22:32:34, which every Windows machine performs on joining a network. The other two go to 5.255.255.5 at 22:41:23, seconds after the repeating traffic starts, and are examined further below.

DNS shows the same shape.

![DNS traffic](./media/defensive06-dns-overview.png)

*All DNS queries and responses.*

The four NXDOMAIN responses early in the capture are for `_ldap._tcp.dc._msdcs.mshome.net`, `wpad.mshome.net` and `isatap.mshome.net`. These are standard Windows workgroup lookups on a consumer router handing out an mshome.net suffix, not algorithmically generated domains. They confirm the absence of a domain controller and nothing more.

The one lookup that matters resolves at 22:41:15.

![DNS A records](./media/defensive06-dns-classicalbitu-resolution.png)

*classicalbitu[.]com resolving to 193.23.181[.]155.*

One second later the host makes its first request to that address.

## The Command and Control Channel

From 22:41:16 to the end of the capture the host runs a fixed loop: a GET for `/ele/config.jpg`, then roughly one second later a POST to `/ele/gate.php`, then silence for about a minute before repeating.

![gate.php POST interval](./media/defensive06-beacon-interval.png)

*Sixteen POSTs to /ele/gate.php at approximately 62 second intervals.*

The regularity is the signal. Human browsing is bursty and hits varied destinations with varied sizes. A host issuing the same two requests to the same two URIs on a fixed timer for sixteen consecutive minutes is automated.

Following one of the transactions shows what is moving.

![Standard check-in](./media/defensive06-standard-checkin-post.png)

*A routine check-in. 346 byte encrypted request body, server response over Apache with PHP 5.4.44.*

The POST body is binary with no readable structure. The response headers give a server fingerprint, Apache with PHP/5.4.44, and a `Date` header of 22:41:25 GMT that matches the packet timestamp exactly, independently confirming the capture is in UTC.

Exporting the HTTP objects and examining the uploaded bodies inside the VM:

![gate.php body analysis](./media/defensive06-gate-file-strings-hash.png)

*file reports no recognizable format and strings returns nothing readable. The object shown is the largest of the uploads at 8,480 bytes, covered below.*

![Entropy measurement](./media/defensive06-gate-entropy.png)

*Entropy of 7.979 bits per byte against a theoretical maximum of 8.*

Plain text measures around 4 to 5 bits per byte and compiled code around 6. A measurement this close to the ceiling means encrypted or compressed content with no remaining structure to recover. The server's replies are equally unreadable, visible as binary following the chunked transfer header in the stream above, so the channel is obfuscated in both directions.

## The Image That Is Not Just an Image

The repeatedly fetched `config.jpg` is 81 kB on every fetch. A browser caches an image. Something re-requesting the same file every 62 seconds is not rendering a page.

![config.jpg examined](./media/defensive06-config-jpg-hex.png)

*file identifies a genuine JPEG. The tail shows base64 text ending in padding immediately before the ffd9 end-of-image marker.*

The file is a real photograph with complete and legitimate camera metadata: a Canon EOS 5D Mark II capture from 2011, processed in Lightroom, with intact IPTC attribution and an embedded sRGB color profile. It opens normally in an image viewer. But the last twenty lines before the end-of-image marker are continuous base64 text terminating in `==` padding, and running exiftool accounts for none of it. No XMP, IPTC, Photoshop or ICC segment holds that content. The 16 malformed packets Wireshark flagged in the protocol hierarchy were the dissector reacting to this structural anomaly.

Hashing the carved file and checking the hash:

![VirusTotal on the config hash](./media/defensive06-virustotal-config-hash.png)

*SHA256 3359a8d1..., popular threat label trojan.zbotjpgcfg, detected by TrendMicro as TROJ_ZBOTJPGCFG.SM.*

The vendor label names the technique directly. This is a Zeus (ZBOT) configuration file carried inside a JPEG. The two of sixty detection rate is itself the finding: a file that opens as a valid photograph, carries authentic camera metadata, and is served with a normal image extension defeats static reputation checks. The behavior around it is what exposes it.

The domain serving it:

![VirusTotal on the domain](./media/defensive06-virustotal-domain.png)

*classicalbitu[.]com, 6 of 89 vendors flagging it as malicious.*

## The Outlier

Fifteen of the sixteen check-ins upload a uniform 346 bytes. One does not.

![Oversized POST](./media/defensive06-oversized-post-exfil.png)

*22:43:28. A POST carrying an 8,480 byte body, roughly 25 times the normal check-in.*

The direction is worth stating plainly, because it is easy to misread a large transfer as a download. This is a POST, and the size is in the request. The data moved from the infected host to the attacker's server. This is the object measured above at 7.979 bits per byte. Its contents cannot be recovered.

What preceded it is the relevant part. Thirty-eight seconds earlier, at 22:42:50, the host issued three NetBIOS name queries for CITIBANK, visible in the NBNS capture above. Those queries sit among otherwise routine WPAD lookups and are the only bank-related activity in the file. Observed: bank-related name resolution at 22:42:50, followed by an encrypted upload 25 times larger than baseline at 22:43:28. Analysis: this ordering is consistent with a credential stealer capturing data during banking activity and reporting it upstream. The upload is encrypted, so the correlation is timing and volume based.

## The Two Yandex Requests

The remaining traffic resolves quickly.

![Yandex request](./media/defensive06-yandex-connectivity-check.png)

*GET to yandex.ru at 22:41:23, answered with a 302 redirect.*

These carry the same User-Agent as the C2 requests, meaning the same process issued both. They land seven seconds after the first config retrieval and are followed immediately by the first check-in. This is the implant verifying it has working internet access by reaching a large, reliably available site, the same logic behind Windows' own NCSI check. It is startup behavior, not infection traffic, and yandex.ru is not treated as an indicator here.

## Root Cause

The workstation was infected with Zeus (ZBOT), a banking trojan. The capture shows the infection fully operational: within seconds of network connectivity it resolved and reached its command-and-control server at classicalbitu[.]com, verified internet access against yandex.ru, retrieved an encrypted configuration concealed inside a JPEG, and ran an encrypted check-in loop. The NetBIOS query for CITIBANK followed by an oversized encrypted upload shows the trojan doing what it exists to do, capturing data during banking activity and reporting it to the operator.

Zeus of this period was distributed primarily through malicious email attachments and exploit-kit drive-by downloads, which is the likely route onto this host. Establishing which of the two, and the specific message or site involved, would require host-based artifacts such as browser history, email client data, prefetch and registry run keys.

## Indicators of Compromise

| Type | Value | Role |
|---|---|---|
| Domain | `classicalbitu[.]com` | Zeus C2, VT 6/89 |
| IP | `193.23.181[.]155` | C2 host, Apache / PHP 5.4.44 |
| URL | `hxxp://classicalbitu[.]com/ele/config.jpg` | Encrypted configuration in a valid JPEG |
| URL | `hxxp://classicalbitu[.]com/ele/gate.php` | Check-in and exfiltration endpoint |
| SHA256 | `3359a8d15eb9de2e6a9d5a91182ae41f7dda053e78f5e9d3b715ab28afa57ea1` | config.jpg, TROJ_ZBOTJPGCFG.SM |
| User-Agent | `Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; Trident/4.0; SLCC2; ...)` | Used for C2 and connectivity check |

Victim: 10.54.112.205, 00:50:8b:01:db:2f, PENDJIEK-PC, Windows 7, workgroup WORKGROUP.

## Tools and Commands

Wireshark display filters

```
dhcp
nbns
dns
dns.a
kerberos.CNameString
http.request
http.user_agent
http.request and http.request.uri contains "gate.php"
http.request and ip.dst != 193.23.181.155
http.request and ip.addr == 5.255.255.5
```

File identification and hashing, run inside the isolated VM

```
file config.jpg
file gate.php
xxd config.jpg | head -5
xxd config.jpg | tail -20
xxd gate.php | head -10
strings -n 6 gate.php | head -40
strings -n 40 config.jpg | tail -40
exiftool config.jpg
sha256sum config.jpg
sha256sum gate.php
```

Entropy measurement

```
ent gate.php
```

## Takeaways

The detection opportunities here are all behavioral, and none of them require knowing what Zeus is.

Fixed-interval outbound requests are the strongest signal in this capture. Sixteen POSTs at roughly 62 second spacing to a single URI, with no referring page and no accompanying page assets, is not something a browser produces. Beacon periodicity detection catches this without any prior knowledge of the family, the domain or the payload, which matters because the domain scored 6 of 89 and the payload file scored 2 of 60. Reputation alone would have missed it.

Repeated retrieval of an identical static object deserves attention on its own. An 81 kB image fetched at the same size every minute, with no HTML, CSS or other resources requested alongside it, is a malformed page load at best. Here it was a configuration channel.

Protocol anomalies are worth reading rather than dismissing. Wireshark flagged 16 malformed JPEG packets in the protocol hierarchy before any file was carved, and that was the dissector correctly reacting to data embedded in the image. A file type that parses as valid but fails structural validation is a useful hunting pivot.

For prevention, egress filtering and DNS filtering would have broken this chain at the first lookup, since the host had no legitimate reason to resolve or reach that domain. Outbound POST volume monitoring would have flagged the 8,480 byte upload against a 346 byte baseline. Neither requires identifying the malware first. Both work on the shape of the traffic rather than its contents, which is the only option available once the channel is encrypted, as it was here in both directions.