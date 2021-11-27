# 参考链接

1. [Build a Node.js and React app with npm](https://www.jenkins.io/doc/tutorials/build-a-node-js-and-react-app-with-npm/)
2. [Build a Java app with Maven](https://www.jenkins.io/doc/tutorials/build-a-java-app-with-maven/)
3. [Jenkins Offical Tutorials overview](https://www.jenkins.io/doc/tutorials/#tools/)
4. [Installing Jenkins](https://www.jenkins.io/doc/book/installing/)
5. [Hardware Recommendations](https://www.jenkins.io/doc/book/scaling/hardware-recommendations/)

----

# 前言

本文仅用于记录自用 Jenkins 的搭建过程，Jenkins 的功能也只用到了基本的 Maven 构建以及自由风格任务构建。[Pipeline](https://www.jenkins.io/doc/book/pipeline/) 及 [Blue Ocean](https://www.jenkins.io/doc/book/blueocean/) 等进阶功能未涉及。

----

# 搭建 Jenkins

Jenkins 可支持安装在多种环境下，如 Docker，K8s，Linux，macOS，Windows。这里以 Centos 为例。

官方的教程地址：[Installing Jenkins in Linux](https://www.jenkins.io/doc/book/installing/linux/)

### 硬件配置要求

|                    最低配置                     |     推荐配置     |
| :---------------------------------------------: | :--------------: |
|                256MB 的运行内存                 | 4GB+ 的运行内存  |
| 1GB 的硬盘空间（如果用 Docker 则推荐最低 10GB） | 50GB+ 的硬盘空间 |

### 运行所需环境

1. [Java 8+](https://www.jenkins.io/doc/administration/requirements/java/)，推荐安装 Java11

### Jenkins 版本选择

为了在日后的使用过程中 Jenkins 自身不出什么奇怪的 Bug，所以选择了 LTS 版本的 Jenkins 进行安装。

### [安装步骤](https://www.jenkins.io/doc/book/installing/linux/#red-hat-centos)

1. 添加 **redhat-stable** 源

    ```shell
    sudo wget -O /etc/yum.repos.d/jenkins.repo \
        https://pkg.jenkins.io/redhat-stable/jenkins.repo
    sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
    ```

2. 更新（可选）

    ```shell
    # 只更新软件包不更新系统与内核
    sudo yum upgrade
    ```

3. 安装 Java11（如果已经有了 Java 环境则这一步可省略）

    ```shell
    sudo yum install epel-release java-11-openjdk-devel
    ```

4. 安装 Jenkins

    ```shell
    sudo yum install jenkins
    ```

5. 重载修改过的配置文件

    ```shell
    sudo systemctl daemon-reload
    ```

**<span style="color: red;">注意放行端口！（Jenkins 的默认端口是 8080）</span>**

----

# Jenkins 的基本设置

- 默认安装目录：`/var/lib/jenkins`
- 默认日志目录：`/var/log/jenkins/jenkins.log`
- 默认配置目录：`/etc/sysconfig/jenkins`

在安装完成并修改好配置后，可直接使用 `systemd` 去启动 Jenkins。

启动后，打开浏览器进入 Jenkins 的配置初始化界面，由于 Jenkins 推荐的插件已经够用了，所以直接选择安装推荐的插件即可。在这之后，输入管理员用户名与密码就配置完成了。

如需外网使用，可以配置 Nginx 反代。

**插件的话我在推荐插件的基础上多装了两个插件，一个是用于传输构建好之后文件的 Publish Over SSH，另外一个是用于构建成功后邮件提醒的 Email Ext Recipients Column Plugin。**

### 提醒

由于 Jenkins 不推荐在 Master 节点（也就是当前安装并管理 Jenkins 的服务器）上执行任务，所以需要先到系统配置中将当前 Master 节点的 Worker 数量设为 0，然后创建一个工作节点。

### 创建一个工作节点

- 名称：任意
- 执行者数量：根据工作节点物理机 CPU 配置设置，假如物理机 CPU 是 4 核，那么可以设为 4
- 远程工作目录：工作节点上存储源码以及构建后结果的目录
- 用法：这个有两个选项，一个是**尽可能的使用这个节点**，另外一个是**只允许运行绑定到这台机器的Job**，这个地方看需求选，没有特殊需求的话就直接选**尽可能的使用这个节点**的选项就行
- 启动方式：选 Java Web 启动代理的话就需要在工作节点物理机上通过 JLNP 运行 agent.jar
- 节点属性：可以在这里填写工作节点上构建环境所在目录

### 系统配置

- Maven 项目配置
- SSH remote hosts 配置
- Github 钩子配置
- 邮件通知配置
- Publish over SSH 配置

### 全局工具配置

- Maven 环境配置
- JDK 环境配置
- Git 环境配置
- NodeJS 环境配置

----

#  Maven 项目的构建

1. 选择新建一个 Maven 项目，任务名可任意填写。

2. 勾选 **Github 项目**并填写项目所在地址，例如 `https://github.com/beiranc/erp-server/`。

3. 勾选 **在必要的时候并发构建**。

4. 如果在前面工作节点的用法选择的是 **只允许运行绑定到这台机器的 Job**，那么这里则需要点开 **高级**，然后勾选 **限制项目的运行节点** 并输入工作节点的名字。

5. 源码管理选择 **Git**，然后填写仓库地址，例如 `git@github.com:beiranc/erp-server.git`，注意这里需要添加认证，建议使用 SSH 认证。**注意需要选择构建的分支**。

6. 构建触发器选项卡中需要勾选 **Build whenever a SNAPSHOT dependency is built** 与 **Github hook trigger for GITScm polling**。

7. 构建环境选项卡勾选 **Add timestamps to the Console Output**

8. Build 选项卡填写 Maven 配置文件位置，默认为 `pom.xml`，然后填写 Maven 的 goals，例如：`clean package -Dmaven.test.skip=true`，后面的参数是跳过 Maven Test 的含义。

9. Post Steps 选项卡选择 **Run only if build succeeds**。这里需要做的事情有两件，一件是使用 Publish Over SSH 插件将构建完成后的 Jar 包复制到服务器；另一件是通过 SSH 连接到服务器并重启服务。

    - Send files or execute commands over SSH 配置（**注意 SSH 服务器需要提前配置好**）

        - 选择 SSH 服务器
        - 传输设置
            - Source files：`target/erp-server.jar`，填写构建完成后 Jar 包的位置
            - Remove prefix：`target`
            - Remote directory：`/root/backend/`，这个需要填写服务器上存储 Jar 包的位置

    - Execute shell script on remote host using ssh 配置（**注意 SSH site 需要提前配置好**）

        - 选择 SSH 服务器

        - 填写需要执行的命令，例如：

            ```shell
            PID=$(ps -ef | grep erp-server.jar | grep -v grep | awk '{ print $2 }')
            if [ -z "$PID" ]
            then
            echo Application is already stopped
            else
            echo kill $PID
            kill $PID
            fi
            
            echo Start Runing New Application...
            cd /root/backend
            JENKINS_SERVER_COOKIE=dontKillMe nohup java -jar erp-server.jar --spring.profiles.active=prod >> nohup.out &
            echo Application is Running...
            echo Over.
            sleep 10
            ```

----

# Vue 项目的构建

1. 创建一个自由风格的任务，名称任意。

2. 勾选 **Github 项目**，填写项目地址，例如：`https://github.com/beiranc/erp-web/`。

3. 勾选 **在必要的时候并发构建**。

4. 在源码管理选项卡中添加 Git 仓库地址，例如 `git@github.com:beiranc/erp-web.git`。

5. 构建触发器选项卡勾选 **Github hook trigger for GITScm polling**

6. 构建环境勾选 **Add timestamps to the Console Output** 以及 **Provide Node & npm bin/ folder to PATH**

    - 选择工具配置中配置好的 NodeJS 环境

7. 构建选项卡选择 **执行 Windows 批处理命令**，这里是因为我的工作节点运行在 Windows 机器上，如果工作节点运行在 Linux 上，则同理。

    - 填写 Vue 项目中的构建命令

        ```powershell
        echo ================= npm install =======================
        call npm install
        
        echo ================== npm run build:prod ===================
        call npm run build:prod
        ```

    - 增加构建步骤 **Execute shell script on remote host using ssh**（**注意 SSH site 需要提前配置好**）

        - 填写命令

            ```shell
            rm -rf /home/dist && mkdir /home/dist
            ```

8. 构建后操作添加 **Send build artifacts over SSH**

    - 选择 SSH 服务器
    - 传输配置
        - Source files：`dist/`
        - Remote directory：`/home`

----

# 总结

使用 Jenkins 的目的是将平时自己手动复制 Jar 包或者前端项目打包后的 dist 文件夹的操作省略，在我们提交后 Jenkins 自动的去执行构建、复制的操作。