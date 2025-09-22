---
layout: post
date: 2025-08-27
tags: Server
---

<!-- # 狂神 —— Docker 学习笔记（二） -->

## 容器管理可视化面板

### Portainer

#### 参考

[官网](https://www.portainer.io/)

[官网 Deploy](https://docs.portainer.io/start/install-ce/server/docker/linux)

#### 安装

这里下载 CE （社区）版本就可以了，BE 是企业版。官网下载命令中带 `lts` Tag，实操拉取的话会超时（已配置国内镜像），所以这里直接省略标签进行下载。

```bash
# 创建卷来存储数据
docker volume create portainer_data
# 拉取运行 portainer
docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
```

```shell
[root@iZ2ze41p5bh3hk80pucxofZ local]# docker pull portainer/portainer-ce:lts
Error response from daemon: Get "https://registry-1.docker.io/v2/": net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
```

下载完成后使用浏览器访问：`https://<公网IP>:9443` 。
打开后就是下面这个样子，首次创建用户即可：

![Description](https://docs.portainer.io/~gitbook/image?url=https%3A%2F%2Fcontent.gitbook.com%2Fcontent%2FXI7douejaBgpZ6CP2zJf%2Fblobs%2FnNR4InVfHQ3iUqIInTAP%2F2.32-initial-setup-username.png&width=768&dpr=4&quality=100&sign=53f28ee7&sv=2)

可以看到我们的 Docker：
![Description](/images/102243-698279703.png)

Docker 中的 Images、Containers、Volumes、Networks ：
![Description](/images/102243-877070341.png)

点击查看容器：
![Description](/images/102243-103613053.png)

### Rancher（CI/CD 用这个）

[官网](https://www.rancher.com/)

阿里云2G的内存扛不住这个，先不搞了。。。

## 镜像（Image）详解

### 镜像是什么？

镜像是一个标准化软件包，包含了运行容器（Container）所需的所有文件、二进制文件、库、配置文件。

两个特点：

（1）镜像是由层（Layers）组成的。每个层表示对文件系统的更改（添加、移除、修改文件）
（2）镜像是不可变的。镜像一旦被创建就无法修改。

### 镜像加载原理

这里我们先上示例：
```shell
# 拉取 mysql:8.0
[root@iZ2ze41p5bh3hk80pucxofZ ~]# docker pull mysql:8.0
8.0: Pulling from library/mysql
# 可以看到下面是一层一层进行拉取的
72a69066d2fe: Pull complete 
93619dbc5b36: Pull complete 
99da31dd6142: Pull complete 
626033c43d70: Pull complete 
37d5d7efb64e: Pull complete 
ac563158d721: Pull complete 
d2ba16033dad: Pull complete 
688ba7d5c01a: Pull complete 
00e060b6d11d: Pull complete 
1c04857f594f: Pull complete 
4d7cfa90e6ea: Pull complete 
e0431212d27d: Pull complete 
Digest: sha256:e9027fe4d91c0153429607251656806cc784e914937271037f7738bd5b8e7709
Status: Downloaded newer image for mysql:8.0
docker.io/library/mysql:8.0
# 拉取 mysql:latest
[root@iZ2ze41p5bh3hk80pucxofZ ~]# docker pull mysql
Using default tag: latest
latest: Pulling from library/mysql
# 这里直接进行复用
Digest: sha256:e9027fe4d91c0153429607251656806cc784e914937271037f7738bd5b8e7709
Status: Downloaded newer image for mysql:latest
docker.io/library/mysql:latest

[root@iZ2ze41p5bh3hk80pucxofZ ~]# docker inspect mysql
...
            "Layers": [
                "sha256:ad6b69b549193f81b039a1d478bc896f6e460c77c1849a4374ab95f9a3d2cea2",
                "sha256:fba7b131c5c350d828ebea6ce6d52cdc751219c6287c4a7f13a51435b35eac06",
                "sha256:0798f2528e8383f031ebd3c6d351f7d9f7731b3fd12007e5f2fdcdc4e1efc31a",
                "sha256:a0c2a050fee24f87fde784c197a8b3eb66a3881b96ea261165ac1a01807ffb80",
                "sha256:d7a777f6c3a4ded4667f61398eb1f9b380db07bf48876f64d93bf30fb1393f96",
                "sha256:0d17fee8db40d61d9ca0d85bff8b32ef04bbd09d77e02cc67c454c8f84edb3d8",
                "sha256:aad27784b7621a3e58bd03e5d798e505fb80b081a5070d7c822e41606b90a5c0",
                "sha256:1d1f48e448f9b8abb9a2aad1e76d4746b69957882d1ddb9c11115302d45fcbbd",
                "sha256:c654c2afcbba8c359565df63f6ecee333c9cc6abaeaa39838b05b4465a82758b",
                "sha256:118fee5d988ac2057ab66d87bbebd1f18b865fb02a03ba0e23762af5b55b0bd5",
                "sha256:fc8a043a3c7556d9abb4fad3aefa3ab6a5e1c02abda5f924f036c696687d094e",
                "sha256:d67a9f3f65691979bc9e2b5ee0afcd4549c994f13e1a384ecf3e11f83d82d3f2"
            ]
...
```
镜像层级化结构的好处：可以复用，减少存储量。如果用到了其他镜像的层级，则直接复用，跳过该层级的拉取。

分层是通过内容寻址存储和联合文件系统实现的。