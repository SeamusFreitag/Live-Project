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

- [Install Kali Linux VM](./setup/1-kali-vm/) - Kali build on KVM/virt-manager and a clean-baseline snapshot
- [Create Juice Shop App](./setup/2-juice-shop/) - OWASP Juice Shop deployed and run via Docker
- [Intro - Burp Suite](./setup/3-burp-suite/) - intercepting proxy configured, traffic capture and request modification

## Offensive

- [Admin Log In](./offensive/1.1-admin-log-in/) - SQL injection auth bypass
- [User Log In](./offensive/1.2-user-log-in/) - Targeted SQLi after user enumeration
- [Reset Admin Password](./offensive/2-reset-admin-password/) - Burp Intruder brute force
- [Admin Access](./offensive/3-admin-access/) - Mass assignment on user registration
- [Admin Page](./offensive/4-admin-page/) - Client-side route discovery + IDOR on baskets + admin panel abuse
- [CAPTCHA Exploit](./offensive/5-captcha-exploit/) - Reusable CAPTCHA + no rate limiting, Burp Intruder flood
- [Access Secured Documents](./offensive/6.1-access-secured-documents/) - Exposed /ftp/ directory, confidential file access
- [Download Secured Documents](./offensive/6.2-download-secured-documents/) - Poison null byte extension-filter bypass
- [HTTP Requests](./offensive/7-http-requests/) - Request tampering: basket IDOR + zero-star feedback via improper input validation

## Defensive

- [Wireshark Intro](./defensive/00-wireshark-intro/) - Okay-Boomer pcap analysis: host/OS fingerprinting, PE file carving, Trickbot confirmed via VirusTotal
- [Malware Traffic](./defensive/01-malware-traffic/) - Exploit kit chain reconstruction: Flash exploit + hidden iframe, executable disguised as text/html, ransomware confirmed via VirusTotal