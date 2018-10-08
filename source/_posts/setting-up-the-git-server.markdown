---
layout: post
title: "Setting Up the Git Server"
date: 2016-07-14 02:49:33
comments: false
categories: miscellaneous
tags: git
keywords: git server
description: how to setup git server on linux
name: setting up the git server
author: Fung Kong
datePublished: 2016-07-14 02:49:33
---
[GITHUB](https://github.com/) is a free and open source public code repository, from small to big project. Many developers are using GitHub everyday, even me, although I'm not a coder. But because of its open source, everyone can visit your code, including sensitive data. Based on this, many develop teams will build their own git server. This post will show you how to build a git server on Linux platform.
<!--more-->
## Setup ssh keys for authentication
With ssh keys authentication, there's no web interface for git server management. And for safety, it's recommended to disable git user login with ssh. The first step we should do is creating a git user, setting up the ssh keys for authentication, and disabling git user for ssh login.

- Create git user on server

```c
groupadd -g 510 git  #Creating group
useradd -g git -m -d /home/git git   #Creating user
```

- Setup ssh authentication

Generate the ssh key from you local user, and put the ssh keys to git server.

```c
# On the client
ssh-keygen -C "admin@oracle ma.com"

# On the server with git user
mkdir .ssh && chmod 700 .ssh
touch .ssh/authorized_keys && chmod 600 .ssh/authorized_keys

# scp the authentication to git server
scp .ssh/id_rsa.pub git@node1:~/.ssh/authorized_keys
```

- Disable ssh login

On the server end, disable shell login for git user.

```bash
usermod -s /usr/bin/git-shell git
```

## Setting remote repository

On the server, I created a directory for my project.

```bash
mkdir -p /git/project
chown -R git:git /git/project/
git init --bare project.git
chown -R git:git /git/project/project.git/
```

- `git init` and `git init --bare`

    Repository created by `git init` are called working directories. Working directories will include two type of files, one is your real project files, another is the `.git` file folder contains the git configuration information about the repository, that is metadata of git repository.   
    But with `git init --bare`, it creates bare repositories, which only contains metadata of git repository. And do not have a `.git` file in the bare repositories. For initializing remote repository or git server, it's recommended to use `--bare` options.   
    With the working directory, we need to modify the git configuration first.

```bash
# Adding below line to remote repository if we're using working directory on git server
$ tail -2 .git/config
[receive]
    denyCurrentBranch = ignore
```

After we pushed from the local to remote, if you want to see the changes on the git server, you need to run `git reset --hard HEAD` command.

## Using git server from local machine

Assume my local repository is `/worktmp/git-tutorial`.

```c
[root@node2]# cd /worktmp/git-tutorial/
[root@node2]# cat >>hello.md<< EOF
> Hello, World!
> EOF
[root@node2]# git init .
[root@node2]# git add .
[root@node2]# git commit -m "initial commit"
[root@node2]# git remote add origin git@node1:/git/project/project.git/
[root@node2]# git push origin master
Counting objects: 3, done.
Writing objects: 100% (3/3), 217 bytes, done.
Total 3 (delta 0), reused 0 (delta 0)
To git@node1:/git/project/project.git/
 * [new branch]      master -> master
```

Now we can clone this repository from somewhere else.

```c
[root@node2]# mkdir /worktmp/test
[root@node2]# cd /worktmp/test/
[root@node2]# git clone git@node1:/git/project/project.git
Initialized empty Git repository in /worktmp/test/project/.git/
remote: Counting objects: 3, done.
remote: Total 3 (delta 0), reused 0 (delta 0)
Receiving objects: 100% (3/3), done.
[root@node2]# ls -ltr
total 4
drwxr-xr-x 3 root root 4096 Jul 14 14:31 project
[root@node2]# cat project/hello.md
Hello, World!
```

Do some changes and push to server.

```c
[root@node2 /worktmp/test:(1)] # cd project/
[root@node2 /worktmp/test/project:(1)] # cat >>hello.md<< EOF
> Adding some lines
> EOF
[root@node2 /worktmp/test/project:(1)] # git add .
[root@node2 /worktmp/test/project:(1)] # git commit -m "adding lines"
[master 9fa2159] adding lines
 Committer: root <root@node2.(none)>
Your name and email address were configured automatically based
on your username and hostname. Please check that they are accurate.
You can suppress this message by setting them explicitly:

    git config --global user.name "Your Name"
    git config --global user.email you@example.com

If the identity used for this commit is wrong, you can fix it with:

    git commit --amend --author='Your Name <you@example.com>'

 1 files changed, 1 insertions(+), 0 deletions(-)
[root@node2 /worktmp/test/project:(1)] # git push origin master
Counting objects: 5, done.
Writing objects: 100% (3/3), 268 bytes, done.
Total 3 (delta 0), reused 0 (delta 0)
To git@node1:/git/project/project.git
   55bc36f..9fa2159  master -> master
```

From the original local directory, pull the newest update to local repository.

```c
[root@node2:/worktmp/git-tutorial]$ls -ltr
total 4
-rw-r--r-- 1 root root 14 Jul 14 14:24 hello.md
[root@node2:/worktmp/git-tutorial]$git pull origin master
remote: Counting objects: 5, done.
remote: Total 3 (delta 0), reused 0 (delta 0)
Unpacking objects: 100% (3/3), done.
From node1:/git/project/project
 * branch            master     -> FETCH_HEAD
Updating 55bc36f..9fa2159
Fast-forward
 hello.md |    1 +
 1 files changed, 1 insertions(+), 0 deletions(-)
[root@node2:/worktmp/git-tutorial]$ls -ltr
total 4
-rw-r--r-- 1 root root 32 Jul 14 14:44 hello.md
[root@node2:/worktmp/git-tutorial]$cat hello.md
Hello, World!
Adding some lines
```

Now, we can push, pull, add, clone this remote repository which only accessible by authorized users.   
Next topic will introduce the `git` commands.


***EOF***
