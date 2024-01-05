+++ 
date = 2022-09-04T22:55:00+02:00
title = "Setup passwordless access via SSH to a remote server"
description = "A quick and easy way to setup passwordless access to a remote server via SSH with the help of a PowerShell one-liner"
authors = ["Stefano Previato"]
tags = ["PowerShell", "SSH", "passwordless", "server", "one-liner"]
categories = ["PowerShell", "Security"]
+++

Managing remote servers is part of my job as a software engineer, and I usually access them via SSH.

One thing that is really annoying is typing your password every single time, and for obvious reason setting an empty password is not a workaround :)

SSH servers usually support key authentication besides password authentication (sometimes it is even the only method of authentication allowed!), and making the server trust your computer is trivially easy to do.

If you never setup your SSH keys, open up a PowerShell and type:

```powershell
ssh-keygen
```

This command will generate a new key pair for your SSH client.

If you already did that, you can skip the key generation as your existing keys can be used as well.

Next, we need to append your public client key (that you just generated or that you already have) to the list of authorized keys on the SSH server.

To do this, we can pipe a couple of commands together in a beautiful PowerShell one-liner:

```powershell
# as an example, let's assume our server is called `myserver`
type $env:USERPROFILE\.ssh\id_rsa.pub | ssh myserver "cat >> .ssh/authorized_keys"
```

Let me explain what the above command does:

- `type` literally types the content of the parameter passed, in this case the `id_rsa.pub` file that is found inside the `.ssh` folder of the current user directory (that's what `$env:USERPROFILE` does)
- `ssh myserver` opens a new connection to the remote server and executes the command passed as a parameter
- `cat >> .ssh/authorized_keys` is a command that takes the standard input and appends it (with the `>>` operator) in the specified file

Since we're using the pipe operator (`|`), the output of the `type` command is being redirected to the input of the `ssh` command.

Run the command, type in your password (for the last time!) and wait. Once the operation is done, the SSH connection is automatically closed.

You could run into this error if the .ssh/authorized_keys file does not exist in the remote server:

```
bash: line 1: .ssh/authorized_keys: No such file or directory
```

To fix this, simply create the directory and the file on the remote server:

```powershell
ssh myserver "mkdir .ssh && cd .ssh && touch authorized_keys"
```

Now run the command above again, everything should work just fine.

You're ready to connect to the SSH server without ever using your password again!

Remember, you need to repeat this process whenever you access a new server or if you lose your SSH keys for some reason, such as formatting your PC.

Enjoy some passwordless SSH fun!
