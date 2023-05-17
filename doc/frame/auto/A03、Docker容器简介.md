> 前言：该系列文章，围绕持续集成：Jenkins+Docker+K8S相关组件，实现自动化管理源码编译、打包、镜像构建、部署等操作；

![](https://images.gitee.com/uploads/images/2022/0213/110229_5a26fb73_5064118.png "00-0.png")

# 一、Docker简介

## 1、基础描述

Docker作为开源的应用容器引擎，可以把应用程序和其相关依赖打包生成一个Image镜像文件，是一个标准的运行环境，提供可持续交付的能力，通过镜像文件可以创建多个Docker容器，这里可以理解为类创建对象的原理；镜像文件可以推送到仓库Repository中，这里可以理解为Git仓库管理代码的原理。

2、核心概念

- Image镜像：包含应用和依赖的类库，配置等；
- Container容器：通过镜像文件创建多个容器，运行打包应用；
- Repository仓库：存放镜像文件的云端服务；

镜像文件与容器，可以理解为基于快照启动虚拟机；或者类与实例对象的关系。

## 3、架构原理

![](https://images.gitee.com/uploads/images/2022/0213/110704_b9babf76_5064118.png "03-1.png")

Docker基于客户端-服务器的架构模式，Docker的守护进程（daemon）监听客户端的请求命令，从而管理镜像文件、容器等。

# 二、管理命令

## 1、查docker信息

```
# 查看基础信息
docker info

# 查看版本信息
docker version

# 查看命令说明
docker --help
```

## 2、镜像文件

**基础命令**

```
# 查看本地镜像列表
docker images  或者 docker image ls

# 搜索镜像
docker search ImageName

# 拉取镜像
docker image pull ImageName

# 删除镜像
docker image rm ImageName
```

**推送仓库**

首先在云服务平台申请私有的镜像管理仓库，并配置好访问仓库的账号和密码，通过docker命令把本地镜像文件推送到该仓库，这里以阿里云为例：

```
# 1、登录仓库
docker login --username=账户名 仓库_url

提示输出仓库密码：Login Succeeded

# 2、查看本地镜像
docker images
REPOSITORY    TAG       IMAGE ID       CREATED          SIZE
cloud-app     latest    b11d221cc3e0   13 seconds ago   662MB

# 3、标记上述镜像
docker tag b11d221cc3e0 仓库_url/cicada-image/cloud-app:latest

# 4、执行推送命令
docker push 仓库_url/cicada-image/cloud-app:latest

# 5、拉取该镜像到本地
docker pull 仓库_url/cicada-image/cloud-app:latest
```

注意：`cicada-image`是仓库的命名空间，点击`cloud-app`可以查看镜像操作的步骤文档：

![](https://images.gitee.com/uploads/images/2022/0213/110722_9d9747d4_5064118.png "03-2.png")

## 3、容器管理

```
# 列出正在运行或运行过的容器
docker ps -a

# 停止容器运行
docker stop 容器ID

# 删除指定容器
docker rm 容器ID

# 删除全部暂停容器
docker rm -f $(docker ps -a -q)
```

## 4、入门案例

```
- 拉取hello-world镜像
docker image pull hello-world

输出日志：
Using default tag: latest
latest: Pulling from library/hello-world

- 查看本地镜像
docker image ls
REPOSITORY    TAG       IMAGE ID       CREATED       SIZE
hello-world   latest    feb5d9fea6a5   7 weeks ago   13.3kB

- 运行hello-world
docker container run hello-world

输出日志：
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

# 三、构建镜像

## 1、Dockerfile

Dockerfile是一个文本文档，包含构建Docker镜像的指令，通过读取该脚本中的指令并执行，完成相关build过程。

**注意事项**

- 脚本命名Dockerfile并且没有任何后缀；
- Docker在构建镜像时，默认识别该文件；
- 通常脚本文件放在打包工程的根目录下；

## 2、基础样例

**语法说明**

- FROM：指定需要使用的基础镜像；
- MAINTAINER：定义脚本维护者；
- VOLUME：指定持久化文件目录；
- WORKDIR：切换到工作目录；
- ADD：将指定文件添加到容器中；
- COPY：将指定文件复制到容器中；
- RUN：镜像构建时执行的命令；
- ENTRYPOINT：容器参数配置；

**使用案例**

```
# 基础镜像
FROM java:8

# 维护者
MAINTAINER cicadasmile

# 持久化目录
VOLUME /data/docker/logs

# 添加应用服务JAR包
ADD auto-client.jar application.jar

# 配置参数
ENTRYPOINT ["java","-Dspring.profiles.active=dev","-Djava.security.egd=file:/dev/./urandom","-jar","/application.jar"]
```

## 3、构建镜像

**项目打包**

这里获取maven项目打包后的jar包，即`auto-client.jar`包，然后复制到docker镜像制作的目录下，与Dockerfile在同一级。

**结构如下**

![](https://images.gitee.com/uploads/images/2022/0213/110739_d82f09e1_5064118.png "03-3.png")

**镜像构建命令**

```
docker build -t client-img:latest .
```

构建流程执行完之后，查看镜像列表，上面构建的镜像已经存在；

## 4、运行容器

```
# 执行命令
docker run -d -p 8079:8079 client-img:latest

# 查看日志
docker logs 容器ID
```

访问容器中应用的接口，查看响应正常即可。

**源码参考：** https://gitee.com/cicadasmile/butte-auto-parent