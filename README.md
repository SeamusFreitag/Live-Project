# Live Project

The Tech Academy's Live Project portion of the cybersecurity bootcamp. Offensive and defensive security work (OWASP Juice Shop, Burp Suite, Wireshark) with incident report writeups.

## Overview

This repository tracks work on The Tech Academy's Live Project: applying offensive and defensive security techniques in a simulated professional environment. Work is organized into stories (offensive, defensive, and setup), each with its own incident report documenting what was investigated, how, what was found, and what the fix or takeaway is.

This repository is updated as stories are completed, not assembled at the end.

## Core Technologies

- **Environment:** Kali Linux, KVM/virt-manager
- **Offensive:** OWASP Juice Shop, Burp Suite, FoxyProxy
- **Defensive:** Wireshark, VirusTotal
- **Research:** ExploitDB, GTFOBins, Rapid7, CVE Details, CIS Security

## Structure

- `setup/` - Offensive Setup stories (Kali VM, Juice Shop, Burp Suite)
- `offensive/` - web application security stories against OWASP Juice Shop
- `defensive/` - network forensics and malware investigation stories

## Setup

- [Offensive Setup #1: Install Kali Linux VM](./setup/1-kali-vm/)
- [Offensive Setup #2: Create Juice Shop App](./setup/2-juice-shop/)
- [Offensive Setup #3: Intro - Burp Suite](./setup/3-burp-suite/)

## Offensive

- [Offensive #1.1: Admin Log In](./offensive/1.1-admin-log-in/) - SQL injection auth bypass
- [Offensive #1.2: User Log In](./offensive/1.2-user-log-in/) - Targeted SQLi after user enumeration
- [Offensive #2: Reset Admin Password](./offensive/2-reset-admin-password/) - Burp Intruder brute force
- [Offensive #3: Admin Access](./offensive/3-admin-access/) - Mass assignment on user registration
- [Offensive #4: Admin Page](./offensive/4-admin-page/) - Client-side route discovery + IDOR on baskets + admin panel abuse
- [Offensive #5: CAPTCHA Exploit](./offensive/5-captcha-exploit/) - Reusable CAPTCHA + no rate limiting, Burp Intruder flood
- [Offensive #6.1: Access Secured Documents](./offensive/6.1-access-secured-documents/) - Exposed /ftp/ directory, confidential file access
- [Offensive #6.2: Download Secured Documents](./offensive/6.2-download-secured-documents/) - Poison null byte extension-filter bypass
- [Offensive #7: HTTP Requests](./offensive/7-http-requests/) - Request tampering: basket IDOR + zero-star feedback via improper input validation

## Defensive

- [Wireshark Intro](./defensive/00-wireshark-intro/) - Okay-Boomer pcap analysis: host/OS fingerprinting, PE file carving, Trickbot confirmed via VirusTotal