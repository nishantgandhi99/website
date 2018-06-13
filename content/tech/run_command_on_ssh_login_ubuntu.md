---
author: "Nishant M Gandhi"
date: 2018-06-13
linktitle: Run command on ssh login in Linux
title: Run command on ssh login in Linux
image : ""
---

If you are using containers(like vagrant) for your development work,
you might be setting up environment variable each time you do ssh login in containers.

Ideally the environment variable should be set automatiaclly when you login inside container.

This can be archieved by adding following lines on ~/.bashrc file. You can write any command in if-loop that you want to be executed when you do ssh login succesfully in Ubuntu or any linux container.

```
if [[ -n $SSH_CONNECTION ]] ; then
    echo "set up environment variable"
fi
```
