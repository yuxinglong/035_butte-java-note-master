> 前言：该系列文章，围绕持续集成：Jenkins+Docker+K8S相关组件，实现自动化管理源码编译、打包、镜像构建、部署等操作；

![](https://images.gitee.com/uploads/images/2022/0213/110229_5a26fb73_5064118.png "00-0.png")

# 一、Jenkins安装

## 1、环境部署

**下载Jenkins包**

注意这里直接下载war文件，以Java服务的形式启动。

- 环境：war运行
- 版本：2.289.3
- 类型：Generic Java package (.war)

**启动命令**

```
java -jar jenkins.war --httpPort=8090
```

**启动日志**

```
Jenkins is fully up and running
```

**访问本地端口：8090**

该页面会提示初始登录密码的位置，查看该文件中初始密码，并完成登录。

```
/.jenkins/secrets/initialAdminPassword
```

**安装推荐插件**

登录之后先把推荐的插件装上。

![](https://images.gitee.com/uploads/images/2022/0213/104923_f774949f_5064118.png "01-1.png")

**创建用户**

插件安装完成之后会提示创建用户。

```
账号：admin  密码：admin
```

这样初始化完成。

**重新启动：restart**

```
Jenkins_url/restart
```

## 2、配置与插件

**基础配置**

打开：`Manage-Jenkins`选项，配置`Global-Tool-Configuration`选项：

![](https://images.gitee.com/uploads/images/2022/0213/104940_5f3eb4e3_5064118.png "01-2.png")

```
- 查看JDK安装目录
/usr/libexec/java_home -V

- 查看Git安装目录
which git

- 查看Maven安装目录
mvn -v
```

配置组件：JDK、Git、Maven，采用开发环境的组件版本；

**插件安装**

![](https://images.gitee.com/uploads/images/2022/0213/105037_a6229c71_5064118.png "01-3.png")

安装如下插件：

```
1、Maven插件
Maven Integration plugin

2、Pipeline插件
Pipeline Utility Steps
```

# 二、本地Git打包

简介：基于Jenkins完成本地的Git仓库项目打包；

## 1、新建Item

![](https://images.gitee.com/uploads/images/2022/0213/105051_df2f737c_5064118.png "01-4.png")

- 任务名称：MavLoc，处理本地maven工程；
- 任务类型：选择构建maven项目；

## 2、构建记录管理

![](https://images.gitee.com/uploads/images/2022/0213/105102_16f29151_5064118.png "01-5.png")

保持构建的天数:3天，保持构建的最大个数：10个；

## 3、构建过程

前置`Pre-Steps`步骤，这里执行一次maven版本查看：

![](https://images.gitee.com/uploads/images/2022/0213/105113_ea383ff5_5064118.png "01-6.png")

构建`Build`步骤，这里直接写项目的pom路径，注意执行的maven命令：

![](https://images.gitee.com/uploads/images/2022/0213/105125_337b07c9_5064118.png "01-7.png")

```
clean package -Dmaven.test.skip=true
```

后置`Post Steps`步骤，注意选择构建成功后才执行，自行忽略这里shell语法的不入流组合：

![](https://images.gitee.com/uploads/images/2022/0213/105137_dcf789b2_5064118.png "01-8.png")

```
#!/bin/bash

BUILD_ID=dontKillMe

# 定义目录
AUTO_PATH=/项目路径/butte-auto-parent/

# 移动Jar包
cd $AUTO_PATH/auto-client/target/
pwd
mv auto-client-1.0-SNAPSHOT.jar $AUTO_PATH

cd $AUTO_PATH/auto-serve/target/
pwd
mv auto-serve-1.0-SNAPSHOT.jar $AUTO_PATH

# 启动服务
cd $AUTO_PATH

nohup java -jar auto-client-1.0-SNAPSHOT.jar &
echo "run auto-client ..."

sleep 20s

nohup java -jar auto-serve-1.0-SNAPSHOT.jar &
echo "run auto-serve ..."
```

## 4、执行构建

上述配置完成后，打开任务页面，执行如下操作：

- Build Now：执行上面的构建任务；
- 构建 #ID：查看控制台输出的日志；

这样就可以通过jenkins完成本地项目的打包和启动了。

# 三、API触发任务

## 1、用户令牌

简介：通过配置用户API访问的token令牌，脱离jenkins控制台，直接触发构建任务；

进入用户面板的设置选项，配置`API Token`:

![](https://images.gitee.com/uploads/images/2022/0213/105154_16c7ab96_5064118.png "01-9.png")

注意这里生成令牌后要立刻复制下来，页面会提示token无法复现。

## 2、任务令牌

任务配置的构建触发器模块，设置远程构建的令牌：

![](https://images.gitee.com/uploads/images/2022/0213/105207_4f4271b6_5064118.png "01-10.png")

上面已经给到token的使用方式。

## 3、脚本触发

通过如下方式，直接触发上述构建任务的流程：

```
curl http://用户:令牌@Jenkins_Url/job/MavLoc/build?token=任务令牌
```

这里通过脚本直接请求URL的方式触发流程。

# 四、打包Git项目

## 1、配置仓库

创建MavGit任务，这里不做过多的配置，与本地仓库相比，只是把仓库地址换成Gitee地址，只配置仓库url和分支即可，其他采用默认：

![](https://images.gitee.com/uploads/images/2022/0213/105222_dbdb5665_5064118.png "01-11.png")

因为是开放的仓库地址，所以不用配置账号密码，默认指定master分支，然后执行build构建。

## 2、查看空间

上面流程执行完后，查看MavGit的工作空间：`/.jenkins/workspace/MavGit`：

这里可以明显发现，仓库的代码已经被pull下来，并且完成了自动打包流程，后续结合shell脚本完成jar启动管理即可。

**源码参考：** https://gitee.com/cicadasmile/butte-auto-parent