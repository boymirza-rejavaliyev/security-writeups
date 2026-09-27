# Security Write-ups & Methodology

A collection of my penetration testing notes, HackTheBox / CTF write-ups, and reusable
methodology for offensive security engagements.

> ⚠️ Write-ups are published for **educational purposes** and only for **retired** machines
> or platforms that permit public write-ups. Never test systems you are not authorized to test.

## 📂 Structure

```
.
├── methodology/               # reusable checklists & workflows
│   ├── web-app-pentest.md
│   └── linux-privesc.md
├── hackthebox/                # HTB machine write-ups (retired only)
│   ├── _template.md
│   └── lame.md
├── wordpress-vuln-research/   # original WordPress plugin vulnerability research
│   ├── nd-booking.md
│   ├── booking-package.md
│   └── media-library-organizer.md
└── ctf/                       # CTF challenge write-ups
```

## 🧭 Methodology

- [Web Application Pentest Checklist](methodology/web-app-pentest.md)
- [Linux Privilege Escalation Checklist](methodology/linux-privesc.md)

## 📝 Write-ups

| Platform | Machine / Challenge | Difficulty | Topics |
|----------|---------------------|------------|--------|
| HTB | [Lame](hackthebox/lame.md) | Easy | Samba `usermap_script` RCE (CVE-2007-2447) |

_More retired-machine and CTF write-ups coming as I complete them._

## 🔎 Original WordPress Plugin Vulnerability Research

Independent research into WordPress plugins, done in isolated local Docker environments and
reported to vendors through responsible disclosure. Some of these did not clear the install-count
or category thresholds required by Patchstack / Wordfence for a CVE, but every finding below was
independently confirmed with a working PoC.

| Plugin | Vulnerability | Class | CVE status |
|---|---|---|---|
| Shared Files ≤ 1.7.71 | Password-protected file download bypass | Broken Access Control | Reported to Patchstack — **in review** (write-up to follow once patched) |
| [ND Booking ≤ 3.8](wordpress-vuln-research/nd-booking.md) | Unauthenticated permanent WooCommerce price override | CWE-20 | Reported, no CVE (below Patchstack mVDP / Wordfence install threshold) |
| [Booking Package ≤ 1.7.28](wordpress-vuln-research/booking-package.md) | Predictable cancellation token → unauthenticated booking cancellation | CWE-330 | Reported, no CVE (below Wordfence install threshold) |
| [Media Library Organizer ≤ 2.1.4](wordpress-vuln-research/media-library-organizer.md) | Authenticated path traversal → arbitrary `.zip` file write | CWE-22 | Reported, no CVE (extension not fully attacker-controlled) |

## 🏷️ About

Maintained by **Boymirza Rejavaliyev** — Penetration Tester | Offensive Security.
Certs: CRTA, CRT-COI, CJCA, HPTC. Find me on [LinkedIn](https://www.linkedin.com/in/boymirza-rejavaliyev/).
