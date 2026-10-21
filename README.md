# htb-writeups

Penetration testing walkthroughs and CTF solutions by **[pWnR1se](https://github.com/pWnR1se)**.

Manual exploitation, full attack chains, and the *why* behind each step — written the way I'd hand a report to the next operator.

---

## ⚠️ Scope & disclosure policy

This repository publishes **only** content that is allowed to be public:

- ✅ **Retired** HTB machines — a machine is included **only after** its HTB page shows the **`Retired`** badge.
- ✅ Completed / finished CTFs where the organizer permits writeups.
- ✅ Bug-bounty findings **only** after they are resolved *and* the program has authorized public disclosure.

**Never published here:** active HTB machines, live/ongoing CTF flags, and any undisclosed bug-bounty vulnerability. This respects the [HTB Terms of Service](https://www.hackthebox.com/tos) and every program's disclosure rules.

---

## 📁 Structure

```
htb-writeups/
├── _template/           # writeup template — copy this to start a new one
├── htb/
│   ├── machines/        # retired HTB machine walkthroughs
│   └── challenges/       # HTB challenge writeups
├── ctf/                 # completed CTF walkthroughs
└── bug-bounty/          # disclosed-only, methodology & sanitized reports
```

To start a new writeup: copy `_template/TEMPLATE.md` into the right folder, rename it (e.g. `htb/machines/previse.md`), and fill it in.

---

## 🗂️ Index

### HTB — Machines (retired)

| Machine | OS | Difficulty | Key techniques | Writeup |
|---|---|---|---|---|
| Previse | Linux | Medium | EAR, command injection, PATH hijack | [previse.md](htb/machines/previse.md) |
| Sau | Linux | Easy | SSRF, Maltrail RCE, pager escape | [sau.md](htb/machines/sau.md) |
| GoodGames | Linux | Easy | SQLi, SSTI, Docker escape | [goodgames.md](htb/machines/goodgames.md) |
| Devvortex | Linux | Medium | Joomla CVE, sudo apport-cli | [devvortex.md](htb/machines/devvortex.md) |
| Paper | Linux | Easy | WordPress draft leak, path traversal, Polkit | [paper.md](htb/machines/paper.md) |
| MonitorsFour | Windows | Easy | IDOR, Cacti CVE, Docker Desktop API | [monitorsfour.md](htb/machines/monitorsfour.md) |
| Responder (SP) | Windows | Very Easy | LFI/RFI, NTLM capture, WinRM | [responder.md](htb/machines/responder.md) |
| Three (SP) | Linux | Very Easy | vhost, open S3 bucket, webshell | [three.md](htb/machines/three.md) |
| Funnel (SP) | Linux | Very Easy | anonymous FTP, SSH port forward | [funnel.md](htb/machines/funnel.md) |

### HTB — Challenges

| Challenge | Category | Writeup |
|---|---|---|
| _add here_ | | |

### CTF

| Event | Task | Category | Writeup |
|---|---|---|---|
| VulnHub — HarryPotter | Aragog (1/3) | boot2root | [ctf/vulnhub-harrypotter/aragog.md](ctf/vulnhub-harrypotter/aragog.md) |

---

## 🏷️ Legend

`web` · `network` · `ad` · `privesc` · `pivoting` · `crypto` · `blockchain` · `forensics` · `stego` · `rev`

---

<sub>All techniques shown were performed in authorized lab or engagement scope. For education and defense.</sub>
