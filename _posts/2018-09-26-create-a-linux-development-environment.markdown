---
layout: post
title: "Create a Linux development environment"
date: "2018-09-26 11:52:08 +0800"
category: Linux
tags: openSUSE
---

# Linux 发行版选择

作为一个`Linux`新手，最重要地就是选择一款真正适合自己的发行版。通过不断地探索我最终选择了`openSUSE Tumbleweed`搭配`Gnome`作为我的软件开发平台。

# Gnome 调教

既然选择了`Gnome`当然要调教一番提高系统颜值。`Gnome`的调教不是很难，所有的东西你都能够在网络上找到，但是如何合理搭配需要自己尝试。
接下来，我将向大家展示我的成果。

![正在加载](/images/Screenshot_20180926131916.png)

## Gnome Tweaks

说到`Gnome`调教，自然无法离开`Tweaks`，我们的调教工作都是在他的基础上进行的。

### 安装

`openSUSE`开发版的`Gnome`桌面环境已经自带了`Tweaks`，无须安装。如果你的`openSUSE`没有该软件请运行以下指令进行安装。

```bash
sudo zypper install gnome-tweaks
```

### 配置主题

详情如图：
![正在加载](/images/Screenshot_20180926135519.png)

#### 安装主题

##### Application: Adwaita-dark

该主题是`gnome`默认安装的主题，无须下载。

##### Cursor: Capitaine-cursors

该主题模拟了苹果的鼠标主题，值得入手。可以点击[这里](https://krourke.org/projects/art/capitaine-cursors)查看详情。预览如下：
![正在加载]({{ site.baseurl }}/images/cursor-preview.png)

##### Icons: Papirus-Dark

该图标是我使用过的最好看的，没有之一。并且可以使用源安装，一次安装永久更新，可以说是非常方便了。可以运行以下命令进行安装：

```bash
sudo zypper install papirus-icon-theme
```

如果有一些朋友不想通过这种方式安装的可以点击[这里](https://github.com/PapirusDevelopmentTeam/papirus-icon-theme/)查看详情。

### 配置插件

除了主题，还有一些插件可供选择，使得你的桌面更加的美观与实用。

#### Dash to Dock

该插件可以使`Dash`变为一个和苹果系统类似的`Dock`，我这里也是通过配置做出了类似`ubuntu`的效果。具体配置不再给出，大家可以自己摸索。

#### Dynamic top bar

通过`Dynamic top bar`可以透明化顶部的状态栏。

# 软件安装

Linux 安装软件并不是一件很困难的事情。接下来我将展示一些常见软件的安装。

## Oracle JDK

我日常使用的语言是`JAVA`，所以需要在`Linux`上安装`JDK`。

### 下载

如果你使用的是比较新的`JDK`，那么你应该下载的是以`rpm`结尾的安装包，下载完双击打开即可。

如果你使用的是相对来说比较旧的`JDK`，那么你应该下载的是以`rpm.bin`结尾的安装包。之后在文件所在目录运行以下命令即可：

```bash
chmod a+x jdk-*-rpm.bin
sudo ./jdk-*-rpm.bin
```

## Visual Studio Code

安装`Visual Studio Code`的方法有许多，这里我通过源的方式进行安装，一次安装升级无忧。
首先通过以下命令添加源：

```bash
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
sudo sh -c 'echo -e "[code]\nname=Visual Studio Code\nbaseurl=https://packages.microsoft.com/yumrepos/vscode\nenabled=1\ntype=rpm-md\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/zypp/repos.d/vscode.repo'
```

再通过以下命令安装`Visual Studio Code`:

```bash
sudo zypper refresh
sudo zypper install code
```

## Atom

安装`Atom`我也使用源进行安装。
首先通过以下命令添加源：

```bash
sudo sh -c 'echo -e "[Atom]\nname=Atom Editor\nbaseurl=https://packagecloud.io/AtomEditor/atom/el/7/\$basearch\nenabled=1\ntype=rpm-md\ngpgcheck=0\nrepo_gpgcheck=1\ngpgkey=https://packagecloud.io/AtomEditor/atom/gpgkey" > /etc/zypp/repos.d/atom.repo'
sudo zypper --gpg-auto-import-keys refresh
```

通过以下命令安装`Atom`

```bash
# Install Atom
sudo zypper install atom
# Install Atom Beta
sudo zypper install atom-beta
```
