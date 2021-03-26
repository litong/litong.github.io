---
layout: post
title: "使用Docker搭建GitLab服务器"
date: "2019-08-15 01:56:12 +0800"
category: Server
tags: Docker Git GitLab
---

# 使用场景

由于一些特殊的原因我们必须在本地搭建一个GitLab的服务器。

恩，这就是这篇博客的由来。

# 开始搭建服务器

![GitLab本地服务器](/images/Screenshot from 2019-08-15 03-32-48.png)

GitLab提供了很多种方式搭建本地服务器，你可以在[这里](<https://about.gitlab.com/install/>)找到它们。这里我们选择了我们认为比较简单的Docker进行搭建。

## 前置条件

既然使用Docker搭建服务器，那么Docker是必不可少的软件。

除此之外，你还需要网络，或者GibLab的镜像。

## 创建GitLab服务

我们希望能够通过服务的方式启动，所以说我们创建了一个服务，通过这个服务，我们完成整个搭建和运行的过程。

### 创建服务文件

首先，我们在/etc/systemd/system目录下创建一个包含以下内容的名为docker.gitlab.service的文件：

```bash
[Unit]
Description=GitLab Service
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
Restart=always
ExecStartPre=-/usr/bin/docker exec %n stop
ExecStartPre=-/usr/bin/docker rm %n
ExecStartPre=/usr/bin/docker pull gitlab/gitlab-ce:latest
ExecStart=/usr/bin/docker run --rm --name %n \
    --hostname gitlab.example.com \
    --volume /srv/gitlab/config:/etc/gitlab:Z \
    --volume /srv/gitlab/logs:/var/log/gitlab:Z  \
    --volume /srv/gitlab/data:/var/opt/gitlab:Z  \
    --publish 443:443 \
    --publish 80:80 \
    --publish 22:22 \
    gitlab/gitlab-ce:latest

[Install]
WantedBy=default.target
```

### 启动和创建自启动

接下来你就可以启动这个服务并为它指定自启动了。

```bash
sudo systemctl enable docker.gitlab.service
sudo systemctl start docker.gitlab.service
```
