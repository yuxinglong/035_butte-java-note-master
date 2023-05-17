> 前言：该系列文章，围绕持续集成：Jenkins+Docker+K8S相关组件，实现自动化管理源码编译、打包、镜像构建、部署等操作；

![](https://images.gitee.com/uploads/images/2022/0213/110229_5a26fb73_5064118.png "00-0.png")

# 一、Webhook原理

Pipeline流水线任务通常情况下都是自动触发的，在Git仓库中配置源码改动后通知的地址即可。

例如在Gitee仓库中，基于WebHook的配置，可以在向仓库push代码后，自动回调预先设定的请求地址，从而触发代码更新后的打包动作，基本流程如下：

![](https://images.gitee.com/uploads/images/2022/0213/105940_b8e9c8ad_5064118.png "02-1.png")

这里涉及到两个核心配置：

- Gitee回调：即仓库接收到push请求后的通知地址；在仓库管理的`WebHooks`选项中；
- Jenkins流程：编写流水线任务，处理代码提交后的自动化流程；这里需要Jenkins地址可以在外网访问，网上的组件很多，自行选择搭建即可；

**注意**：可以先随意设置回调地址，在请求日志中直接拷贝请求参数，在postman中去触发Jenkins任务，这样在测试时会方便很多。

![](https://images.gitee.com/uploads/images/2022/0213/105952_c99756a7_5064118.png "02-2.png")

这里结合Gitee的帮助文档，去分析不同push动作的参数标识，可以判断分支的创建、推送、删除等操作，例如：

```
"after": "1c50471k92owjuh37dsadfs76ae06b79b6b66c57",
"before": "0000000000000000000000000000000000000000",
```

创建分支：before字符都是0；删除分支：after字符都是0；

# 二、流水线配置

## 1、插件安装

在Jenkins插件管理中，安装`Generic-Webhook-Trigger`插件，流水线`pipeline`相关组件在Jenkins初始化的时候已经安装了。

## 2、创建流水线

新建Item，输入任务名称，选择`pipeline`选项即可：

![](https://images.gitee.com/uploads/images/2022/0213/110017_8b093af2_5064118.png "02-3.png")

选择Webhook选项，页面提示了触发的方式。

## 3、触发流水线

```
http://用户名:密码@JENKINS_URL/generic-webhook-trigger/invoke
```

基于如上方式通过认证，触发流水线执行，会生成任务日志，即流程是通顺的。

# 三、Pipeline语法

## 1、结构语法

- triggers：基于hook模式触发流水线任务；
- environment：声明全局通用的环境变量；
- stages：定义任务步骤，即流程分段处理；
- post.always：最终执行的动作；

常规流程中的整体结构如下：

``` sh
pipeline {
    agent any
    triggers {}
    environment {}
    stages {}
    post { always {}}
}
```

把各个节点下的脚本配置进去，就会生成一个自动化的流水线任务。注意这里**不勾选**`使用Groovy沙盒`选项。

## 2、参数解析

这里说的参数解析是指，Gitee通过hook机制请求Jenkins服务携带的参数，这里主要解析post参数即可，解析方式看说明：

![](https://images.gitee.com/uploads/images/2022/0213/110040_e590045f_5064118.png "02-4.png")

这里从hook回调的参数中选了几个流程中使用的参数，下面看具体解析方式，在上图中点击新增：

```json
{
    "ref":"refs/heads/master",
    "repository":{
        "name":"butte-auto-parent",
        "git_http_url":"仓库地址-URL"
    },
    "head_commit":{
        "committer":{
            "user_name":"提交人名称",
        }
    },
    "before":"277bf91ba85996da6c",
    "after":"178d56ae06b79b6b66c"
}
```

![](https://images.gitee.com/uploads/images/2022/0213/110053_c39e0071_5064118.png "02-5.png")

把上述参数依次做好配置即可，这样在工作流中就可以使用这些参数。

## 3、触发器节点

这里即`triggers`模块配置，核心作用是加载触发流程的一些参数，后续在脚本中使用，其他相关配置按需选择即可，注意这里的参数需要在上个步骤中配置：

```
triggers {
    GenericTrigger(
        genericVariables: [
            [key: 'ref', value: '$.ref'],
            [key: 'repository_name', value: '$.repository.name'],
            [key: 'repository_git_url', value: '$.repository.git_http_url'],
            [key: 'committer_name', value: '$.head_commit.committer.user_name'],
            [key: 'before', value: '$.before'],
            [key: 'after', value: '$.after']
        ],
        // causeString: ' Triggered on $ref' ,
        // printContributedVariables: true,
        // 打印请求参数
        // printPostContent: true
    )
}
```

## 4、环境变量

声明一些全局的环境变量，也可以直接定义，在流程中用`${变量}`的方式引用：

```
environment {
    branch = env.ref.split("/")[2].trim()
    is_master_branch = "master".equals(branch)
    is_create_branch = env.before.replace('0','').trim().equals("")
    is_delete_branch = env.after.replace('0','').trim().equals("")
    is_success = false
}
```

这里根据hook请求参数，解析出分支的操作类型：是否创建、是否删除、是否主干分支，以及定义一个`is_success`流程是否成功的标识。

## 5、分段流程

这里主要分为五个步骤：解析数据、拉取分支、处理Pom文件、分支推送、项目打包；

```
stages {
    // 解析仓库信息
    stage('Parse') {
        
        steps {
            echo "仓库分支 : ${branch} \n仓库名称 : ${repository_name} \n仓库地址 : ${repository_git_url} \n提交用户 : ${committer_name}"
            script {
                if ("true".equals(is_master_branch)) {
                    echo "保护分支 : ${branch}"
                }
                if ("true".equals(is_create_branch)) {
                    echo "创建分支 : ${branch}"
                }
                if ("true".equals(is_delete_branch)) {
                    echo "删除分支 : ${branch}"
                }
            }
        }
    }
        
    // 拉取仓库分支
    stage('GitPull') {
        steps {
            script {
                if ("false".equals(is_delete_branch)) {
                    echo "拉取分支 : ${branch}"
                    git branch: "${branch}",url: "${repository_git_url}"
                }
            }
        }
    }
        
    // 解析仓库Pom文件
    stage('MvnPom') {
        steps {
            script {
                // 解析Pom文件内容
                def pom = readMavenPom file: 'pom.xml'
                def version = "${pom.version}"
                def encode = pom.getProperties().get("project.build.sourceEncoding")
                echo "Pom版本 : "+ version
                echo "Pom编码 : "+ encode
                def devVersion = "${branch}-"+version
                def jarName = "${branch}-"+version+".jar"
                echo "Now版本 : "+ devVersion
                echo "Jar名称 : "+ jarName
                
                // 修改Pom文件内容
                // pom.getProperties().put("dev.version","${devVersion}".trim().toString())
                // writeMavenPom file: 'pom.xml', model: pom
                
                echo "update pom success"
            }
        }
    }
        
    // 推送仓库分支
    stage('GitPush') {
        steps {
            script {
                echo "git push success"
            }
        }
    }
        
    // 本地打包流程
    stage('Package') {
        steps {
            script {
                sh 'mvn clean package -Dmaven.test.skip=true'
                is_success = true
            }
        }
    }
}
```

- 解析数据：解析并输出部分参数信息；
- 拉取分支：结合Git命令，拉取分支代码；
- 处理Pom文件：对pom文件的读取和修改；
- 分支推送：结合Git命令，推送分支代码；
- 项目打包：结合Mvn命令，完成项目打包；

**注意**：这里在本地测试流程时，并没有推送代码；在项目打包完成后，结合shell脚本完成服务的启动发布。

## 6、消息通知

在流程的最后，识别任务的执行标识`is_success`，通知相关人员是否打包成功，这里的通知方式可以选择邮件或者其他API推送的通知类型，不过多描述：

```
post {
    always {
        script {
            echo "notify : ${committer_name} , pipeline is success : ${is_success}"
        }
    }
}
```

## 7、执行日志

完成上面`pipeline`流水线脚本开发后，通过postman工具不断发送请求，完成脚本调试：

![](https://images.gitee.com/uploads/images/2022/0213/110112_69852738_5064118.png "02-6.png")

这里也可以点击流程里的不同模块，查看该模块下的日志信息：

![](https://images.gitee.com/uploads/images/2022/0213/110122_15b2926f_5064118.png "02-7.png")

说明：完整的`pipeline`脚本内容放在末尾的Gitee开源仓库中，有需要的自行获取。

**源码参考：** https://gitee.com/cicadasmile/butte-auto-parent