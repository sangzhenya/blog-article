# Jenkins 自动部署

博客后台每次 build 之后都需手动放到服务器上，然后停掉原本的服务，重新启动服务。为了简化流程所以使用 Jenkins 进行自动化部署。

首先项目是基于 Docker 的（至于 Jenkins 和 Docker 的安装非常流程化，这边不赘述）。

## 将项目打包成 Docker 镜像

首先编写 `Dockerfile` 简单如下，

```Dockerfile
FROM openjdk:11.0-jre-slim
MAINTAINER xinyue<xinyue@xinyue.com>
EXPOSE 8081
VOLUME /tmp
ADD /target/blog-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT [ "java", "-jar", "/app.jar", "--spring.profiles.active=prd"]
```

对于我的项目非常的简单，就是基于 openjdk 11 的镜像，运行一个 jar 包即可。编译，运行分别是以下两个 `docker` 命令, 相关的命令还会在 Jenkins 的配置中使用。

```bash
sudo docker build -f Dockerfile -t blog  . # 根据 Dockerfile 打包 Docker 镜像
sudo docker run -d -p 8081:8081 --name blog -t blog # 启动镜像，即根据镜像生成一个 container，启动 container
```

## 在 GitHub 中的设置

在 git 项目中的 setting 中的 WebHooks 中添加一个 payload url，即 Jenkins 部署的地址后加上 `/github-webhook/`

![GitHub 中 webhook 设置](https://i.loli.net/2020/05/02/ZxjuY7NSw2BdhAV.png)

然后在 Profile - setting - developer settings - Personal access tokens 可以生成一个车新的 token 为了 jenkins 访问账户使用（如果没有这一步，在之后第一次打开 jenkins blue ocean 的时候也会自动跳转到该处创建 token）。**创建完成之后会在页面上有一个 code，这个一定要保存下来后续会使用到。**

![Personal access tokens](https://i.loli.net/2020/05/02/QDKO9Ha4qJFT5nS.png)

## 在 Jenkins 中的配置

首先在 Jenkins 中添加凭据，首页 - 凭据 - 系统 - 全局凭据。然后点击 Add Credentials 即可添加凭据，

![添加凭据](https://i.loli.net/2020/05/02/jnb1N5GSQMrqCsa.png)

添加凭据的时候选择 kind 为 "Secret Text" 然后 随便起一个 ID，点击 OK Save 即可。

![添加凭据](https://i.loli.net/2020/05/02/5XA9jFnQyxHvYPU.png)

然后在 Jenkins 中添加 GitHub 服务器的信息，首页 - Manage Jenkins - Configure System 中 找到 GitHub 服务器的配置。

选择刚才创建的凭据，然后勾选上 “管理 Hook”，然后点击 “连接测试”，测试通过则表明已经创建好。

![配置 GitHub 服务器](https://i.loli.net/2020/05/02/lRfc1rNxVy3ptiT.png)

在 Jenkins 中使用的 Blue Ocean 插件，如果没有安装，可以先到 Manage Jenkins - Manage Plugins 中安装 Blue Ocean 插件。如果已经安装则登录后直接点击打开 Blue Ocean 即可。

![打开 Blue Ocean](https://i.loli.net/2020/05/02/yxWAtlPkc31MjEf.png)

会打开流水线面板，如下所示：

![Blue Ocean 首页](https://i.loli.net/2020/05/02/GwDpVlorgNeSPbz.png)

点击创建流水线，如果没有配置过 GitHub 的信息会提示要先在 GitHub 中生成 Token。

![需要创建 token](https://i.loli.net/2020/05/02/Q69lfTbnuJjmM2t.png)

如果已经创建过则会显示 GitHub 账户，点击选择账户则会提示需要使用哪个仓库创建流水线。

![选择账户创建流水线](https://i.loli.net/2020/05/02/a7loHeDyEnjcLQX.png)

创建流水线的界面如下：

![创建流水线页面](https://i.loli.net/2020/05/02/Z7iFcohSlm8HtBb.png)

按需配置即可，目前我的一个配置如下：

```jenkinsfile
pipeline {
  agent any
  stages {
    stage('Start') {
      steps {
        echo 'Process start'
      }
    }

    stage('Build ') {
      steps {
        sh 'mvn clean package -Dmaven.test.skip=true'
      }
    }

    stage('Docker') {
      steps {
        sh 'sudo docker build -f Dockerfile -t blog  .'
        sh 'sudo docker stop blog || echo "No need stop"'
        sh 'sudo docker rm blog || echo "No need remove"'
        sh 'sudo docker run -d -p 8004:8004 --name blog -t blog'
      }
    }

    stage('End') {
      steps {
        echo 'Process end'
      }
    }

  }
}
```

完成后，可以尝试本地 push 代码，然后验证一下流水线是否正常运行了。