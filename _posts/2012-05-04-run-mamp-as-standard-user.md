---
layout: post
title: "Run MAMP as standard user"
date: 2012-05-04
tags: [enterprise-it, unprivileged-users]
---

At work we are starting to roll out Macbook Pro's to many of our developers. At the same time, we're looking to keep security policies in place and have people operate as unprivileged users. We're finding this can be more than a bit difficult. There's nothing more lazy than instructions to 'run as root, turn off any firewall or selinux.' Why should your application require me to invalidate all sensible security practices? Fail.

The problem today was running MAMP with a standard Mac OS user. More than a few posts and responses on MAMP's forums claim this can't be done. I call bullshit. So lets get to it.

1. First install MAMP as an admin user.

2. Then create a user group that will hold any MAMP users. Mine's called **mamp**.

3. Using Terminal, change all files in the MAMP folder to be owned by this new group and give the group read, write, and execute privileges:

```
chown -R :mamp /Applications/MAMP
chmod -R g+rwx /Applications/MAMP
```

4. Edit the httpd.conf file to use the logged in user and mamp group. In `/Applications/MAMP/conf/apache/httpd.conf` look for the line starting with **User**. Comment it out and change the group to your mamp group name:

```
#User severin
Group mamp
```

This should do the trick. Starting or stopping services via the MAMP application will still ask for admin user authentication but you can just cancel the prompt (twice) and the services should start fine.

I've posted to the MAMP forums asking exactly what the application is trying to do that requires admin privileges. Previous posts imply they need to use low numbered, privileged port (not the case) or edit the hosts file (doesn't seem to need to). If it tries to run the services as a specified user, it would require root but that's exactly what we don't want to do. If I see an answer, I will follow up here.

Additionally, I had to edit the httpd.conf to fix a path reference to PHP 5.3.5. I assume this is just a config bug in the 2.0.5 release.

```
LoadModule php5_module /Applications/MAMP/bin/php/php5.3.6/modules/libphp5.so
```
