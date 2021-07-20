---
title: 'Remote-call a bash function'
published: true
date: '12-12-2017 00:00'
taxonomy:
    tag:
        - Short
        - Bash
author: 'Christian Beneke'
---

I've just recently discovered the bash builtin `typeset -f`. This command will print the definition of all (or a given) function(s).

# But why do I write about this?
Thats the neat part: Lets say you have some application-servers and an admin-server, but don't want to write a lot of

```
ssh $user@$server -o 'aLotOfOptions' --somemoreparameters command
```

lines in your admin-shell-script. Therefor I used to put (small) shell-scripts on my application-servers via automated deployment and call this script on all of them from my admin server. This creates the small overhead to not only have to manage one script, but n many.

# The solution
With `typeset -f` you can call a function on your application-servers, which is defined in the script. I just put the small routine into a (bash-) function, let `typeset -f` "re-define" this function on the application-server and use it afterwards:

```
#!/usr/bin/env bash

. /usr/local/etc/config.sh

myfunction() {
  [...]
}

for server in ${applicationServers}; do
  ssh -o 'aLotOfOptions' --someparameters user@${server} "$(typeset -f myfunction); myfunction"
done
```

Now I can work on the function without having to check each of the applications-servers, if the script is up to date (or re-run ansible/puppet/etc on all servers).