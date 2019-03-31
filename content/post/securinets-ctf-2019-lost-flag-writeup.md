---
title: "Securinets CTF 2k19 Lost Flag Writeup"
author: "CHERIEF Yassine"
date: 2019-03-26T17:37:37+02:00
subtitle: "Exploiting VCS (Version Control System)"
tags: ["writeup", "ctf", "security", "securinets-ctf", "web"]
---



## Description

| **Category**  | **Points** | **Solves** |
|-----------|--------|--------|
| Forensics | 994    | 19     |
<br>
<br>
**`Description`** : Help me get back my flag !  
**`URL`**: https://web8.ctfsecurinets.com/  


After visiting the link above, we get a login page.We can login using `admin/admin`, but unfortunately the flag is deleted.

Then I started searching in the website for any other infos, but with no success. the admin of the challenge (**`Tr'GFx`**) said that `bruteforcing` directories is enough to solve the challenge, so I used [dirsearch.py](https://github.com/maurosoria/dirsearch) to search for directories.  
  
Using dirsearch I found `/.bzr/README`, which means there is a Bazaar repository on the website, Bazaar is like `git`.  
  
> Bazaar is a version control system that helps you track project history over time and to collaborate easily with others.
  
  
We can now use `bzr branch` utility to clone the remote repository.

```
bzr branch -Ossl.cert_reqs=none https://web8.ctfsecurinets.com/
```

`for cloning a bzr repository use bzr branch and not bzr clone`  
  

After cloning the repo, we can see the log by executing **`bzr log`**

![bzr_log_lost_flag](https://res.cloudinary.com/https-omega-coder-github-io/image/upload/v1554052445/bzr_log_lost_flag.png)

  


We can see that the flag was deleted in the revision number 2, so let's revert back to rev 1.  
We use the command `bzr revert -r1` to revert back to the revision 1.  

  
Now we can see the file `flag.php` which has the flag in it.

`FLAG: Securinets{BzzzzzzzzZzzzzzzzzzZrR_roCk$}`.















