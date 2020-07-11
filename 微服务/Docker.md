---
title: "Docker 入门"
tags: ["Docker"]
categories: ["微服务"]
date: "2020-01-01T09:00:00+08:00"
---

### 基本概念

Docker **镜像**是一个只读的模板。镜像可以用来创建 Docker **容器**，一个镜像可以创建多个容器。**仓库**是存放镜像文件的地方，仓库注册服务器上存放着多个仓库。

> 镜像可以类比为 Java 的类，而容器可以类比为对象。



### Docker 工作原理

Docker 是一个 Client-Server 结构的系统，Docker 守护进程运行在主机上，然后通过 Socket 连接客户端访问，守护进程从客户端接受命令并管理运行在主机上的容器，是一个运行时环境。如下图所示：

![Docker 工作原理](https://i.loli.net/2019/06/15/5d04a77399fcc37849.png)



### Docker 与虚拟机比较

1. Docker 有更少的抽象层。不需要 Hypervisor 实现硬件资源虚拟化，运行在 Docker 容器上的程序直接使用物理机上的硬件资源，因此在 CPU、内存利用率上 Docker 都有优势。
2. Docker 利用的是宿主机的内核，而不需要 Guest OS。因此新建一个容器时不需要像虚拟机一样重新加载一个操作系统内核。所以 新建一个 Docker 只需要几秒钟的时间。

|            | Docker 容器              | 虚拟机                      |
| ---------- | ------------------------ | --------------------------- |
| 操作系统   | 与宿主机共享 OS          | 宿主机上运行虚拟机OS        |
| 存储大小   | 镜像小，便于存储与传输   | 镜像庞大                    |
| 运行性能   | 几乎无额外性能损失       | 操作系统额外的CPU、内存消耗 |
| 移植性     | 轻便、灵活、适应于 Linux | 笨重，与虚拟化技术耦合度高  |
| 硬件亲和性 | 面向开发者               | 面向硬件运维者              |
| 部署速度   | 快速，秒级               | 较慢，10s 以上              |



### 常用命令

#### 简单命令

```shell
docker version
docker info
docker -help
```

#### 镜像命令

```shell
docker images	# 列出本地镜像
-a	# 列出所有的本地所有镜像（含中间映像层）
-q	# 只显示镜像 ID
--digests	# 显示镜像摘要信息
--no-trunc	# 显示镜像完整信息
# REPOSITORY	镜像的仓库源
# TAG 	镜像的标签
# IMAGE ID	镜像 ID
# CREATED	镜像创建时间
# SIZE 镜像大小

#---
docker search <镜像名称>	# 搜索镜像
--no-trunc	# 显示完整的镜像描述
-s	# 列出收藏数不小于指定值的镜像， replace by --filter=stars=
-automated	# 仅显示 automated build 类型的镜像

#---
docker pull <镜像名称>	# 拉取镜像

#---
docker rmi <镜像名称/ID>	# 删除镜像
-f	# 强制删除
docker rmi -f $(docker images -qa)	# 强制删除全部
```

#### 容器命令

```shell
docker run [options] <image> [command][arg]	# 新建并启动容器
--name	# 为容器指定一个名称
-d	# 后天运行  启动守护式容器
-i	# 以交互模式运行 常与 t 同时使用
-t	# 为容器重新分配一个伪终端输入 常与 i 同时使用
-P	# 随机端口映射
-p: 	# 指定端口映射
		# ip:hostPort:containerPort
		# ip:containerPort
		# hostPort:containerPort
		# containerPort

#---
docker ps	# 列出所有运行的容器
-a	# 列出当前所有正在运行的容器 + 历史上运行过的容器
-l	# 显示最近创建的容器
-n	# 显示最近 n 个创建的容器
-q 	# 静默模式，只显示容器编号
--no-trunc	# 不截断输出

#---
exit	# 退出容器
ctrl p q	# 离开容器，容器不关闭

#---
docker start <容器名称/ID>	# 启动容器
docker restart <容器名称/ID>	# 重启容器
docker stop <容器名称/ID>	# 停止容器
docker kill <容器名称/ID>	# 强制停止容器
docker rm <容器ID>
-f	# 强制删除

#---
docker logs -f -t --tail <容器ID>	# 查看容器日志
-f	# 跟随最新的日志打印
-t	# 假如时间戳
--tail <数字>	# 显示最后多少条

# 后台运行，并打印日志
docker run -d centos /bin/sh -c "while true;do echo hello;sleep 2;done"

#---
docker top <容器ID>	# 查看容器内部运行的进程
docker inspect	<容器ID>	# 查看容器内部的细节

#---
docker exec -it <容器ID> <bashShell>	#重新进入正在运行的容器	/bin/bash
docker attach <容器ID>	#重新进入正在运行的容器
# 前者在容器中打开新的终端，并且可以启动新的进程，而且可以直接通过
# docker exec -it <容器ID> <bash 命令> 的方式不进入容器，直接拿到执行结果
# 后者则直接进入容器的命令终端，不会启动新的进程

#---
docker cp <容器ID>:<容器内路径> <目的主机路径>	# 容器和主机之间复制文件
```

#### 其他命令

![概览](https://i.loli.net/2019/06/15/5d04bab2461bd91179.png)



### 镜像的基本概念

镜像是一种轻量级的、可执行的独立软件包，用来打包软件运行环境和基于运行环境开发的软件，包含运行某个软件所需的所有内容，包括代码、运行时、库、环境变量和配置文件。

> **UnionFS**: Union 文件系统是一种分层，轻量级并且高性能的文件系统，它对文件系统的修改作为一次提交来一层层的叠加，同时可以将不同目录挂载到同一虚拟文件系统下。Union 文件系统是 Docker 镜像的基础。镜像可以通过分层来继承，基于基础镜像，可以制作各种具体的应用镜像。特性：一次同时加载多个文件系统，但从外面看起来，只是一个文件系统，联合加载会把各层文件系统叠加起来，这样最终的文件系统会包含所有底层的文件和目录。
>
> 参考：[Docker技术原理之Linux UnionFS（容器镜像）](https://www.jianshu.com/p/3ba255463047)

Docker 的镜像实际上是由一层一层的文件系统组成，这种层级的文件系统叫做 UnionFS。 

bootfs(Boot file system) 主要包含 bootloader 和 kernel。Bootloader 主要加载 kernel, Linux 刚启动的时候会加载 bootfs 文件系统，在 Docker 镜像的最底层是 bootfs。这一层与我们典型的 Linux/Unix 操作系统是一样的，包含 boot 加载器和内核。当 boot 加载完成之后整个内核的就在都在内存中了，此时内存的使用权已由 bootfs 转交给内核，此时系统也会卸载 bootfs.

rootfs (Root file system) 在 bootfs 之上。包含典型的 Linux 系统中的 /dev, /proc, /bin, /etc 等标准目录和文件。 rootfs 就是各种不同操作系统的发行版，比如 Ubuntu， Centos 等等。

对于一个精简的 OS， rootfs 可以只需要包括最基本的命令、工具和程序库就可以了，因为底层直接调用 Host 的 Kernel。由此可见不同  Linux 发行版的 bootfs 基本上一致的，rootfs 会有差别。

Docker 之所以采用分层镜像，主要是为了共享资源。例如多个镜像都从相同的 base 镜像构建而来，那么宿主机上只需要保存一份 Base 镜像，同时内存中也只需要加载一份 base 镜像，就可以为所有的容器服务了。与此同时，镜像的每一层都可以被共享。分层可以参考下图：

![分层镜像](https://i.loli.net/2019/06/15/5d04e5c566a6a23098.png)

另外 Docker 镜像都是只读的，当容器启动的时候，一个新的可写层会被加载到镜像顶部。这一层通常被称为 ”容器层“，容器层之下的都叫做镜像层。



### 修改镜像

可以通过 docker commit 命令提交容器副本使之成为一个新的镜像。

```shell
# 使用 tomcat 为例
# 首先下载并启动 Tomcat
docker -it -p 8888:8000 tomcat

# 查看运行的容器 ID
docker ps

# 打开新的 bash 删除 docs 目录
docker exec -it 2b26592f2e9b /bin/bash
> rm -rf /webapps/docs

# 使用 doc commit 创建新的镜像
docker commit -a="xinyue" -m="customized tomcat" 2b26592f2e9b xinyue/tomcat:1.2

# 使用新的镜像启动容器
docker run -it -p 7777:8080 xinyue/tomcat:1.2

```



### 容器卷

#### 基本概念

卷就是目录或文件，存在于一个或多个容器中，由 docker 挂载到容器，但不属于联合文件系统，因此可以绕过 UnionFS 提供的一些用于持续存储或共享数据的特性。卷的设计目的就是数据持久化，完全独立于容器的生存周期，因此 Docker 不会在容器删除时删除其挂载的数据卷。

1. 数据卷可以在容器之间共享或者重用数据。
2. 卷中的更改可以直接生效。
3. 数据卷中的更改不会包含在镜像的更新中。
4. 数据卷的生命周期一直持续到没有容器使用它为止。

#### 添加数据卷的方法

##### 1 是通过命令的方式添加

```shell
# 通过命令添加
docker run -it -v /宿主机绝对路径目录:/容器内目录 镜像名称
# 容器类的目录只可以读取
docker run -it -v /宿主机绝对路径目录:/容器内目录:ro 镜像名称

#---
docker run -it -v /demodemo:/demodemo centos

#---
docker volume create my-vol	# 创建容器卷
docker volume ls	# 查看所有容器卷
docker volume inspect my-vol	# 查看指定容器卷的信息
docker volume rm my-vol	# 删除容器卷
docker volume prune	# 清空容器卷
```

*也可以使用 mount 参数具体可以参考：[docker volume 容器卷的那些事（一）](https://deepzz.com/post/the-docker-volumes-basic.html)*

##### 2 是通过 DockerFile 添加

```shell
# 编写 dockerfile
# build docker
docker build -f dockerfile -t xinyue/centos2 .

# 启动容器
docker run -it xinyue/centos2
# 会从上一个镜像中同步过来 volumes
docker run -it --volumes-from admiring_mendel xinyue/centos2

--volumes-from # 用于容器间传递共享

#---
docker history <镜像名称> # 查看镜像历史
```

挂载结果

```json
{                                                                                                                  
    "Type": "volume",                                                                                              
    "Name": "e6e17cca6f7ff2759150a510e4e4f5130fcc26fe684011fdbd74be83b30d6ba6",                                    
    "Source": "/var/lib/docker/volumes/e6e17cca6f7ff2759150a510e4e4f5130fcc26fe684011fdbd74be83b30d6ba6/_data",    
    "Destination": "/dataVolumeContainer2",                                                                        
    "Driver": "local",                                                                                             
    "Mode": "",                                                                                                    
    "RW": true,                                                                                                    
    "Propagation": ""                                                                                              
},                                                                                                                 
{                                                                                                                  
    "Type": "volume",                                                                                              
    "Name": "f081f21bb65f5b3effd32bb8844dc84d5bc40ee291aa628ac5c59895592451a6",                                    
    "Source": "/var/lib/docker/volumes/f081f21bb65f5b3effd32bb8844dc84d5bc40ee291aa628ac5c59895592451a6/_data",    
    "Destination": "/dataVolumeContainer1",                                                                        
    "Driver": "local",                                                                                             
    "Mode": "",                                                                                                    
    "RW": true,                                                                                                    
    "Propagation": ""                                                                                              
}                                                                                                                  
```

dockerfile

```dockerfile
# volume test
FROM centos
# 创建了两个容器卷
VOLUME ["/dataVolumeContainer1", "/dataVolumeContainer2"]
CMD echo "Finished --- sucess!"
CMD /bin/bash
```



###  DockerFile

用来构建 docker 的文件。下面是一个简单的例子

```dockerfile
# scratch 相等于 Object 类
FROM scratch
ADD centos-7-x86_64-docker.tar.xz /

LABEL org.label-schema.schema-version="1.0" \
    org.label-schema.name="CentOS Base Image" \
    org.label-schema.vendor="CentOS" \
    org.label-schema.license="GPLv2" \
    org.label-schema.build-date="20190305"

# 默认命令
CMD ["/bin/bash"]
```

#### 简单语法规则

1. 每条保留字指令必须为大写字母，且后面至少有一个参数
2. 指令按照从上到下，允许执行
3. `#` 表示注释
4. 每条指令都会创建一个新的镜像层，并对镜像进行提交

#### DockerFile 执行流程如下

1. docker 从基础镜像运行一个容器
2. 执行一条指令并对容器作出修改
3. 指令类似 docker commit 的操作提交一个新的镜像层
4. docker 再基于刚提交的镜像运行一个新的容器
5. 执行 dockerfile 中的下一条指令，直到所有指令执行完成

简单来讲

1. DockFile 相等于软件原材料
2. Docker 镜像是软件交付品
3. Docker 容器是软件运行态

#### 保留关键字

```shell
FROM			# 基础镜像，当前镜像基于哪个镜像
MAINTAINER		# 作者 + 作者邮箱
RUN				# 容器构建时需要运行的命令
EXPOSE			# 暴露出的对外服务的端口号
WORKDIR			# 登录后的工作目录
ENV				# 用来在构建警醒过程中设置环境变量 eg: ENV I_PATH /usr/demo
				# 后续可以通过 $I_PATH 取出该值
ADD				# 将宿主机目录下的文件 Copy 到镜像，且会自动处理 URL 和解压缩 tar 压缩包
COPY			# 类似ADD，COPY 文件和目录到镜像中
				# 将构建上下文目录中 <源路径> 的文件/目录复制到新的一层的镜像内的 <目标路径> 位置
				# COPY src dest || COPY ["src", "dest"]
VOLUME			# 容器数据卷，用于数据保存和持久化工作
CMD				# 指定一个容器启动时要运行的命令，多个的时候只有一个生效
				# CMD 会被 docker run 之后的参数替代
ENTRYPOINT		# 同 CMD，但是不会被 docker run 之后的参数替代，而是会追加到原命令末尾
ONBUILD			# 当构建一个被继承的 DockerFile 时运行的命令
				# 父镜像的这个命令会在子镜像构建的时候执行
```



### 简单的几个例子

```dockerfile
FROM centos
ENV mypath /tmp
WORKDIR $mypath

RUN yum -y install vim
RUN yum -y install net-tools

EXPOSE 80
CMD /bin/bash
```

```shell
FROM centos
MAINTAINER xinyue<xinyue@xinyue.com>
COPY c.txt /usr/local/container.txt
ADD jdk-8u171.tar.gz /usr/local/
ADD apache-tomcat-9.tar.gz /usr/local/

RUN yum install -y vim
ENV MYPATH /usr/local
WORKDIR $MYPATH
ENV JAVA_HOME /usr/local/jdk1.8.0.1_171
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$$JAVA_HOME/lib/tools.jar
ENV CATALINA_HOME /usr/local/apache-tomcat-9
ENV CATALINA_BASE /usr/local/apache-tomcat-9
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:/$CATALINA_HOME/bin
EXPOSE 8080

# ENTRYPOINT ["/usr/local/apache-tomcat-9/bin/startup.sh"]
# CMD ["/usr/local/appache-tomcat-9/bin/catalna.sh", "run"]
CMD /usr/local/apache-tomcat-9/bin.startup.sh && tail -F /usr/local/apache-tomcat-9/bin/logs/catalina.out

```

