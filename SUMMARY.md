# Table of contents

* [Checkpoint](README.md)
* [Enigma](enigma.md)


# Hack The Box — Writeups

A collection of Hack The Box machine writeups for learning and reference.

---

## Machines

| # | Machine | OS | Difficulty | Key Techniques | User Flag | Root Flag |
|---|---------|-----|-----------|----------------|-----------|-----------|
| 1 | [Enigma](./enigma.md) | Linux | Medium | NFS Enum, IMAP Enum, Zip Filename Injection, OliveTin API Command Injection | ✅ | ✅ |
| 2 | [Checkpoint](./checkpoint.md) | Windows | Hard | AD Deleted Object Restore, Malicious VSIX, dMSA Abuse, VMware Memory Dump, Pass-the-Hash | ✅ | ✅ |

---

## Techniques Index

### Reconnaissance
- Full TCP port scan with Nmap (`-Pn -p- --min-rate 5000`)
- Service version scan (`-sC -sV`)
- NFS enumeration (`nfs-showmount`, `nfs-ls`, `nfs-statfs` scripts)
- LDAP enumeration with `bloodyAD`
- IMAP enumeration with `curl imaps://`

### Initial Access / Foothold
- NFS share mounting and sensitive file extraction
- IMAP mailbox reading for credential discovery
- Password reuse across accounts
- Zip filename OS command injection (OpenSTAManager)
- PHP webshell upload and reverse shell
- Malicious VS Code extension (VSIX) via writable SMB share

### Privilege Escalation — Linux
- Bcrypt hash cracking with Hashcat (`-m 3200`) + rockyou.txt
- `su` lateral movement between users
- OliveTin API guest execution misconfiguration
- `password`-type argument injection → shell command injection
- SUID bit set on `/bin/bash` for root shell

### Privilege Escalation — Windows / Active Directory
- Deleted AD object restoration with `bloodyAD`
- Kerberos TGT retrieval with `impacket-getTGT`
- dMSA badSuccessor abuse for service account impersonation
- WinRM access with `evil-winrm` (Pass-the-Hash)
- VMware memory snapshot credential extraction with VMkatz
- Pass-the-Hash to Administrator

### Tools Used
| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning and service enumeration |
| `netcat (nc)` | Reverse shell listener |
| `curl` | HTTP/IMAP interaction |
| `hashcat` | Offline password cracking |
| `bloodyAD` | AD LDAP enumeration and exploitation |
| `nxc (NetExec)` | SMB authentication and share enumeration |
| `impacket-getTGT` | Kerberos TGT retrieval |
| `evil-winrm` | WinRM shell with Pass-the-Hash |
| `smbclient` | SMB file upload/download |
| `VMkatz` | Credential extraction from VMware memory snapshots |
| `faketime` | Clock sync for Kerberos attacks |
| `python3` | Custom exploit scripts (zip injection, base64 payloads) |

---

## Attack Chains

### Enigma (Linux)
```
NFS Mount → PDF (kevin:Enigma2024!) → IMAP (sarah:Enigma2024!)
→ Sarah's Email (OpenSTAManager creds) → Zip Filename Injection
→ PHP Webshell → www-data shell → DB Dump → Hash Crack (haris:bestfriends)
→ su haris → OliveTin API (localhost:1337) → SUID bash → root
```

### Checkpoint (Windows AD)
```
alex.turner creds → LDAP writable objects → Restore mark.davies
→ DevDrop SMB share → Malicious VSIX → ryan.brooks shell
→ dMSA badSuccessor → svc_deploy (WinRM) → VMBackups share
→ VMkatz memory dump → Administrator NTLM hash → Pass-the-Hash → root
```

---

## Notes

- All machines are from [Hack The Box](https://www.hackthebox.com) and are intended for legal, educational use only
- Writeups are published after machines are retired or with explicit permission
- Flags shown are from personal playthroughs and may differ on reset machines

---

*Happy hacking and keep learning!*
