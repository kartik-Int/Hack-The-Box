# Hack The Box — Writeups

A collection of Hack The Box machine writeups for learning and reference.

---

## Machines

| # | Machine | OS | Difficulty | Key Techniques | User Flag | Root Flag |
|---|---------|-----|-----------|----------------|-----------|-----------|
| 1 | [Enigma](./enigma.md) | Linux | Medium | NFS Enum, IMAP Enum, Zip Filename Injection, OliveTin API Command Injection | ✅ | ✅ |
| 2 | [Checkpoint](./checkpoint.md) | Windows | Hard | AD Deleted Object Restore, Malicious VSIX, dMSA Abuse, VMware Memory Dump, Pass-the-Hash | ✅ | ✅ |
| 3 | [MakeSense](./makesense.md) | Linux | Medium | WordPress Enumeration, Stored XSS, Admin Account Creation, Theme Editor RCE, OCR Service Exploitation | ✅ | ✅ |
| 4 | [Paperwork](./paperwork.md) | Linux | Medium | LPD Command Injection, Jetdirect Path Traversal, Unix Socket FD Passing, Printer Service Exploitation | ✅ | ✅
| 5 | [Bedside](./bedside.md) | Linux | Easy | PDF Pickle Deserialization (CVE-2025-64512), Path Traversal, SSH Key Extraction, PyTorch Checkpoint RCE, SUID Bash Privilege Escalation | ✅ | ✅ |

---
---

# Techniques Index

## Reconnaissance
- Full TCP port scan with Nmap (`-Pn -p- --min-rate 5000`)
- Service version scan (`-sC -sV`)
- WordPress enumeration with WPScan
- NFS enumeration (`nfs-showmount`, `nfs-ls`, `nfs-statfs`)
- LDAP enumeration with `bloodyAD`
- IMAP enumeration with `curl imaps://`
- Upload directory enumeration
- - Virtual host enumeration with ffuf
- Printer service enumeration (LPD/Jetdirect)

---

## Initial Access / Foothold
- WordPress username enumeration
- Audio file credential discovery
- Stored XSS
- Administrator account creation via XSS
- Theme Editor PHP webshell
- PHP reverse shell
- NFS share mounting
- IMAP mailbox enumeration
- Password reuse
- Zip filename OS command injection
- Malicious VS Code Extension (VSIX)
- PDF malicious font encoding (pdfminer.six)
- Pickle object deserialization RCE
- Docker container reverse shell
- PyTorch checkpoint manipulation
- LPD (Line Printer Daemon) command injection
- Unsanitized subprocess parameters
- Printer job queue exploitation

---

## Privilege Escalation — Linux
- WordPress configuration credential extraction
- Local service enumeration
- SSH port forwarding
- OCR service abuse
- PHP code generation through OCR
- OliveTin API command injection
- Hashcat bcrypt cracking (`-m 3200`)
- SUID `/bin/bash`
- User lateral movement with `su`
- SSH private key extraction from LFI
- SSH authentication to main system
- Jetdirect path traversal
- Internal service communication exploitation

---

## Privilege Escalation — Windows / Active Directory
- Deleted AD object restoration
- Kerberos TGT retrieval
- dMSA badSuccessor abuse
- Pass-the-Hash
- VMware memory dump credential extraction
- WinRM authentication

---

# Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning & service enumeration |
| wpscan | WordPress enumeration |
| curl | HTTP / IMAP interaction |
| ssh | Remote shell & port forwarding |
| hashcat | Password cracking |
| ImageMagick | PHP image generation |
| base64 | Payload encoding |
| bloodyAD | AD exploitation |
| NetExec (nxc) | SMB authentication |
| impacket-getTGT | Kerberos |
| evil-winrm | WinRM shell |
| smbclient | SMB interaction |
| VMkatz | VMware credential extraction |
| faketime | Kerberos clock synchronization |
| python3 | Custom exploit scripts | ffuf | Virtual host enumeration |
| curl (with --path-as-is) | Path traversal exploitation |
| python3 (pickle, zipfile, gzip) | Exploit payload generation
| LPD printer protocol exploitation


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

## MakeSense (Linux)
```

WordPress Enumeration → Audio File (jake:CleanLightNiceSmooth4923)
→ Stored XSS → Administrator Account Creation (pwned)
→ Theme Editor RCE → wp-config.php (walter:JbhHDAEgXvri3!)
→ SSH as walter → Port Forward (OCR Service :8001)
→ OCR PHP Payload Generation → Save as PHP → Execute as Root → Root
```

### Paperwork (Linux)
```

Port 1515 (LPD Printer) → Download vulnerable server source (port 1337)
→ Identify command injection in job_name parameter
→ Craft malicious LPD control file with shell metacharacters
→ Send to queue: archive_intake → subprocess.Popen executes injected command
→ RCE as printer daemon user → Jetdirect path traversal → Extract internal service credentials → Unix socket FD passing
→ Escalate to root via privileged printer service → root shell
```

### Bedside (Linux)
```
research.bedside.htb (pdfminer.six) → Upload malicious PDF + pickle payload
→ Pickle deserialization → datawrangler shell (Docker)
→ Port 3000 path traversal (/pr/x/y@99/../../home/developer/.ssh/id_rsa)
→ Extract SSH key → SSH as developer
→ Discover sudo: /usr/bin/python3 /opt/trainer/bedside_trainer.py (NOPASSWD)
→ Create malicious PyTorch checkpoint (.pt file with pickled RCE)
→ Upload checkpoint → Trigger via sudo → PyTorch deserialization
→ SUID bash creation (chmod 4755) → /home/developer/rootbash2 -p
→ Root shell (euid=0) → Read /root/root.txt
```

---

## Notes

- All machines are from [Hack The Box](https://www.hackthebox.com) and are intended for legal, educational use only
- Writeups are published after machines are retired or with explicit permission
- Flags shown are from personal playthroughs and may differ on reset machines

---

*Happy hacking and keep learning!*
