---
title: "Faster docker contexts"
date: 2023-07-17T03:53:29+02:00
description: "Speeding up docker contexts with mounted sockets"
# weight: 1
# aliases: ["/first"]
tags: ["devops", "linux"]
showToc: true
TocOpen: false
draft: false
ShowBreadCrumbs: true
---

So if you are like me, who likes to develop with an remote docker host, you likely used 
[docker contexts](https://docs.docker.com/engine/context/working-with-contexts/) for connecting to a remote endpoint:
```bash
docker context create remote --docker "host=ssh://user@myhost.org"
docker context use remote
```
With the `docker context use` command, your remote endpoint is now the default docker endpoint on your system.
You can take a look at your contexts with the command `docker context ls`.


However, depending on your internet connection, you might notice that a command is always a little slow.
This happens because on every docker command, a new SSH session to your server is started.

Becoming tedious of this, I wrote a small script to make it faster:
```bash
#! /bin/bash
MYHOST=yourwebsite.org
SSHREMOTE=user@$MYHOST
SOCKET=/tmp/docker.remote.sock

# Cleanup proxy'ed socket
function onexit() {
  echo "Deleting old socket ..."
  rm $SOCKET
}
# Cleanup when Ctrl-C is pressed
trap onexit EXIT
# Delete old socket file if it exists
if [ -f "$SOCKET" ]; then  
  onexit
fi

echo "Proxying docker from '$YOURWEBSITE' on '$SOCKET' ..."
# Forward remote docker socket to your host
ssh -nNT -L $SOCKET:/var/run/docker.sock $SSHREMOTE
```

With this command, you take your remote docker socket (`/var/run/docker.sock`), 
and mount it at your local system with a temporary file (`/tmp/docker.remote.sock`).
You can configure your remote host and the local socket location. The following lines are just for cleaning up an old socket.
The last line is for mounting the remote socket at your host. Let's break it down:
- `-n`: so ssh can be run in background
- `-N`: don't execute a remote command, just forward the socket
- `-T`: don't start a terminal session, just forward the socket
- `-L <localsocket>:<remotesocket>`: forward a remote socket to your host

The advantage of this command: it holds a long-running connection to your server of choice.
I execute this script in a `tmux` session so it's always running in the background.

Now you just have to create a context for your forwarded socket:
```bash
docker context rm remote
docker context create remote --docker "unix:///tmp/docker.remote.sock"
docker context use remote
```

**That's it!**

Now you can use test your connection:

```
docker ps
```

You should see a noticable improvement if don't have the best server or internet connection.