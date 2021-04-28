### Nginx 负载均衡有哪几种配置？

#### 负载均衡常用算法

1. **轮询（round-robin）**

    - 轮询作为负载均衡中较为基础的也较为简单的算法，它不需要配置额外的参数。假设配置文件中共有 N 台服务器，轮询算法会遍历服务器节点列表，并按节点次序每轮选择一台服务器处理请求。当所有的节点均被调用过一次后，轮询算法则会从第一个节点开始重新开始一轮遍历。
    - 由于轮询算法中每个请求按时间顺序逐一分配到不同的服务器处理，因此适用于服务器性能相近的集群，集群中的每台服务器承载相同的负载。而对于服务器性能不同的集群，轮询算法容易引发资源分配不合理的问题。

2. **加权轮询**

    - 在加权轮询中，每个服务器会有各自的权重（`weight`），一般情况下，权重越高意味着该服务器性能越好，可以承载更多的请求。在加权轮询算法中，客户端的请求按权重值比例分配，当一个请求到达时，优先为其分配权重值最大的服务器。
    - 加权轮询算法可以应用于服务器性能不等的集群，使资源分配更加合理。

3. **IP 哈希（IP hash）**

    - IP 哈希依据发出请求的客户端 IP 的 Hash 值来分配服务器，这个算法可以保证同一个 IP 发出的请求会映射到同一台服务器，或者具有相同 Hash 值的不同 IP 映射到同一服务器。
    - IP 哈希算法能在一定程度上解决集群部署环境下 Session 不共享的问题。实际应用中，可以使用这个算法将一部分 IP 下的请求转发到运行新版本的服务器，另一部分转发到旧版本服务器上，实现灰度发布。或者，如果遇到文件过大导致请求超时的情况，也可以利用这个算法进行文件的分片上传，它可以保证相同客户端发出的文件切片转发到同一服务器上，利于其接收切片以及后续的文件合并操作。

4. **其他算法**

    - URL hash
    
        根据访问的 URL 的 Hash 值来分配请求，每个请求的 URL 会指向固定的某个服务器（在 Nginx 1.7.2 之前该算法并不是默认支持的）。
    
    - fair
    
        智能调整调度算法，动态的根据服务器的请求处理到响应的时间进行均衡分配，响应时间短处理效率高的服务器分配到请求的概率高，响应时间长处理效率低的服务器分配到的请求少（Nginx 默认不支持，若需启用需要安装 upstream_fair 模块）。
    
    - least_conn
    
        当有新的请求出现时，会遍历服务器节点列表并选取其中连接数最小的一台服务器来响应当前请求，连接数可以认为是当前该服务器处理的请求数。

#### 总结

Nginx 默认自带的负载均衡策略有三种，即 `round-robin（包括加权轮询）`，`least-connected`，`ip-hash` 等（Nginx 1.7.2 以后 `URL Hash` 也集成了进去）。

例子：

```conf
http {
    upstream loadDemo {
        server server1.example.com weight=5;
        server server2.example.com weight=5;
        server server3.example.com weight=5;
    }
    
    server {
        listen 80;
        
        location / {
            proxy_pass http://loadDemo;
        }
    }
}
```

参考链接：

[Using nginx as HTTP load balancer](http://nginx.org/en/docs/http/load_balancing.html)

[Simple Load Balancing](https://www.nginx.com/resources/wiki/start/topics/examples/loadbalanceexample)

[Nginx Docs](http://nginx.org/en/docs/)

----

### 项目部署时相关配置（待改善）？

#### Docker

由于使用了 MySQL 与 Redis，所以为了方便使用 Docker 来安装服务器上 MySQL 与 Redis 的环境。

```sh
# 安装 docker 所需软件包
sudo yum install -y yum-utils device-mapper-persistent-data lvm-2
# 设置官方源
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
# 可以安装最新版本的 Docker Engine-Community 和 containerd
# sudo yum install docker-ce docker-ce-cli containerd.io
# 也可以手动选择特定版本的 Docker Engine-Community
# 列出当前存储库中可用版本的 Docker Engine-Community
yum list docker-ce --showduplicates | sort -r
# 安装
sudo yum install docker-ce-19.03.8 docker-ce-cli-19.03.8 containerd.io
# 启动 docker
systemctl start docker
# 运行自带的 hello-world 镜像测试 docker 是否正常运行
sudo docker run hello-world
# 可以配置阿里云镜像加速器来提高拉取镜像的速度
# cd /etc/docker/
# vim daemon.json
# 加上阿里云镜像 {"registry-mirrors":["https://phkq1uwz.mirror.aliyuncs.com"]}
# 配置完成后需要重启 docker 服务
# sudo systemctl daemon-reload
# sudo systemctl restart docker
# 安装 MySQL, 首先搜索可用的 MySQL 版本
docker search mysql
# 直接拉取官方的最新的版本
docker pull mysql:latest
# 启动 MySQL 并映射 docker 容器中 MySQL 服务的端口到宿主机的端口
docker run -itd --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql
# 安装 Redis, 这里直接拉取最新版本
docker pull redis:latest
# 启动 Redis 并映射 docker 容器中 Redis 服务的端口到宿主机的端口
docker run -itd --name redis -p 6379:6379 redis
# 若存在自带 JDK 则将其移除并重新安装 JDK
java -version
rpm -qa | grep java
rpm -qa | grep jdk
rpm -qa | grep gcj
yum list java-1.8*
yum install java-1.8.0-openjdk* -y
java -version 

#### Nginx

```sh
# 首先修改 Nginx 的源, 需要预先安装 yum-utils
sudo yum install -y yum-utils
sudo vim /etc/yum.repos.d/nginx.repo
# 向 nginx.repo 文件中添加如下内容
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key

# 安装
sudo yum install -y nginx
```

#### 项目脚本

```sh
# 创建一个 nohup.out 文件
touch nohup.out
# 启动脚本 start.sh
nohup java -jar erp-server.jar --spring.profiles.active=prod &
# 停止脚本 stop.sh
PID=$(ps -ef | grep erp-server.jar | grep -v grep | awk '{ print $2 }')
if [ -z "$PID" ]
then
echo Application is already stopped
else
echo kill $PID
kill $PID
fi
# 查看日志脚本 log.sh
tail -f nohup.out
# 清理日志脚本 clean_log.sh
cp /dev/null nohup.out
# 在运行上述脚本前需要先赋予可执行的权限
chmod +x ./start.sh
chmod +x ./stop.sh
chmod +x ./log.sh
chmod +x ./clean_log.sh
```

#### Nginx 配置文件

```conf
server {
        listen       9881;
        server_name  114.55.255.214;

        #charset utf-8;
        
        # 开启 gzip
        gzip on;
        # 低于 1kb 的资源不压缩
        gzip_min_length 1k;
        # 设置压缩所需要的缓冲区大小
        gzip_buffers 4 16k;
        # 压缩级别 (1-9), 压缩级别越大压缩率越高, 同时消耗的 CPU 资源也就越多
        gzip_comp_level 6;
        # 需要压缩哪些响应类型的资源
        gzip_types text/css text/javascript application/javascript;
        # 配置禁用 gzip 的条件, 支持正则, 在这里表示 IE6 及以下不启用 gzip
        gzip_disable "MSIE [1-6]\.";
        # 是否添加 "Vary: Accept-Encoding" 响应头
        gzip_vary on;

        index index.html;
        # dist 上传的路径
        root /root/dist;
        # 避免访问出现 404 错误
        location / {
            try_files $uri $uri/ @router;
            index index.html;
        }
        location @router {
            rewrite ^.*$ /index.html last;
        }
    }
```

----

### SpringMVC 的请求处理过程？



----

### MyBatis 中批量插入的实现？



----

### MyBatis 如何防止 SQL 注入？



----

### JDBC 的使用过程？



----

### 线程池的创建？



----

### Spring Cloud 的了解？



----

### 深拷贝与浅拷贝的区别？



----

### IOC 与 AOP 的理解？AOP 有哪些应用场景？



----

### Ajax 的原理？



----

### 谈谈你对 JWT 的理解？



----

### RBAC 是什么？



----

### 权限设计方案是什么？如何实现权限控制的？



----

### 如何使用 Spring Security 来实现认证与授权？为什么不用 Shiro？



----

### 导出日志是如何导出的？



----

### 接口设计？



----

### Redis 在这个项目中的作用？



----

### 用了什么设计模式？



----

### 工厂设计模式？



----

### RESTful 风格是什么？



----

### 常用的 Linux 命令？



----

### UML 有了解过吗？说一下用例图、类图等？



----

### 单元测试有写过吗？黑盒白盒测试有了解过吗？



----

### 编写过文档吗？

