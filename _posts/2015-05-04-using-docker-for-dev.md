---
layout: post
title: "使用Dockers管理个人开发环境"
description: ""
category: Docker
tags: []
---
{% include JB/setup %}
当我们新安装Linux系统或者登陆了一个非自己环境的Linux系统时，通常都得做许多繁杂而又重复的工作，安装各种软件、搭建开发环境等，为了解决这个问题，我将使用Docker来管理自己的开发环境。

简单版的在线docker介绍：[Docker —— 从入门到实践](http://dockerpool.com/static/books/docker_practice/index.html)

#### 基础镜像
- - -
自己搭建了一个CentOS 7的基础开发镜像`centos-dev:base-v1.2`，修改软件源为163源，装了各种常用软件以及个人配置：git、vim、zsh等等。

![docker images](/assets/img/201505040101.png)

<!--more-->

#### 基于基础镜像创建开发环境镜像
- - -
可以通过镜像`centos-dev:base-v1.2`启动容器，然后搭建开发环境，最后提交为新的镜像。但是这样做，如果基础镜像进行更新了，又得手动基于新的基础镜像搭建开发环境了。所以采用Dockerfile，来生成开发环境镜像，以后如果基础镜像更新了，只需要执行`docker build`就能重新自动创建新的开发环境镜像了。

上图中的`centos-dev:blog-v1.1`就是我用于写博客的镜像，以下是该镜像的Dockerfile：

{% highlight php linenos %}
FROM centos-dev:base-v1.2

MAINTAINER Rokety Yang <yangrokety@gmail.com>

RUN yum -y install ruby ruby-devel gem samba samba-client
RUN gem sources -r https://rubygems.org/;gem sources -a https://ruby.taobao.org/
RUN gem install jekyll execjs therubyracer rake -V
RUN cd ~;git clone https://github.com/yangxikun/yangxikun.github.com.git

CMD ["/bin/zsh"]
{% endhighlight %}

一些小问题：

* Docker build不支持交互式
* 默认的容器中，无法使用systemctl命令，有篇博文介绍如何在容器里运行systemd：[Running systemd within a Docker Container](http://developerblog.redhat.com/2014/05/05/running-systemd-within-docker-container/)

小技巧：

* 查看运行的容器：`sudo docker ps`，查看所有容器：`sudo docker ps -a`。
* 参考[进入容器](http://dockerpool.com/static/books/docker_practice/container/enter.html)，下载`.bashrc_docker`，添加到`.bashrc`中，之后就可以通过`docker-enter <container name>`进入容器shell了。
* 在run的时候加上`--name <container name>`参数可指定容器名称，便于记忆。如：`sudo docker run --name blog -t -i -p 4000:4000 -p 139:139 -p 445:445 centos-dev:blog-v1.1`

#### 将镜像提交到Docker Hub上
- - -
在docker hub上注册账号，之后就可以将自己的镜像push上去了，这样就可以在需要的时候pull下来。

之前创建的仓库命名不符合要求，无法push，可通过如下命令修改名字：`sudo docker tag centos-dev:base-v1.2 yangxikun/centos-dev:base-v1.2`

![docker images](/assets/img/201505040102.png)

#### 通过挂载“宿主机”目录使用个人开发环境示例
- - -
例如，我登陆了一台服务器，需要修改某些文件，但是服务器上没安装vim，此时我可以pull镜像`centos-dev:base-v1.2`，然后执行`sudo docker run -t -i --name test -v /:/hostroot yangxikun/centos-dev:base-v1.2 /bin/zsh`将“宿主机”的根目录挂载到容器的/hostroot目录下，这样我就能用自己熟悉的环境修改“宿主机”的配置了。当然，前提是要宿主机支持docker。

