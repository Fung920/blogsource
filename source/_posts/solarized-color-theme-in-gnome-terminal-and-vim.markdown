---
layout: post
title: "Solarized Color Theme in Gnome-terminal and Vim"
date: 2016-07-12 03:27:18
comments: false
categories: miscellaneous
tags: solarized
keywords: solarized, vim, gnome-terminal
description: How to setup solarized color theme in gnome-terminal and vim
name: solarized color theme in gnome-terminal and vim
author: Fung Kong
datePublished: 2016-07-12 03:27:18
---
[Solarized](http://ethanschoonover.com/solarized) color theme is the most popular color theme in the world. For me, solarized color not only can protect my eyes from the computer but also can make my typing more efficiency. My laptop is using RHEL6 gnome desktop, all the configurations of this post will base on gnome terminal.
<!--more-->

## Gnome-terminal setting

### Downloading the source code from github

Download the dircolor(for ls command colorized) setting and solarized color theme for gnome-terminal from github: [dircolors-solarized](https://github.com/seebi/dircolors-solarized), [Solarized Colorscheme for Gnome Terminal](https://github.com/Anthony25/gnome-terminal-colors-solarized).

```vim
git clone https://github.com/seebi/dircolors-solarized
git clone https://github.com/Anthony25/gnome-terminal-colors-solarized
```

### Configuring the dircolor and solarized for gnome-terminal

After downloaded the source code, do the following steps:

```bash
##I'd like to put all my configurations into the .vim directory
cp -rp ./dircolors-solarized/dircolors.256dark ~/.vim/dircolors
ln -s  ~/.vim/dircolors ~/.dircolors
eval `dircolors ~/.dircolors`
cp -rp ./gnome-terminal-colors-solarized ~/.vim

##setting solarized color for gnome-terminal
cd ~/.vim/gnome-terminal-colors-solarized
./set_dark.sh

##Just for ubuntu
#For ubuntu solarized color, put following lines into .profile in ubuntu
if [ -x /usr/bin/dircolors ]; then
    test -r ~/.dircolors && eval "$(dircolors -b ~/.dircolors)" || eval "$(dircolors -b)"
fi

alias l.='ls -d .* --color'
alias ll='ls -l --color'
alias ls='ls --color' 
```

Now, my gnome-terminal looks like this (I am also using tmux):

![Solarized for Gnome-termimal](/images/solarized-gnome.png)

It looks perfect, doesn't it?   

My bash PS1:

```bash
if [ "`id -u`" -eq 0 ]; then
   PS1="\e[0;32m|\d \t| \e[0;37m[\e[0;31m\u\e[0;32m@\[\e[0;34m\]\h \e[0;34m\e[4;35m\w\e[0m\e[0;37m:(\$(ls | wc -l))]\n\e[0;31m# \[\e[0m\]"
else
   PS1="\e[0;36m|\d \t| \e[0;37m[\e[0;31m\u\e[0;32m@\[\e[0;34m\]\h \e[4;35m\w\e[0m\e[0;37m:(\$(ls | wc -l))]\n$ \[\e[0m\]"
fi
```

## VIM color theme setting
When I configured the VIM solarized color, I faced one issue. The result looks ugly:

![VIM solarized mixed](/images/vim-solarized-mixed.png)

I don't know why, but I found a solution via google, below code snippet is from my vimrc file:

```bash
#Try to change the color number from 16 to 256 or versa vise while the color looks weird
if $TERM == "xterm-256color"
   set t_Co=256
endif

if $COLORTERM == 'gnome-terminal'
   set t_Co=256
endif

let g:solarized_termcolors=16
set background=dark
colorscheme solarized
```

After that, my vim color theme looks perfect:

![vim solarized](/images/vim-solarized.png)

***EOF***
