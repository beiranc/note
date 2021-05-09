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

### 对 Spring Cloud 的了解？



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

- Ajax 是一种异步请求数据的 WEB 开发技术，简单来说，在不需要重新刷新页面的情况下，Ajax 通过异步请求加载后台数据，并在网页上呈现出来。

- Ajax 最核心的依赖是浏览器提供的 XMLHttpRequest 对象

- Ajax 的使用

    - 创建 Ajax 核心对象 XMLHttpRequest（需要考虑兼容性）

        ```javascript
        var xhr = null;
        if (window.XMLHttpRequest) {
            // IE7+, Firefox, Chrome, Opera, Safari
            xhr = new XMLHttpRequest();
        } else {
            // IE6, IE5
            xhr = new ActiveXObject("Microsoft.XMLHTTP");
        }
        ```

    - 向服务器发送请求

        ```javascript
        // method: 请求的类型（GET, POST, PUT, DELETE）
        // url: 请求地址
        // async: 是否为异步请求
        xhr.open(method, url, async);
        // 请求体
        xhr.send(string);
        
        // 例如: 
        xhr.open("POST", "http://127.0.0.1:8080/api/user", true);
        // 设置模拟提交表单数据的请求头
        xhr.setRequestHeader("Content-type", "application/x-www-form-urlencoded");
        xhr.send("name=TestName&password=123456");
        ```

    - 处理服务器的响应

        ```javascript
        // 每当 readyState 属性发生变化时, 就会调用 onreadystatechange 函数
        xhr.onreadystatechange = function() {
            // readyState 用于标识当前 XMLHttpRequest 对象处于什么状态, 共有 5 个值
            // 0: 请求未初始化（还没有调用 open()）
            // 1: 请求已建立, 但是还没发送（还没有调用 send()）
            // 2: 请求已接收（通常在这个状态可以从响应中获取内容头）
            // 3: 请求处理中
            // 4: 请求已完成, 且响应已就绪
            if (xhr.readyState === 4) {
                if (xhr.status >= 200 && xhr.status < 300) {
                    // 响应成功处理
                    console.log(xhr.responseText);
                } else {
                    // 响应失败处理
                }
            }
        }
        ```

----

### 谈谈你对 JWT 的理解？

JWT（JSON Web Token）是目前最流行的跨域认证解决方案。

#### 跨域认证的问题

用户认证流程一般是这样：

- 用户向服务器发送用户名和密码
- 服务器验证通过后，在当前 Session 中保存相关数据，例如用户角色、登录时间等
- 服务器向用户返回一个 session_id，写入用户的 Cookie
- 用户在此之后的每一次请求，都会通过 Cookie 将 session_id 传回服务器
- 服务器收到 session_id，找到之前保存的数据，由此得知用户的身份

这种方式的问题在于，扩展性不好。在单机上的话这种方式没有任何问题，但是当部署到集群上去，或者说是跨域的服务导向架构，那么就需要 Session 数据共享，每台服务器都能够读取到 Session。

一种解决方案是 Session 数据持久化，写入数据库或者别的持久层。在各种服务收到请求后，都向持久层请求数据。这种方案的优点是架构清晰，缺点是工程量比较大。并且，如果持久层挂了，就会单点失败。

另一种解决方案是服务器干脆就不保存 Session 数据了，所有的数据都保存在客户端，每次请求都发回服务器。JWT 就是这种方案的一个代表。

#### JWT 原理

JWT 的原理是服务器认证之后，生成一个 JSON 对象，发回给用户。在这之后用户与服务端通信的时候，都要发回这个 JSON 对象。服务器完全只靠这个 JSON 对象认定用户身份。为了防止用户篡改数据，服务器在生成这个对象的时候，会加上签名。这时候服务器就不保存任何 Session 数据了，也就是说服务端变成无状态的了。

#### JWT 结构

实际的 [JWT](https://jwt.io/) 可能像这样：

```json
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

它是一个很长的字符串，中间用 `.` 分隔成三个部分。

JWT 的三个部分（Header\.Payload\.Signature）：

- 头部（Header）

    Header 部分是一个 JSON 对象，描述 JWT 的元数据。

    ```json
    {
        "alg": "HS256",
        "typ": "JWT"
    }
    ```

    如上结构中，`alg` 属性表示签名的算法（algorithm），默认是 HMAC SHA256（HS256）；`typ` 属性表示这个令牌（token）的类型（type），JWT 令牌统一写为 `JWT`。最后，将这个 JSON 对象使用 Base64URL 算法转成字符串。

- 负载（Payload）

    Payload 部分也是一个 JSON 对象，用来存放实际需要传递的数据，JWT 规定了 7 个字段供选用：

    - iss（issuer）：签发人
    - exp（expiration time）：过期时间
    - sub（subject）：主题
    - aud（audience）：受众
    - nbf（Not Before）：生效时间
    - iat（Issued At）：签发时间
    - jti（JWT ID）：编号

    当然，除了规定的 7 个字段以外，也可以在这个部分添加私有字段。但是 JWT 默认是不加密的，所以不要将秘密信息放在这个部分。这个 JSON 对象也要使用 Base64URL 算法转成字符串。

- 签名（Signature）

    Signature 部分是对前两部分的签名，防止数据篡改。

    首先，需要指定一个密钥（secret），这个密钥只有服务器才知道，不能泄露给用户，然后，使用 Header 里面指定的签名算法（默认是 HMAC SHA256），通过如下公式产生签名：

    ```json
    HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)
    ```

    算出签名之后，将 Header、Payload、Signature 三个部分拼成一个字符串，每个部分之间用 `.` 分隔，就可以返回给用户了。

    之所以使用 Base64URL 算法，是因为 JWT 作为一个令牌（token），有些场合可能会放到 URL 中去，而 Base64 有三个字符 `+`、`/`、`=` 在 URL 中有特殊含义，所以需要替换掉，将 `=` 省略，`+` 替换为 `-`，`/` 替换为 `_`。

#### 使用 JWT

客户端收到服务器返回的 JWT，可以存储在 Cookie 中也可以存在 LocalStorage 里。在此之后，客户端每次与服务端通信，都要带上这个 JWT。可以将其放在 Cookie 中自动发送，但是这样不能跨域，所以可以将其放在 HTTP 请求头中的 `Authorization` 字段中。

```http
Authorization: Bearer <token>
```

另一种做法是，跨域的时候，JWT 放在 POST 请求的数据体里。

#### JWT 的几个特点

- JWT 默认是不加密的，但也可以加密。生成原始 Token 后，可以用密钥再加密一次。
- JWT 不加密的情况下，一定不要将秘密信息写入 JWT。
- JWT 不仅可以用于认证，也可以用于交换信息，有效使用 JWT，可以降低服务器查询数据库的次数。
- JWT 最大的缺点是，由于服务器不保存 Session 状态，因此无法在使用过程中废除某个 Token，或者更改 Token 的权限，也就是说，一旦 JWT 签发了，在到期之前就会始终有效，除非服务器部署额外的逻辑。
- JWT 本身包含了认证信息，一旦泄露，任何人都可以获得该 Token 的所有权限。为了减少盗用，JWT 的有效期应该设置得比较短。对于一些比较重要的权限，使用时应该再次对用户进行认证。
- 为了减少盗用，建议使用 HTTPS 协议传输。

参考链接：[JSON Web Token](http://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html)

----

### RBAC 是什么？

RBAC（Role-based access control）即基于角色的访问控制。通过取消用户和权限的直接关联，改为用户关联角色，角色关联权限的方法来间接地赋予用户权限，从而实现解耦。

基本的 RBAC 如下图所示：

![RBAC0](/Question.assets/RBAC0.jpg)

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

#### 单例设计要点

- 一个类只能有一个实例（构造器私有化）
- 必须自行创建这个实例（含有一个该类的静态变量来保存这个唯一的实例）
- 必须自行向整个系统提供这个实例
    - 直接暴露出去
    - 用静态变量的 Get 方法获取

#### 饿汉式（直接创建对象，不存在线程安全问题）

- 直接实例化饿汉式（简洁直观）

    ```Java
    // SimpleSingleton.java
    public class SimpleSingleton {
        // 直接创建实例对象，不管调用方是否需要这个对象
        public static final SimpleSingleton INSTANCE = new SimpleSingleton();
        
        private SimpleSingleton() { }
    }
    ```

- 枚举式（最简洁）

    ```Java
    // SimpleSingleton.java
    public enum SimpleSingleton {
        INSTANCE
    }
    ```

- 静态代码块饿汉式（适合复杂实例化）

    ```Java
    // src/data.properties
    // 假设在 src 路径下有一个 data.properties 文件，其内容为 Pro=Akatsuki
    
    // 获取 Properties 文件中属性的单例
    // 适用于创建单例对象时需要获取数据的场景
    public class PropertySingleton {
        public static final PropertySingleton INSTANCE;
        
        private String pro;
        
        static {
            Properties properties = new Properties();
            try {
                properties.load(PropertySingleton.class.getClassLoader().getResourceAsStream("data.properties"));
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
            INSTANCE = new PropertySingleton(properties.getProperty("Pro"));
        }
        
        private PropertySingleton(String pro) {
            this.pro = pro;
        }
        
        public String getPro() {
            return pro;
        }
    
        public void setPro(String pro) {
            this.pro = pro;
        }
    
        @Override
        public String toString() {
            return "PropertySingleton{" +
                    "pro='" + pro + '\'' +
                    '}';
        }
    }
    ```

#### 懒汉式（延迟创建对象）

- 线程不安全（适用于单线程）

    ```Java
    // LazySingleton.java
    public class LazySingleton {
        private static LazySingleton instance;
        
        private LazySingleton() { }
        
        // 在类初始化时不直接创建实例，而是在调用方需要时去主动获取实例
        public static LazySingleton getInstance() {
            if (instance == null) {
                instance = new LazySingleton();
            }
            return instance;
        }
    }
    ```

- 线程安全（适用于多线程）

    ```Java
    // LazySingleton.java
    public class LazySingleton {
        // 这里使用 volatile 关键字主要是为了禁止指令重排序
        private volatile static LazySingleton instance;
        
        private LazySingleton() { }
        
        // 在类初始化时不直接创建实例，而是在调用方需要时去主动获取实例
        public static LazySingleton getInstance() {
            if (instance == null) {
                synchronized (LazySingleton.class) {
                    if (instance == null) {
                        instance = new LazySingleton();
                    }
                }
            }
            return instance;
        }
    }
    ```

- 静态内部类形式（适用于多线程）

    ```Java
    // LazySingleton.java
    public class LazySingleton {
        private LazySingleton() { }
        
        public static final LazySingleton getInstance() {
            return LazySingletonHolder.INSTANCE;
        }
        
        // 使用静态内部类的方式获取实例
        // 这是利用了静态内部类不会随着外部类的加载和初始化而初始化的特性来实现的
        private static class LazySingletonHolder {
            private static final LazySingleton INSTANCE = new LazySingleton();
        }
    }
    ```

----

### RESTful 风格是什么？

- REST 即表现层状态转化（Representational State Transfer），其中表现层指的是资源（Resources）的表现层。所谓资源，就是网络上的一个实体，或者说是网络上的一个具体信息，它可以是一段文本、一张图片、一首歌曲、一种服务，可以用一个 URI（统一资源定位符）去指向它，每种资源对应一个特定的 URI，要获取这个资源，只需访问它的 URI 即可。
- 资源是一种信息实体，它可以有多种外在表现形式，把资源具体呈现出来的形式，叫做它的表现层。URI 只代表资源的实体，不代表它的形式，它的具体表现形式，应该在 HTTP 请求头中用 Accept 和 Content-Type 字段指定，这两个字段才是对表现层的描述。
- 访问一个网站，就代表了客户端和服务器的一个互动过程，在这个过程中，势必涉及到数据和状态的变化。HTTP 协议是一个无状态协议，这意味着，所有的状态都保存在服务器端，因此，如果客户端想要操作服务器，必须通过某种手段，让服务器端发生状态转化（State Transfer），而这种转化是建立在表现层上的，所以称之为表现层状态转化。
- HTTP 协议里面，四个表示操作方式的动词：GET、POST、PUT、DELETE。它们分别对应四种基本操作：GET 用于获取资源；POST 用于新建资源（也可用于更新资源）；PUT 用于更新资源；DELETE 用于删除资源。
- 总结一下就是：
    - 每一个 URI 代表一种资源
    - 客户端与服务器之间，传递这种资源的某种表现层
    - 客户端通过四个 HTTP 动词，对服务器上的资源进行操作，实现表现层状态转化

参考链接：

[理解 RESTful 架构](https://www.ruanyifeng.com/blog/2011/09/restful.html)

[RESTful API 设计指南](https://www.ruanyifeng.com/blog/2014/05/restful_api.html)

----

### 常用的 Linux 命令？

ls、cat、cd、df、free、top、ps、pwd、touch、history、tar 等等。

----

### PUT 请求与 POST 请求的区别？

最大的区别是 PUT 请求具有幂等性，而 POST 请求没有。这里的幂等性指的是多次发起 PUT 请求的结果与发起一次 PUT 请求的结果应当是相同的。

-----

### `@Configuration` 与 `@Component` 有什么区别？

- `@Configuration` 中所有带 `@Bean` 注解的方法都会被动态代理，因此调用该方法返回的都是同一个实例。
- `@Configuration` 标记的类必须符合以下条件：
    - 配置类必须以类的形式提供（不能是工厂方法返回的实例），允许通过生成子类在运行时增强（CGLIB 动态代理）
    - 配置类不能是 final 类（无法动态代理）
    - 配置注解通常为了通过 `@Bean` 注解生成 Spring 容器管理的类
    - 配置类必须是非本地的（即不能在方法中声明，不能是 private）
    - 任何嵌套配置类都必须声明为 static
    - `@Bean` 方法可能不会反过来创建进一步的配置类（也就是返回的 bean 如果带有 `@Configuration` ，也不会被特殊处理，只会作为最普通的 bean）

参考链接：[@Configuration 与 @Component 区别](https://blog.csdn.net/isea533/article/details/78072133)

----

### `@Autowired` 与 `@Resource` 以及 `@Inject` 的区别？

- @Autowired 注解默认是按照类型装配依赖对象的（byType），默认情况下它要求依赖对象必须存在，当没有找到相应 bean 时，IOC 容器就会报错。若设置 @Autowired 注解的 required 属性为 false，则在找不到相应 bean 时就会注入 null，而不会报错。若需要按照名称来装配（byName），则可以结合 @Qualifier 注解一起使用。注意 @AutoWired 注解是 Spring 的注解。
- @Resource 注解默认按照名称（byName）来装配依赖对象，是 JSR-250 标准的注解，需要导入 `javax.annotation.Resource` 包。Spring 将 @Resource 注解的 name 属性解析为 bean 的名称，而 type 属性则解析为 bean 的类型，所以当使用 name 属性则使用 byName 的注入策略，而使用 type 属性则使用 byType 的注入策略。如果两个属性都不指定，则会通过反射机制使用 byName 注入策略。
- @Inject 注解默认是使用按照类型（byType）装配依赖对象，如果需要按名称进行装配，则需要结合 @Named 注解一起使用。@Inject 注解没有 required 属性，因此在找不到依赖对象时会报错。注意 @Inject 注解是 JSR-330 标准的注解，需要导入 `javax.inject.Inject` 包。
- @Autowired 与 @Inject 是使用 `org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor` 来处理的，而 @Resource 则是使用 `org.springframework.context.annotation.CommonAnnotationBeanPostProcessor` 来处理的。

----

### wait() 与 sleep() 的区别？

- 最主要的区别在于：`sleep()` 方法没有释放锁，而 `wait()` 方法释放了锁
- 两者都可以暂停线程的执行
- `wait()` 通常被用于线程间交互/通信，`sleep()` 通常被用于暂停执行
- `wait()` 方法被调用后，线程不会自动苏醒，需要别的线程调用同一个对象上的 `notify()` 或者 `notifyAll()` 方法。而 `sleep()` 方法执行完成后，线程会自动苏醒。或者可以使用 `wait(long timeout)` 超时后线程会自动苏醒。

----

### String 与 StringBuilder 以及 StringBuffer 的区别？

- 可变性
    - String 类中使用 `final` 关键字来修饰字符数组来保存字符串，所以 String 是不可变的。（在 Java9 之后，从 char 数组变成了 byte 数组）
    - StringBuilder 与 StringBuffer 都继承自 AbstractStringBuilder 类，在 AbstractStringBuilder 类中也是使用字符数组来保存字符串的，但是并没有使用 `final` 关键字来修饰，所以这两种对象都是可变的。（同样的在 Java9 之后，从 char 数组变为了 byte 数组）
- 线程安全性
    - String 中的对象是不可变的，也就可以理解为常量，故是线程安全的。StringBuffer 对方法加了同步锁或者对调用的方法加了同步锁，所以是线程安全的。StringBuilder 并没有对方法加同步锁，所以是非线程安全的。
- 性能
    - 每次对 String 类型进行修改操作时，都会生成一个新的 String 对象，然后将指针指向新的 String 对象（若字符串常量池中已有则不会生成直接指向已有的这个对象）。StringBuffer 每次都会对 StringBuffer 对象本身进行操作，而不是生成新的对象并改变对象引用。相同情况下使用 StringBuilder 比使用 StringBuffer 仅能获得 10%~15% 左右的性能提升，但是却要冒线程不安全的风险。
- 总结
    - 操作少量数据：使用 String
    - 单线程操作字符串缓冲区下操作大量数据：使用 StringBuilder
    - 多线程操作字符串缓冲区下操作大量数据：使用 StringBuffer

