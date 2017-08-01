---
title: docker命令
date: 2017-06-23 18:15:15
tags: docker
categories: docker
---
> 这些命令运行都是在本地Mac上运行的并非线上。
> Mac安装docker参考[官网](https://store.docker.com/editions/community/docker-ce-desktop-mac)
> 之后最好申请个加速器[阿里加速](https://account.aliyun.com/login/login.htm?oauth_callback=https%3A%2F%2Fcr.console.aliyun.com%2F&lang=zh#/imageList?onepasswdfill=3722C10CDB884416BE57D1C99C2C26A4&onepasswdvault=A2A7CEC8C02947EAAFD5227D16727010)，再配置一下 `Preferences`-->`Deamon`-->`Basic`-->`Registry mirrors` 
 

# 镜像相关
## 获取镜像
```
docker pull [选项] [Docker Registry地址]<仓库名>:<标签>
```

## 查看镜像
`显示镜像`
```
docker images
docker images ubuntu:16.04
```
`查看虚悬镜像`(有些镜像升级后旧版本就会变成虚悬镜像[dangling image] 仓库  标签 都是 <none>)
```
docker images -f dangling=True
```
`删除虚悬镜像`
```
docker rmi $(docker images -q -f dangling=true)
```
`安装自定义格式显示`
```
➜  workspace docker images --format "{{.ID}}: {{.Repository}}"
958a7ae9e569: nginx
```
`--filter 过滤器`
```
 docker images -f since=mongo:3.2
```

# 容器相关
## 容器启动
```
docker run -d -p 80:80 --name webserver nginx
```
这条命令 是用`nginx` 镜像 启动一个名为webserver的 容器， 并且映射到80 端口

## 进到容器
```
docker exec -it webserver bash
```
可以进到容器里面对容器webserver容器进行更改 `exit` 退出 

## 保存镜像(docker commit)
> 修改定制完容器，我们可以使用`docker commit`把它保存为新的镜像(但是因为修改历史不好查看，版本不好控制，一般很忌讳这样修改，要慎用)

```
➜ docker commit --author "fanquqi" --message "change index.html" webserver nginx:v2
sha256:2ad516b8fb8a76714e58a865d8099cbf6c31c36aabf0e4733b7d2950706674c3
```
可以通过docker images 查看
```
➜ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               v2                  2ad516b8fb8a        10 seconds ago      109 MB
nginx               latest              958a7ae9e569        3 weeks ago         109 MB
```
可以通过docker history 查看更改历史
```
➜  workspace docker history nginx:v2
IMAGE               CREATED              CREATED BY                                      SIZE                COMMENT
2ad516b8fb8a        About a minute ago   nginx -g daemon off;                            306 B               change index.html
958a7ae9e569        3 weeks ago          /bin/sh -c #(nop)  CMD ["nginx" "-g" "daem...   0 B
<missing>           3 weeks ago          /bin/sh -c #(nop)  STOPSIGNAL [SIGTERM]         0 B
<missing>           3 weeks ago          /bin/sh -c #(nop)  EXPOSE 80/tcp                0 B
<missing>           3 weeks ago          /bin/sh -c ln -sf /dev/stdout /var/log/ngi...   22 B
<missing>           3 weeks ago          /bin/sh -c apt-get update  && apt-get inst...   52.2 MB
<missing>           3 weeks ago          /bin/sh -c #(nop)  ENV NJS_VERSION=1.13.1....   0 B
<missing>           3 weeks ago          /bin/sh -c #(nop)  ENV NGINX_VERSION=1.13....   0 B
<missing>           6 weeks ago          /bin/sh -c #(nop)  MAINTAINER NGINX Docker...   0 B
<missing>           6 weeks ago          /bin/sh -c #(nop)  CMD ["/bin/bash"]            0 B
<missing>           6 weeks ago          /bin/sh -c #(nop) ADD file:a90ec883129f86b...   57.1 MB
```
紧接着我们可以运行这个镜像到一个新的容器
```
docker run --name web2 -d -p 81:80 nginx:v2
```

## 使用Dockerfile



