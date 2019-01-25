---
layout: post
title: "Git Commands"
date: 2016-07-15 03:45:35
comments: false
categories: miscellaneous
tags: git
keywords: git commands
description: basic git operations
name: basic git operations
author: Fung Kong
date Published: 2016-07-15 03:45:35
---
Git is a powerful distributed version control system. In 2005, Linus Torvalds spent only two weeks developing this tool. The original purpose of git was version control for Linux OS source code. Since then, git become the most popular distributed version control system. In 2008, github was established for storing source code of open source projects. Now, more and more open source projects are moving to the github. This post introduces fundamentals of git commands.
<!--more-->
## 1. Initialization
The git server setting based on [setting git server](/setting-up-the-git-server.html).

### 1.1 Installation
You can install git from source code [Git for all platforms](https://git-scm.com/) or by Linux software management tool, such as `yum`, `apt-get`. Here shows the second approach.

```bash
#For RHEL
yum install git
#For ubuntu
sudo apt-get install git git-core
```

When installed, you can find the version through the `git --version` command:

```bash
git --version
git version 2.7.4
```

### 1.2 Identify yourself
Git provide `git config` command enables you configure your username and email address with the github, `global` means all the configurations will use this setting for local machine.

```bash
git config --global user.name "Your Name"
git config --global user.email "YourEmail@YourDomain.com"
#Using git config -l show all your git configuration
git config -l
```

After you configured your username and email address on the local machine, you also need to provide your SSH keys for authentication:

```bash
#Generate ssh public keys from your local machine
$ ssh-keygen -t rsa
```

Paste your `~/.ssh/id_rsa.pub` to github account setting's SSH keys. After uploaded, you can test the authentication from your local machine. For a private git server, please refer to [setting git server](/setting-up-the-git-server.html) for setting up ssh authentication.

```bash
ssh -T USERNAME@github.com
# if successfully, will show:
Hi USERNAME! You've successfully authenticated, but GitHub does not provide shell access.
# Or showing the following messages while failed with authentication
Permission denied (publickey).
```

### 1.3 Initializing local repository
Before we go through the next section, we need to know two important terminologies of git: ***working directory*** and ***staging area***. As its name implies, working directory is the local directory in our laptop/desktop. Besides, if we initialized the working directory as local repository, we would have a `.git` directory under the working directory, this file folder contains the git configurations, the version information, and most of all, the staging area(also are called index area). The command `git add file1` actually puts the changes of file1 into the staging area, we issue the `git commit` command to commit all the changes of the staging area to the current branch.

```bash
$ cd git-tutorial
$ vim readme.md
$ git init .         #Initial local repository in git-tutorial
Initialized empty Git repository in /worktmp/git-tutorial/.git/
$ git add .          #Add all the files in the current working directory to the staging area
$ git commit -m "commit messages"      #Commit the changes to the current branch

# For github repository
$ git remote add origin git@github.com:USERNAME/PROJECT_NAME.git

# For private git server
$ git remote add origin git@hostname:/project_dir/project.git

# Push your code to master branch
$ git push origin master
```

## 2. Repository commands
### 2.1 Push
As its name implies, push is putting all of your committed changes from working directory to remote directory.

```bash
[root@node2:/worktmp/git-tutorial]# cat hello.md
Hello everyone, this is my second push.
[root@node2:/worktmp/git-tutorial]# git status
# On branch master
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#
#  hello.md
nothing added to commit but untracked files present (use "git add" to track)
```

The `git status` result shows us that we changed a file, but haven't been committed.

```bash
[root@node2:/worktmp/git-tutorial]# git add .
[root@node2:/worktmp/git-tutorial]# git commit -m "second push"
[master b82094a] second push
 1 files changed, 1 insertions(+), 0 deletions(-)
 create mode 100644 hello.md
```

If you modified some files, `git diff` would show you the difference with different versions.

```bash
[root@node2:/worktmp/git-tutorial]# cat >>readme.md <<EOF
diff command to compare with before and after version
EOF
[root@node2:/worktmp/git-tutorial]# git commit -m "git diff command"
# On branch master
# Changed but not updated:
#   (use "git add <file>..." to update what will be committed)
#   (use "git checkout -- <file>..." to discard changes in working directory)
#
#  modified:   readme.md
#
no changes added to commit (use "git add" and/or "git commit -a")
[root@node2:/worktmp/git-tutorial]# git diff readme.md
diff --git a/readme.md b/readme.md
index 18a27a1..7379c0f 100644
--- a/readme.md
+++ b/readme.md
@@ -1 +1,2 @@
 Hi, this is the first initial.
+diff command to compare with before and after version
```

After we committed the changed, we can push them to master branch now.

```bash
[root@node2:/worktmp/git-tutorial]# git add .
[root@node2:/worktmp/git-tutorial]# git push origin master
Counting objects: 4, done.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 313 bytes, done.
Total 3 (delta 0), reused 0 (delta 0)
To git@node1:/git/project.git
   ae69504..b82094a  master -> master
```

### 2.2 Pull

`git pull` command can let you update your local repository to the newest one, for example, when you updated your changes on your office desktop, but when you came home, you would like to modify something and push the changes to the remote repository, then you must pull from remote repository before you modified your local files.


```bash
cd project
git pull orgin master

# For force pull, discard the modification of local repository by overwriting it
git reset --hard
git pull origin master
# Or
git fetch origin
git reset --hard origin/master
```

`git reset` command will be introduced in the following sections.   

After you update your local repository, you can push your changes into the remote repository.


### 2.3 Branches management

- Creating and switching branches

Git can have multiple branches, let's create another branch first.

```bash
# Creating source branch
[root@node2:/worktmp/git-tutorial]# git checkout -b source
Switched to a new branch 'source'
```

With the `-b` option, it means create and switch to new branch, this command equals:

```bash
git branch source
git checkout source
```

`git branch` command can show you what branch are you in.

```bash
[root@node2:/worktmp/git-tutorial]# git branch
    master
  * source
```

- Merging the branches

We can commit and push changed data when we are in branches. Let's modify something, and push it to the remote repository.

```bash
# Do some modifications
[root@node2:/worktmp/git-tutorial]# cat >>hello.md <<EOF
new branches.
EOF

# Commit the changes
[root@node2:/worktmp/git-tutorial]# git add hello.md
[root@node2:/worktmp/git-tutorial]# git commit -m "adding branches"
[source 33e6404] adding branches
 1 files changed, 1 insertions(+), 0 deletions(-)
```

Now, guess what happen if we back to the master branch? The modifications we changed before are gone.

```bash
[root@node2:/worktmp/git-tutorial]# git checkout master
Switched to branch 'master'
[root@node2:/worktmp/git-tutorial]# cat hello.md
Hello everyone, this is my second push.
[root@node2:/worktmp/git-tutorial]# git checkout source
Switched to branch 'source'
[root@node2:/worktmp/git-tutorial]# cat hello.md
Hello everyone, this is my second push.
new branches.
```

It's just because we committed at the source branch, not at the master branch. `git merge` command can merge multiple branches, and because of creating, merging, deleting branches is very fast, git recommend us using branches for some tasks, after merged the branches, delete those branches. This process just like what we did in master branch, but it's more safety.

```bash
[root@node2:/worktmp/git-tutorial]# git checkout master
Switched to branch 'master'
[root@node2:/worktmp/git-tutorial]# git branch
* master
  source
[root@node2:/worktmp/git-tutorial]# git merge source
Updating b33f2a9..33e6404
Fast-forward
 hello.md |    1 +
 1 files changed, 1 insertions(+), 0 deletions(-)
[root@node2:/worktmp/git-tutorial]# cat hello.md
Hello everyone, this is my second push.
new branches.
[root@node2:/worktmp/git-tutorial]# git branch -d source
Deleted branch source (was 33e6404).
[root@node2:/worktmp/git-tutorial]# git branch
* master
[root@node2:/worktmp/git-tutorial]# git push origin master
Counting objects: 5, done.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 327 bytes, done.
Total 3 (delta 0), reused 0 (delta 0)
To git@node1:/git/project.git
   b33f2a9..33e6404  master -> master
```

If we want to keep the source branch, and do some different task from master branch, we also can treat it as master branch.

```bash
[root@node2:/worktmp/git-tutorial]# touch branches.md
[root@node2:/worktmp/git-tutorial]# git add .
[root@node2:/worktmp/git-tutorial]# git commit -m "branches testing"
[source cd86d13] branches testing
 0 files changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 branches.md
[root@node2:/worktmp/git-tutorial]# git branch
  master
* source
[root@node2:/worktmp/git-tutorial]# git push origin source
Counting objects: 4, done.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 308 bytes, done.
Total 3 (delta 0), reused 0 (delta 0)
To git@node1:/git/project.git
 * [new branch]      source -> source
```

- Conflict resolution

Switch to the source branch, and do some modifications to files.

```bash
$ git status
# On branch source
# Changed but not updated:
#   (use "git add <file>..." to update what will be committed)
#   (use "git checkout -- <file>..." to discard changes in working directory)
#
#       modified:   readme.md
#
no changes added to commit (use "git add" and/or "git commit -a")

$ git branch
  master
* source

$ git add readme.md 
$ git commit -m "modified by source branch"
[source 6041c6f] modified by source branch
 1 files changed, 1 insertions(+), 1 deletions(-)
```

After that, we switch to the master branch, do some modifications, and commit it:

```bash
$ git status
# On branch master
# Changed but not updated:
#   (use "git add <file>..." to update what will be committed)
#   (use "git checkout -- <file>..." to discard changes in working directory)
#
#       modified:   readme.md
#
no changes added to commit (use "git add" and/or "git commit -a")
$ git add .
$ git commit -m "master branch"
[master 3e1a68c] master branch
 1 files changed, 1 insertions(+), 0 deletions(-)
$ git push origin master
Counting objects: 5, done.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 298 bytes, done.
Total 3 (delta 1), reused 0 (delta 0)
To git@node1:/git/git-tutorial.git
   7a96d51..3e1a68c  master -> master
```

Now, we have committed the changes on both source and master branch, if we try to merge them, it will warn you there're some conflicts need you to solve it manually before merge.

```bash
$ git merge source
Auto-merging readme.md
CONFLICT (content): Merge conflict in readme.md
Automatic merge failed; fix conflicts and then commit the result.

$ git status
# On branch master
# Unmerged paths:
#   (use "git add/rm <file>..." as appropriate to mark resolution)
#
#       both modified:      readme.md
#
no changes added to commit (use "git add" and/or "git commit -a")
```

The contents of readme.md file changed to as following, you can modify it and re-commit again.

```bash
$ cat readme.md
Conflict Resolution.
<<<<<<< HEAD
1. adding some line from node2 and pushing it to remote repo.
2. adding some line from node1
=======
Another change from master branch.
>>>>>>> source
```


- Clone multiple branches to local repository

With multiple branches, if you want clone the whole repository, you need to clone all of the branches one by one.

```bash
# Clone multiple branches into local repo
git clone -b source https://github.com/username/projectname.git
cd projectname
git clone https://github.com/username/project master
```

### 2.4 Rollback to the before version
Once we changed somethings wrong in the local repository, we need to revert the changes to the before version. Depending on what scenarios are, git provides kind of commands to let you rollback to the before version.

- Wrong modifications on the working directory

```bash
$ cat >>readme.md <<EOF
Only Wrong modifications on the working directory
EOF
$ git status
# On branch master
# Changed but not updated:
#   (use "git add <file>..." to update what will be committed)
#   (use "git checkout -- <file>..." to discard changes in working directory)
#
#       modified:   readme.md
#
no changes added to commit (use "git add" and/or "git commit -a")
```

The `git status` command already told us what should we do if we want to discard the changes, by using `git checkout -- file1` command. This command will let file1 back to the status of the last `git commit` or `git add`.

```bash
$ git checkout -- readme.md
$ git status
# On branch master
   nothing to commit (working directory clean)
$ cat readme.md
   The first initial.
```

- Wrong modifications on the working directory and added the changes to the staging area

    The same example, you modified the readme.md file, and you added these changes to the staging area.

```bash
$ cat >>readme.md <<EOF
Only Wrong modifications on the working directory
EOF

$ git status
# On branch master
# Changed but not updated:
#   (use "git add <file>..." to update what will be committed)
#   (use "git checkout -- <file>..." to discard changes in working directory)
#
#       modified:   readme.md
#
no changes added to commit (use "git add" and/or "git commit -a")

$ git add readme.md
$ git status
# On branch master
# Changes to be committed:
#   (use "git reset HEAD <file>..." to unstage)
#
#       modified:   readme.md
#
```

After you added the changes to the staging area, `git status` showed you another recommendation for unstaging.

```bash
$ git reset HEAD readme.md
Unstaged changes after reset:
M       readme.md
```

After we resetted the HEAD of readme.md file, we're back to the before added version, so we can use `git checkout -- filename` to revert all the changes.

```bash
$ git status
# On branch master
# Changed but not updated:
#   (use "git add <file>..." to update what will be committed)
#   (use "git checkout -- <file>..." to discard changes in working directory)
#
#       modified:   readme.md
#
no changes added to commit (use "git add" and/or "git commit -a")
```

- Wrong modifications on the working directory and committed the changes to the current branch

    The above two scenarios both are not committed the changes, what if we do when the wrong modifications are committed but not being pushed to the remote repository?

```bash
$ cat >>readme.md <<EOF
Only Wrong modifications on the working directory
EOF

$ git add readme.md
$ git commit -m "reverting committed changes testing"
```

`git log` can show us the committed history. `git reflog` can show us the commit id and HEAD version, we can revert our changes by using `git reset --hadr/soft commit_id` command.

```bash
$ git reflog
4da4888 HEAD@{0}: commit: reverting committed changes testing
3d88e42 HEAD@{1}: commit: adding hello.md
e340212 HEAD@{2}: commit: adding hello.md
5f9a0f6 HEAD@{3}: commit (initial): First initialize
```

For our example, we need to back to the version of commit id *3d88e42*, also the last committed version which can use `HEAD^` to specify.

```bash
$ git reset --hard 3d88e42
# or
$ git reset --hard HEAD^
HEAD is now at 3d88e42 adding hello.md
$ cat readme.md
The first initial.
$ git status
# On branch master
nothing to commit (working directory clean)
```

Now the `git reflog` output as below:

```bash
$ git reflog
3d88e42 HEAD@{0}: HEAD^: updating HEAD
4da4888 HEAD@{1}: commit: reverting committed changes testing
3d88e42 HEAD@{2}: commit: adding hello.md
e340212 HEAD@{3}: commit: adding hello.md
5f9a0f6 HEAD@{4}: commit (initial): First initialize
```

If we regret for the reverting and want to back to the commit id *4da4888*, we also can use `git reset` back to the specified version:

```bash
$ git reset --hard 4da4888
$ cat readme.md
The first initial.
Only Wrong modifications on the working directory
```

- `hard` and `soft`

    `HEAD` stands for version, `HEAD^` or `HEAD~` option stands for the latest commit's parent, `HEAD~10` means back to last 10 commit version.    
    `git reset --soft` only revert the commit, but remain the modified files in the working directory.   
    `git reset --hard` revert the changes to the last commit status, that means it will delete all the committed information, revert the file to the last committed status.

    For example:

```bash
$ cat >>readme.md <<EOF
differential of soft and hard
EOF

$ git add readme.md 
$ git commit -m "soft and hard first commit"
[master c9fb3e9] soft and hard first commit
 1 files changed, 1 insertions(+), 0 deletions(-)
$ git reset --soft HEAD^
$ git diff
$ git diff --cached
diff --git a/readme.md b/readme.md
index ae00a84..8cc4a14 100644
--- a/readme.md
+++ b/readme.md
@@ -1,2 +1,3 @@
The first initial.
Only Wrong modifications on the working directory
+differential of soft and hard

$ git diff HEAD
diff --git a/readme.md b/readme.md
index ae00a84..8cc4a14 100644
--- a/readme.md
+++ b/readme.md
@@ -1,2 +1,3 @@
 The first initial.
 Only Wrong modifications on the working directory
+differential of soft and hard
```

From the above outputs, the `git diff` shows nothing, but the `git diff --cached` and `git diff HEAD` shows the line we just added to readme.md file.   
**So, if you just want to revert the commit and keep the modifications in the working directory, use `--soft` option. If you don't want keep the modifications and want to revert the commit, please use `--hard` option.**



### 2.6 Reverting pushed committed changes

Last example shows how to revert committed changes on the local repository, but those changes were not pushed yet. What if we pushed those wrong changes to the remote repository?

```bash
# Add a new file to local repository and pushed it to remote repository
$ git add reverting.md
$ git status
# On branch master
# Changes to be committed:
#   (use "git reset HEAD <file>..." to unstage)
#
#  new file:   reverting.md

$ git commit -m "adding reverting.md"
[master c727ce3] adding reverting.md
 1 files changed, 1 insertions(+), 0 deletions(-)
 create mode 100644 reverting.md
$ git push origin master
Counting objects: 4, done.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 344 bytes, done.
Total 3 (delta 0), reused 0 (delta 0)
To git@node1:/git/project/project.git/
   6c71346..c727ce3  master -> master
```

Reverting the pushed data by using `git reset` and `git push` with `force` option.

```bash
$ git reflog
c727ce3 HEAD@{0}: commit: adding reverting.md
6c71346 HEAD@{1}: HEAD^: updating HEAD
9b6bdd6 HEAD@{2}: commit: modified readme.md

$ git reset --hard HEAD^
HEAD is now at 6c71346 adding readme file

$ git push origin -f
Total 0 (delta 0), reused 0 (delta 0)
To git@node1:/git/project/project.git/
 + c727ce3...6c71346 master -> master (forced update)
```

Above commands can be combined with one command:

```bash
git push -f origin HEAD^:master
```

### 2.7 Deleting the files
Besides add, modify files, we also can delete the files we don't want any more with `git rm` command.   

Below example showing deleting the file from working directory and version repository, that is deleted permanently

```bash
$ git rm hello.md
rm 'hello.md'
$ git commit -m "remove hello,md"
[master a7dd8e0] remove hello,md
 1 files changed, 0 insertions(+), 2 deletions(-)
 delete mode 100644 hello.md
$ ls
readme.md
```

Below example showing the deleted files by OS `rm` command, and before commit it, the version repository still keeping the information, so we can use `git checkout -- file` command to restore it.

```bash
$ rm -f readme.md
$ git checkout -- readme.md
$ ls
readme.md
```


***EOF***
