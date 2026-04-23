# Evidence Notes

This folder contains visual proof for the Samurai machine walkthrough.

## File Index

- `01-nmap-scan.png` — full TCP scan showing exposed services (SSH and HTTP)
- `02-joomla-admin.png` — Joomla administrator login page confirming CMS attack surface
- `03-cve-config-leak.png` — CVE-2023-23752 config leak exposing database credentials
- `04-user-enum-api.png` — Joomla API user enumeration showing `Miyamoto` as Super User
- `05-template-rce.png` — template editor showing PHP payload injection for command execution
- `06-reverse-shell.png` — reverse shell connection as `www-data` plus `sudo -l` privilege escalation clue
- `07-root-flag.png` — final root flag proof
- `08-user-flag.png` — user flag proof

## Notes

Screenshots were organized to match the attack path from reconnaissance to full compromise.
Sensitive values were only included when necessary to explain the exploitation chain.

