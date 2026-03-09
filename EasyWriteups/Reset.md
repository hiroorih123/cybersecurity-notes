# Reset Writeup

## 1. Enumeration

### Nmap

'''bash
sudo nmap -p- -sC -sV --min-rate 1000 10.129.3.115
'''
5 tcp port is open, one of them is http port
-> visit via browser and we can find admin login page.

### Web Enumeration
there's "Forgot Password" link. that allow us to change password.
try "user" as username, but username not found.
try "admin" as username, we got the message means that we successfully changed the password!!
using developer tool of browser, we found new_password in response body!

on admin dashboard, we see the content of log file.
by using burpsuite on admin dashboard, we see the parapeter "file" specifies what file is to be read.
we already know the running service on the target is apache, by nmap output.
-> google search abount apache log file path
Let's try to see log file /var/log/apache2/access.log, by specify the file parameter "file".
Now, we can see the content of access.log, and HTTP header "User-Agent" is included in access.log
->there is Local File Inclusion vulnerability on the target, and probably we can use Log Poisoning.

Local File Inclusion (LFI) is a web vulnerability where an attacker manipulates input parameters to force an application to include files stored on the server

Condition for Log Poisoning to RCE
1. The application has a Local File Inclusion (LFI) vulnerability.
2. The attacker can read a log file (e.g., Apache access.log) via LFI.
3. User input (e.g., User-Agent) is written into the log file.
4. The log file is included by the application using functions like include() so that PHP code is executed.

we know 1,2,3 is met, so try 4.


---

## 2. Initial Foothold

### Vulnerability Identification

### Exploitation

---

## 3. User Flag

### Post-Exploitation Enumeration

### Credential Discovery

### Lateral Movement (if applicable)

---

## 4. Privilege Escalation

### Privilege Check

### Vulnerability / Misconfiguration

### Exploitation

---

## What We Learned
