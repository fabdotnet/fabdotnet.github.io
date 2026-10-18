+++
title = "Published CVEs"
date = 2024-07-16T00:00:00Z
draft = false
layout = "single"
ShowToc = false
hideMeta = true
+++

<div style="display:flex;gap:1.5rem;align-items:center;flex-wrap:wrap;margin-bottom:1rem;">
  <div style="flex:1 1 320px;min-width:280px;">
    <p>Security research and coordinated vulnerability disclosures, most recent first.
    Each entry links to a full writeup with root-cause analysis and proof of concept.</p>
  </div>
  <img src="/img/vulnerabilities-everywhere.jpg" alt="Toy Story meme: Vulnerabilities, vulnerabilities everywhere" width="300" loading="lazy" style="flex:0 0 auto;max-width:100%;height:auto;border-radius:8px;">
</div>

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
