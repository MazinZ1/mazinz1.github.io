---
layout: post
title: "iTunes乱码处理"
date: 2011/11/19
categories: [cn, misc]
tags: [itunes]
---

使用Mac时一個很头疼的问题就是mp3导入iTunes时的各种乱码。搜索后并没有发现很好的解决方法，所以就还是按在Ubuntu时的方式来料理了。不知道大家有没有更简单的方式。

首先是安装Mutagen

~~~
pip install mutagen
~~~

使用方法时配合find命令即可，我比较喜欢的方式是：

~~~
find . -iname "*.mp3" -execdir mid3iconv -e gbk --remove-v1 {} \;
~~~

效果是转换当前目录及子目录下的所有mp3标签为Unicode编码，并同时填满ID3v2, APEv2标签。

-e gbk 参数只针对gbk编码，其他编码的文件则相应修改，如 -e big5 或者 -e gb18030。

\-\-remove-v1 参数则删除ID3v1标签，因为ID3v1不支持中文Unicode 编码Orz……所以如果播放器只支持ID3v1标签的话就会遇到问题——一片空白，这也是我还没想到解决办法的地方。不过就iTunes来说这样就足够了。

-----------

EDIT1(May 30, 2014 12:42:34):

在OS X Mavericks下曾经遇到过`dyld: Library not loaded: /usr/local/homebrew/homebrew/opt/readline/lib/libreadline.6.2.dylib`这样的问题，我解决的方法是用下面的命令重新安装`readline`:

~~~
brew install --build-from-source readline
~~~



