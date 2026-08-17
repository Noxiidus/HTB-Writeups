# Hack The Box — Writeups

Personal writeups for retired Hack The Box machines and completion summaries for
Pro Labs. Flags are redacted per the HTB content policy — these notes focus on the
methodology and the vulnerabilities, not on spoiling the flags.

## Pro Labs

| Lab | Type | Key techniques |
|-----|------|----------------|
| [Mythical](Mythical/) | Mini Pro Lab — AD | Mythic C2 · ADCS abuse · MSSQL lateral movement · Token impersonation · WDigest credential extraction |

## Machines

| Machine | OS | Difficulty | Key techniques |
|---------|----|-----------|----------------|
| [Nexus](Nexus/) | Linux | Medium | Git history credential leak · Krayin CRM file-upload RCE · SSH password reuse · Gitea template-sync path traversal (root) |
| [Orion](Orion/) | Linux | Easy | Craft CMS pre-auth RCE (CVE-2025-32432) · `.env` DB creds · bcrypt crack + password reuse · telnetd `-f` auth bypass (CVE-2026-24061, root) |
| [Abducted](Abducted/) | Linux | Medium | Samba print-spooler injection (CVE-2026-4480) · rclone reveal + password reuse · wide-links symlink SMB pivot · operators systemd drop-in (root) |

---

*For educational and defensive-security purposes. All testing was performed against
machines on my own Hack The Box account over the official VPN.*
