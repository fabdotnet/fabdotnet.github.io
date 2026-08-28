+++
title = "Published CVEs"
date = 2024-07-16T00:00:00Z
draft = false
layout = "single"
ShowToc = false
hideMeta = true
+++

Security research and coordinated vulnerability disclosures, most recent first.
Each entry links to a full writeup with root-cause analysis and proof of concept.

## IBM

| CVE | Class | Component | Fixed in |
|-----|-------|-----------|----------|
| [CVE-2024-28796](/posts/cve-2024-28796-stored-xss-ibm-clearquest/) | Stored XSS | Rational ClearQuest CQWeb | 9.1.0.7 |

## pfSense (Netgate)

A series of three findings in the pfSense WebGUI, credited in Netgate security
advisories.

| CVE | Class | Component | Fixed in |
|-----|-------|-----------|----------|
| [CVE-2023-27100](/posts/cve-2023-27100-pfsense-bruteforce-bypass/) | Anti-brute-force bypass | WebGUI / sshguard | CE 2.7.0 / Plus 23.01 |
| [CVE-2022-29273](/posts/cve-2022-29273-pfsense-stored-xss-aliases/) | Stored XSS | `firewall_aliases.php` (URL Table) | CE 2.7.0 / Plus 22.05 |
| [CVE-2022-23993](/posts/cve-2022-23993-pfsense-reflected-xss-pkg/) | Reflected XSS | `pkg.php` (`pkg_filter`) | CE 2.6.0 / Plus 22.01 |
