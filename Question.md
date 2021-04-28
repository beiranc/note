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
```

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

1. 用户发送请求到前端控制器 DispatcherServlet
2. DispatcherServlet 接收到请求后，调用 HandlerMapping 去获取 Handler
3. HandlerMapping 根据请求 URL 找到具体的 Handler，生成 Handler 对象及 HandlerExecutionChain （如果有则生成）一并返回给 DispatcherServlet
4. DispatcherServlet 调用 HandlerAdapter
5. HandlerAdapter 适配器调用具体的 Handler
6. Handler 执行完成后返回 ModelAndView
7. HandlerAdapter 将 Handler 执行完成后返回的 ModelAndView 返回给 DispatcherServlet
8. DispatcherServlet 将 ModelAndView 传给 ViewResolver 视图解析器进行解析
9. ViewResolver 调用当前系统中配置好的具体的视图解析器进行解析，解析后返回具体 View
10. DispatcherServlet 对 View 进行渲染（将 Model 数据填充到视图中）
11. DispatcherServlet 进行响应

----

### MyBatis 中批量插入的实现？

有三种方式。

1. 普通的使用循环来重复执行数条 INSERT 语句

    这种方式是在代码中创建一个循环来重复生成多条 INSERT 语句并执行，从而实现批量插入的。相应的 SQL 语句如下所示：

    ```XML
    <insert id="insert">
        INSERT INTO user(id, name)
        VALUES(#{id}, #{name})
    </insert>
    ```

2. MyBatis 的 Batch 模式

    这种方式不同于第一种方式的地方在于，获取 SqlSession 对象时传入一个 `ExecutorType.BATCH` 参数，这个参数会使执行器批量执行所有的更新语句（包括插入和删除）。

    > `ExecutorType` 这个枚举类型定义了三个值：
    >
    > - `ExecutorType.SIMPLE`：该类型的执行器没有特别的行为。它为每个语句的执行创建一个新的预处理语句
    > - `ExecutorType.REUSEExecutorType.BATCH`：该类型的执行器会复用预处理语句
    > - `ExecutorType.BATCH`：该类型的执行器会批量执行所有更新语句，如果 SELECT 在多个更新中间执行，将在必要时将多条更新语句分隔开来，以方便理解
    >
    > 参考：[MyBatis 的 SqlSession 说明](https://mybatis.org/mybatis-3/zh/java-api.html#sqlSessions)
    
    ```XML
    <!-- 这种方式使用的 SQL 语句与第一种方式一致 -->
    <insert id="insert">
        INSERT INTO user(id, name)
        VALUES(#{id}, #{name})
    </insert>
    ```
    
3. 使用 foreach 标签插入

     这种方式通过使用 foreach 标签来进行批量插入，这样的好处是最后生成出来的 INSERT 语句只有一条。

    ```XML
    <insert id="insertBatch">
        INSERT INTO user(id, name)
        VALUES
        <foreach collection="userList" item="user" separator=",">
            (#{user.id}, #{user.name})
        </foreach>
    </insert>
    ```

建议使用第三种方式，也就是使用 foreach 标签执行批量插入操作，因为这种方式生成的 SQL 语句仅为一条，执行速度上来说是最快的。

----

### MyBatis 如何防止 SQL 注入（`#{}` 与 `${}` 的区别）？

- `${}` 是 Properties 文件中的变量占位符，它可以用于标签属性值和 SQL 内部，属于静态文本替换，比如 `${driver}` 会被静态替换为 `com.mysql.cj.jdbc.Driver`。
- `#{}` 是 SQL 的参数占位符，MyBatis 会将 SQL 中的 `#{}` 替换为 `?` 号，在 SQL 执行前会使用 PreparedStatement 的参数设置方法，按序给 SQL 的 `?` 号占位符设置参数，例如 ps.setInt(1, parameterValue)。`#{item.value}` 的取值方式为使用反射从参数对象中获取 item 对象的 value 属性值，相当于 `param.getItem().getValue()`。

----

### 为什么说 MyBatis 是半自动 ORM 映射工具？与全自动的区别在哪？

- Hibernate 属于全自动的 ORM 映射工具，使用 Hibernate 查询关联对象或者关联集合对象时，可以根据对象关系模型直接获取，所以它是全自动的（当然 Hibernate 也支持手动编写 SQL 来完成，甚至是 HQL 或者是 Criteria 查询）。

- 而 MyBatis 在查询关联对象或关联集合对象时，需要手动编写 SQL 来完成，所以，称之为半自动 ORM 映射工具。

----

### JDBC 的使用过程？

Java 程序在使用 JDBC 接口访问关系数据库时，需要以下几步（注意使用 try-catch-finally 语句或者 try-resource 语句，目的是释放数据库资源，并且涉及到事务的操作还需要正确提交或者回滚事务）：

- 创建全局 `DataSource` 实例（即创建一个数据库连接池）
- 在需要读写数据库的方法内部，按如下步骤访问数据库：
    - 从全局数据库连接池中获取到 `Connection` 实例
    - 通过 `Connection` 实例创建 `PreparedStatement` 实例
    - 执行 SQL 语句。若为查询语句，则通过 `ResultSet` 读取结果集，若为修改，则返回的为 `int` 结果（即 SQL 语句执行后影响的行数）

CallableStatement 接口继承 PreparedStatement 接口，而 PreparedStatement 接口又继承自 Statement 接口。

CallableStatement 接口在 PreparedStatement 的基础上扩展了输出查询数据的能力，这是为了能处理存储过程及函数的执行。

PreparedStatement 接口则是在 Statement 的基础上扩展了对于参数的处理（动态插入参数，即 `?` 号占位符）。

Statement 接口则是定义了基本的 SQL 的执行。

----

### 线程池的创建（JUC 部分 -> 《Java 并发编程艺术》）？

池化技术思想主要是为了减少每次获取资源的消耗，提高对资源的利用率。例如数据库连接池、线程池、HTTP 连接池等都是对这个思想的应用。

线程池提供了一种限制和管理资源的方式。每个线程池还维护一些基本的统计信息，例如已完成任务的数量。

使用线程池的好处包括：

- 降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁所造成的消耗。
- 提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。
- 提高线程的可管理性。线程时稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一分配，调优与监控。

#### TODO

----

### Spring Cloud 的了解？



----

### 深拷贝与浅拷贝的区别？

- 深拷贝

    对基本数据类型进行值传递，对引用数据类型，创建一个新的对象，并复制其内容，即为深拷贝。

- 浅拷贝

    对基本数据类型进行值传递，对引用数据类型进行引用传递般的拷贝。

----

### IOC 与 AOP 的理解？AOP 有哪些应用场景？

#### IOC

- **IOC（Inverse of Control：控制反转）是一种设计思想，就是将原本在程序中手动创建对象的控制权，交由 Spring 框架来管理。**IOC 在其他语言中也有应用，并非 Spring 特有。IOC 容器是 Spring 用来实现 IOC 的载体，IOC 容器实际上就是个 Map，Map 中存放的为各种对象。
- 将对象之间的相互依赖关系交由 IOC 容器来管理，并由 IOC 容器完成对象的注入。这样可以很大程度上简化应用的开发，将应用从复杂的依赖关系中解放出来。**IOC 容器就像是一个工厂一样，当我们需要创建一个对象的时候，只需要配置好配置文件（或者注解）即可，完全不需要去考虑对象是如何被创建出来的。**在实际项目中一个 Service 类可能有几百甚至上千个类作为它的依赖，假设我们要实例化这个 Service，在不使用 IOC 的情况下，可能每次都需要搞清楚这个 Service 所有依赖的构造方法，这就显得十分的麻烦。但如果使用 IOC 的话，只需要配置好，然后在需要的地方引用就可以了，这大大增加了项目的可维护性，同时也降低了开发难度。

#### AOP

- AOP（Aspect-Oriented Programming：面向切面编程）能够将那些与业务无关，却为业务模块所共同调用的逻辑或责任（例如**事务处理，日志管理，权限控制**等）封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可拓展性和可维护性。
- Spring AOP 是基于动态代理的，如果要代理的对象，实现了某个接口，那么 Spring AOP 就会使用 JDK Proxy 去创建代理对象，而对于没有实现接口的对象，就无法使用 JDK Proxy 去进行代理了，这时候 Spring AOP 会采用 Cglib 生成一个被代理对象的子类来作为代理。
- 当然也可以使用 AspectJ，Spring AOP 已经集成了 AspectJ，AspectJ 应该算得上是 Java 生态系统中最完整的 AOP 框架了。
- 在使用 AOP 之后我们可以把一些通用功能抽象出来，例如**日志功能与事务管理**等场景，在需要用到的地方直接使用即可，这样大大的简化了代码量。我们需要增加新功能时也方便，这样也提高了系统扩展性。

#### Spring AOP 与 AspectJ AOP 的区别？

- **Spring AOP 属于运行时增强，而 AspectJ 是编译时增强**。Spring AOP 基于代理（Proxying），而 AspectJ 基于字节码操作（Bytecode Manipulation）。
- Spring AOP 集成了 AspectJ，AspectJ 相比于 Spring AOP 来说功能更加强大，但是 Spring AOP 相对 AspectJ 来说更简单。
- 如果切面比较少，那么两者性能差距不大。如果切面太多的话，建议使用 AspectJ，因其基于字节码，故速度会比 Spring AOP 快很多。

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

### 单例模式？



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

