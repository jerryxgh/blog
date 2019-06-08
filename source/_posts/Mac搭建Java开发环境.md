---
title: Mac下搭建Java开发环境
date: 2019-01-13 15:59:25
tags:
- java
- mac
- maven
- git
categories:
- wiki
---

说实话，工作这么多年，除了前两年多在写C，后来进入阿里每天都是写Java，对Java也越来越有感情了。 新Mac到手首先得搭建一下Java开发环境，毕竟Java是赖以吃饭的家伙呀。

# JDK
首先是安装JDK，先到[Orcale官方网站](https://www.oracle.com/technetwork/java/javase/downloads/index.html "Oracle JDK下载页")下载对应的JDK版本，然后执行安装即可。但是不推荐用这种方式安装JDK，这是因为官方的安装器可能不可靠，推荐用下面的方式进行安装。
```sh
# 增加cask包其他版本的tap
brew tap homebrew/cask-versions
brew cask install java8
# 使用jenv管理多个版本的jdk
brew install jenv
```
我在安装jenv的时候，报了一个错误：
```text
Updated 1 tap (homebrew/cask).
No changes to formulae.

Error: The following directories are not writable by your user:
/usr/local/sbin

You should change the ownership of these directories to your user.
  sudo chown -R $(whoami) /usr/local/sbin
```
按照提示运行这个命令即可：
```sh
## 修改目录 /usr/local/sbin 的owner为当前用户
sudo chown -R $(whoami) /usr/local/sbin
```

如果已经用Oracle的安装器安装了Java8，可以用下面的命令删除，之后用homebrew安装。
```sh
# Run this command to just remove the JDK
sudo rm -rf /Library/Java/JavaVirtualMachines/jdk<version>.jdk

# Run these commands if you want to remove plugins
sudo rm -rf /Library/PreferencePanes/JavaControlPanel.prefPane
sudo rm -rf /Library/Internet\ Plug-Ins/JavaAppletPlugin.plugin
sudo rm -rf /Library/LaunchAgents/com.oracle.java.Java-Updater.plist
sudo rm -rf /Library/PrivilegedHelperTools/com.oracle.java.JavaUpdateHelper
sudo rm -rf /Library/LaunchDaemons/com.oracle.java.Helper-Tool.plist
sudo rm -rf /Library/Preferences/com.oracle.java.Helper-Tool.plist
```
brew默认的官方源非常慢，可以用下面的方法切换源：
```sh
# 替换brew.git:
cd "$(brew --repo)"
git remote set-url origin https://mirrors.ustc.edu.cn/brew.git

# 替换homebrew-core.git:
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://mirrors.ustc.edu.cn/homebrew-core.git
```
上面的源如果比较慢，可以试试下面的这个：
```sh
https://git.coding.net/homebrew/homebrew.git
```
如果想切换回官方源，用下面的命令：
```sh
# 重置brew.git:
cd "$(brew --repo)"
git remote set-url origin https://github.com/Homebrew/brew.git

# 重置homebrew-core.git:
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://github.com/Homebrew/homebrew-core.git
```

# Maven
Maven当然是必须安装的，到[Maven官方网站](http://maven.apache.org/download.html)下载Maven安装包，解压出来，按照[Maven安装指南](http://maven.apache.org/install.html)安装即可。我是同时使用Zsh和Bash的，为了同时配置这两个shell的环境变量，我把公共的环境变量设置，放在了`~/.profole`。
```sh
## 配置Maven的环境变量
export PATH="/Users/xxx/local/apache-maven-<version>/bin:$PATH"
```
然后在`~/.zshrc`和`~/.bashrc`中添加下面的代码加载环境变量配置：
```sh
if [ -f ~/.profile ]; then
    source ~/.profile
fi
```
`~/.profile`只用来配置环境变量，需要满足加载幂等性，即无论被加载了多少遍，效果都一样。

## IntelliJ IEDA
IDEA几乎一统Java IDE的天下了，记得之前写C的时候，编辑器五花八门，Emacs、Vim、Source Edit等等各种都有人用，后来搞Java发现只有一个：IDEA。
Idea支持插件扩展，这里记录下我使用的插件：
* IdeaVim
  我是个Emacs控，平时的文本编辑、GTD等都是在Emacs中完成的，同时我也是Vim控，希望Vim的高效，因此我是用Emacs+Evil的方案搭建C语言的开发环境的。Java用Emacs不太方便，切换到IDEA必须安装的插件就是IdeaVim。
* VisualVm Launcher
  方便本地做Java的app profiling。

## git
git就不多说了，必备。
```sh
brew install git
```

## emacs
emacs的用户真的很少，但是用了就离不开啦。
```sh
brew cask install emacs
```
## http-server
有时候做一些实验，需要快速搭建http静态服务器，用`http-server`一条命令就能搞定：
```sh
cd <target directory> && http-server
```
so easy ;D，用下面的命令安装`http-server`即可：
```sh
npm install http-server -g
```

# 参考资料
1. mac安装Java：https://stackoverflow.com/questions/24342886/how-to-install-java-8-on-mac
2. brew加速：https://lug.ustc.edu.cn/wiki/mirrors/help/brew.git
3. mac删除jdk: https://stackoverflow.com/questions/19039752/removing-java-8-jdk-from-mac
