---
layout: post
title: "Make Mac Terminal Console / iTerm Colorful"
date: 2014-09-11 14:36:31 +0800
comments: true
categories: Mac
tags: [Mac, Terminal] 
keywords: mac, tool, iterm, terminal, color, 终端, 颜色
description: 无论是Mac的Terminal终端，还是iTerm，默认都是黑白色的，比较单调。最起码应该支持目录和文件用不同的颜色显示。好在我们可以通过自己配置命令终端的颜色。Neither Terminal or iTerm on Mac is white/black color by default, which looks tedious. What if different entries shown in different colors? 
---
Neither Terminal or iTerm on Mac is white/black color by default, which looks tedious. What if different entries shown in different colors?  
Fortunately, we can do this just by writing a configuration file, then the terminal console or iTerm can look like this:  


![dayo icon](/images/post/2014/09/iterm.png)  
<!--more-->
  
So how can we do this?   
It's very simple, just create a file named `~/.bash_profile`, and fill it with the following content:  

``` bash ~./bash_profile
#enables colorin the terminal bash shell export
export CLICOLOR=1

#sets up thecolor scheme for list export
export LSCOLORS=gxfxcxdxbxegedabagacad

#sets up theprompt color (currently a green similar to linux terminal)
export PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;36m\]\w\[\033[00m\]\$ '

#enables colorfor iTerm
export TERM=xterm-color
```


<br/>
If you are not satisfied with that color, you can change the value of LSCOLORS.    

The value of LSCOLORS describes what color to use for which attribute when colors are enabled with CLICOLOR.  This string is a concatenation of pairs of the format fb, where f is the foreground color and b is the background color.
 
The color designators are as follows:
 
- a     black
- b     red
- c     green
- d     brown
- e     blue
- f     magenta
- g     cyan
- h     light grey
- A     bold black, usually shows up as dark grey
- B     bold red
- C     bold green
- D     bold brown, usually shows up as yellow
- E     bold blue
- F     bold magenta
- G     bold cyan
- H     bold light grey; looks like bright white
- x     default foreground or background
 

The order of the attributes are as follows:
 
1.   directory
2.   symbolic link
3.   socket
4.   pipe
5.   executable
6.   block special
7.   character special
8.   executable with setuid bit set
9.   executable with setgid bit set
10.  directory writable to others, with sticky bit
11.  directory writable to others, without sticky bit


>I'm tring to write my blog in English, please leave a message to me if you find any mistake, I'll appreciate it :)  
>我会尽量用英文来写博客，如果你发现任何语法错误或表达错误，不要犹豫，请帮忙斧正，感激不尽！