# <font style="color:rgb(51, 51, 51);">Tomcat简介</font>
## JVM
```plain
Java业务都是运行在java虚拟机上，java虚拟机简称JVM( java virtual machine)
       虚拟机是通过软件模拟出具有完整硬件系统的功能
```

## JVM作用
### 为什么Java需要JVM虚拟机
      早期C语言不支持跨平台，如果C语言想要在Windows Linux Mac上运行，需要进行分别编译，那么在Linux上有很多优秀的软件，如果需要在Windows上使用需要重新编译，移植性差

而Java则不同，Java是可以跨平台，只需要将源码进行一次编译，能够在不同的操作系统运行

###    JAVA是如何做到的?
   它只需要在Windwos Linux系统上运行一个jvm，这样我们能将Java编译好的war包在Windows和Linux平台运行起来，无需我们重复编译。而JVM是由jre提供



##  JVM、JRE、JDK  
```plain
JDK
└── JRE
    └── JVM


JVM → 跑程序            运行 Java 字节码        
JRE → 提供运行环境      提供 Java 程序运行所需要的环境。里面包含 JVM 以及 Java 运行所需的类库等。
JDK → 开发 + 运行       Java 程序的开发和运行
```



## Tomcat
```plain
Tomcat和Nginx类似，都是WEB服务器软件 只不过Tomcat是基于JAVA开发的WEB服务，主要解析JAVA代码

Nginx仅支持静态资源解析，而Tomcat支持解析Java开发的WEB应用，还支持解析静态资源(效率不高)

Nginx适合做前端负载均衡，Tomcat适合做后端应用服务处理

通常情况企业会使用Nginx+Tomcat结合，Nginx处理静态资源，Tomcat处理动态资源
```



# **JAR 和 WAR**
## 作用
**JAR/WAR是为了把开发完成的 Java 应用变成可部署的应用包， 相当于把开发完成的应用“预制好”，方便交付和部署。  **

```plain
Java源码
 ↓
编译/构建
 ↓
xxx.jar / xxx.war
 ↓
JVM + Web容器
 ↓
Linux运行
```

## 区别
| <font style="color:rgb(15, 17, 21);">对比维度</font> | <font style="color:rgb(15, 17, 21);">JAR 包 (Java Archive)</font> | <font style="color:rgb(15, 17, 21);">WAR 包 (Web Application Archive)</font> |
| --- | --- | --- |
| **<font style="color:rgb(15, 17, 21);">主要用途</font>** | <font style="color:rgb(15, 17, 21);">打包通用的Java类库、工具，或是包含完整业务逻辑的可执行应用程序（如Spring Boot微服务）。</font> | <font style="color:rgb(15, 17, 21);">专门用于打包和部署Java Web应用程序，包含Servlet、JSP、HTML、CSS等所有Web资源。</font> |
| **<font style="color:rgb(15, 17, 21);">核心目录结构</font>** | <font style="color:rgb(15, 17, 21);">结构相对自由，由开发者定义。如果用于可执行JAR，则必须在META-INF/MANIFEST.MF中指定主类（Main-Class）。</font> | <font style="color:rgb(15, 17, 21);">结构由Servlet规范严格规定，必须包含WEB-INF目录。web.xml（部署描述符）、classes（编译后的类）和lib（依赖的JAR包）都在其中。</font> |
| **<font style="color:rgb(15, 17, 21);">运行与部署方式</font>** | <font style="color:rgb(15, 17, 21);">独立运行：可直接通过命令java -jar your-app.jar启动。</font> | <font style="color:rgb(15, 17, 21);">依赖容器：需要部署到外部的Servlet容器（如Tomcat、Jetty、WebLogic）中才能运行。通常将WAR文件放入容器的webapps目录，容器启动时会自动加载。</font> |
| **<font style="color:rgb(15, 17, 21);">是否需要Web容器</font>** | <font style="color:rgb(15, 17, 21);">不需要。如果使用Spring Boot等技术，JAR包本身就内置了Tomcat等Web服务器，开箱即用。</font> | <font style="color:rgb(15, 17, 21);">需要。WAR包本身不包含Web服务器，必须依赖外部容器提供的Servlet运行环境。</font> |


```plain
JAR → 通常可以直接启动
WAR → 通常交给 Tomcat 这个 Web 容器部署
```



# <font style="color:rgb(51, 51, 51);">部署Tomcat</font>
## 安装JDK
```plain
第一步: 安装JDK
两种安装方式:
第一种: 直接通过YUM仓库安装
第二种: 下载上传rpm包安装
[root@web01 ~]# rpm -ivh jdk-8u181-linux-x64.rpm
检查
[root@web01 ~]# rpm -qa |grep jdk
jdk1.8-1.8.0_181-fcs.x86_64

```

## 安装tomcat
```plain
[root@web01 ~]# wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.121/bin/apache-tomcat-9.0.121.tar.gz
解压即用:
[root@web01 ~]# mkdir /soft
[root@web01 ~]# ll /soft/
total 0
[root@web01 ~]# tar xf apache-tomcat-9.0.121.tar.gz -C /soft/
[root@web01 ~]# ll /soft/
total 0
drwxr-xr-x 9 root root 220 Sep  7 09:17 apache-tomcat-9.0.121
#创建软链接
[root@web01 ~]# ln -s /soft/apache-tomcat-9.0.121/ /soft/tomcat
[root@web01 ~]# ll /soft/
total 0
drwxr-xr-x 9 root root 220 Sep  7 09:17 apache-tomcat-9.0.121
lrwxrwxrwx 1 root root  28 Sep  7 09:18 tomcat -> /soft/apache-tomcat-9.0.121/

```



## 启动Tomcat
```plain
[root@web01 ~]# /soft/tomcat/bin/startup.sh

#检查端口是否运行 Tomcat默认端口8080
[root@web01 ~]# netstat -tnulp
...       
tcp6       0      0 :::8080                 :::*                    LISTEN      3017/java           
...


#停止Tomcat服务
[root@web01 ~]# /soft/tomcat/bin/shutdown.sh
```



<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/38996408/1788782257179-758d1247-d2b7-4147-85b7-3df92353f759.png)



## <font style="color:rgb(51, 51, 51);">配置启动方式</font>
### 配置命令
```plain
cat >/usr/lib/systemd/system/tomcat.service<<'EOF'
[Unit]
Description=Apache Tomcat Server
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
ExecStart=/soft/tomcat/bin/startup.sh
ExecStop=/soft/tomcat/bin/shutdown.sh
ExecReload=/soft/tomcat/bin/shutdown.sh && sleep 2 && /soft/tomcat/bin/startup.sh

[Install]
WantedBy=multi-user.target
EOF
```

### 含义
```plain
cat >/usr/lib/systemd/system/tomcat.service<<'EOF'
[Unit]
Description=Apache Tomcat Server             # 服务描述
After=network.target remote-fs.target nss-lookup.target  # 在服务之后运行tomcat

[Service]
Type=forking  # 以forking方式运行
ExecStart=/soft/tomcat/bin/startup.sh  # 启动命令
ExecStop=/soft/tomcat/bin/shutdown.sh  # 停止命令
ExecReload=/bin/sh -c "/soft/tomcat/bin/shutdown.sh && sleep 2 && /soft/tomcat/bin/startup.sh"  # 重载命令

[Install]
WantedBy=multi-user.target  # 运行在3级别 完全多用户
EOF
```



### 检查
```plain
#重新加载systemd服务
[root@web01 ~]# systemctl daemon-reload


#使用systemctl 运行tomcat
[root@web01 ~]# systemctl start tomcat
[root@web01 ~]# netstat -tnulp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:9000          0.0.0.0:*               LISTEN      1447/php-fpm: maste 
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      2398/nginx: master  
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1089/sshd: /usr/sbi 
tcp6       0      0 127.0.0.1:8005          :::*                    LISTEN      3586/java           
tcp6       0      0 :::8080                 :::*                    LISTEN      3586/java           
tcp6       0      0 :::22                   :::*                    LISTEN      1089/sshd: /usr/sbi 
udp        0      0 127.0.0.1:323           0.0.0.0:*                           756/chronyd         
udp6       0      0 ::1:323                 :::*                                756/chronyd  


```



# <font style="color:rgb(51, 51, 51);">tomcat目录文件</font>
```plain
tomcat软件目录结构：
​
bin           ---主要包含启动和关闭tomcat的脚本（启停java脚本依赖jar包文件）
conf            ---tomcat配置文件的目录(站点配置：server.xml)
lib           ---tomcat运行时需要加载的jar包
logs            ---tomcat日志存放位置
temp            ---tomcat临时存放文件路径
webapps       ---tomcat默认站点目录
work            ---tomcat运行时产生的缓存文件




[root@web01 conf]# cat server.xml
<?xml version="1.0" encoding="UTF-8"?>
<Server port="8005" shutdown="SHUTDOWN">			# 关闭Tomcat
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  <Listener className="org.apache.catalina.core.AprLifecycleListener" />
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener 										# 监听器-各种事件				 className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  <GlobalNamingResources>							# 全局资源配置
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>

  <Service name="Catalina">

    <Connector port="8080" protocol="HTTP/1.1"		# 连接器
               connectionTimeout="20000"
               redirectPort="8443"
               maxParameterCount="1000"
               />

    <Engine name="Catalina" defaultHost="localhost">	# 是整个Servlet容器的入口，负责接收Connector转发过来的请求，并将请求路由到对应的 Host（虚拟主机）进行处理

      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>
		# hostname 域名    appbase 站点目录
      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">	# 自动解压自动部署war包

        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"	# 日志名称
               pattern="%h %l %u %t &quot;%r&quot; %s %b" /># 日志格式

      </Host>
    </Engine>
  </Service>
</Server>

```



## <font style="color:rgb(15, 17, 21);">Tomcat Access Log 格式选项完整对照表</font>
| <font style="color:rgb(15, 17, 21);">格式</font> | <font style="color:rgb(15, 17, 21);">含义</font> | <font style="color:rgb(15, 17, 21);">示例值</font> |
| --- | --- | --- |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%a</font>` | <font style="color:rgb(15, 17, 21);">远程IP地址</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">192.168.1.100</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%A</font>` | <font style="color:rgb(15, 17, 21);">本地IP地址</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">10.0.0.7</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%b</font>` | <font style="color:rgb(15, 17, 21);">发送字节数（不含HTTP头），无则</font><font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">-</font>` | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">2048</font>`<br/><font style="color:rgb(15, 17, 21);"> </font><font style="color:rgb(15, 17, 21);">或</font><font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">-</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%B</font>` | <font style="color:rgb(15, 17, 21);">发送字节数（不含HTTP头），无则为</font><font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">0</font>` | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">2048</font>`<br/><font style="color:rgb(15, 17, 21);"> </font><font style="color:rgb(15, 17, 21);">或</font><font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">0</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%h</font>` | <font style="color:rgb(15, 17, 21);">远程主机名</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">192.168.1.100</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%H</font>` | <font style="color:rgb(15, 17, 21);">请求协议</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">HTTP/1.1</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%l</font>` | <font style="color:rgb(15, 17, 21);">远程逻辑用户名（identd）</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">-</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%m</font>` | <font style="color:rgb(15, 17, 21);">请求方法</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">GET</font>`<br/><font style="color:rgb(15, 17, 21);">、</font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">POST</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%p</font>` | <font style="color:rgb(15, 17, 21);">本地端口</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">8080</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%q</font>` | <font style="color:rgb(15, 17, 21);">查询字符串（带</font><font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">?</font>`<br/><font style="color:rgb(15, 17, 21);">）</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">?id=123</font>`<br/><font style="color:rgb(15, 17, 21);"> </font><font style="color:rgb(15, 17, 21);">或空字符串</font> |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%r</font>` | <font style="color:rgb(15, 17, 21);">请求第一行</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">GET /index.jsp?id=123 HTTP/1.1</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%s</font>` | <font style="color:rgb(15, 17, 21);">HTTP响应状态码</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">200</font>`<br/><font style="color:rgb(15, 17, 21);">、</font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">404</font>`<br/><font style="color:rgb(15, 17, 21);">、</font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">500</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%S</font>` | <font style="color:rgb(15, 17, 21);">用户会话ID</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">A1B2C3D4E5F6...</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%t</font>` | <font style="color:rgb(15, 17, 21);">日期时间（通用日志格式）</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">[07/Sep/2026:16:30:25 +0800]</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%u</font>` | <font style="color:rgb(15, 17, 21);">远程认证用户</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">admin</font>`<br/><font style="color:rgb(15, 17, 21);"> </font><font style="color:rgb(15, 17, 21);">或</font><font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">-</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%U</font>` | <font style="color:rgb(15, 17, 21);">请求URL路径</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">/index.jsp</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%v</font>` | <font style="color:rgb(15, 17, 21);">本地服务器名</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">web01.example.com</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%D</font>` | <font style="color:rgb(15, 17, 21);">处理耗时（毫秒）</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">123</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%T</font>` | <font style="color:rgb(15, 17, 21);">处理耗时（秒）</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">0.123</font>` |
| `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%I</font>` | <font style="color:rgb(15, 17, 21);">当前请求线程名</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">http-nio-8080-exec-1</font>` |




# 实验
## tomcat配置文件
### 位置
```plain
[root@web01 conf]# pwd
/soft/tomcat/conf
[root@web01 conf]# ll server.xml 
-rw------- 1 root root 9526 Sep  7 19:24 server.xml
```

### 修改内容
```plain
      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">

        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>

	  <!--增加一个主机名称zrlog 代码目录是/code/zrlog-->
      <Host name="www.zrlog.com"  appBase="/code/zrlog"	
            unpackWARs="true" autoDeploy="true">

        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="zerlog_access" suffix=".log"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>

    </Engine>
  </Service>
</Server>
```

### 重启生效
```plain
[root@web01 code]# systemctl restart tomcat

```

## 站点
### 位置
```plain
[root@web01 zrlog]# ll /code/zrlog/ROOT/
total 4
-rw-r--r-- 1 root root 7 Sep  7 10:25 index.html
```

### 部署
#### 下载代码,解压代码
```plain
unzip zrlog-2.2.1-efbe9f9-release.war
```

#### 在51服务器创建zrlog数据库
```plain
[root@db01 ~]# mysql -uroot -plzy123.com -e "create database zrlog;"
[root@db01 ~]# mysql -uroot -plzy123.com -e "show databases;"
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| wp                 |
| zh                 |
| zrlog              |
+--------------------+

```



## 扩展web02
```plain
#部署运行环境JDK
[root@web02 ~]# ll
total 166040
-rw-r--r-- 1 root root 170023183 Sep 27  2024 jdk-8u181-linux-x64.rpm
[root@web02 ~]# rpm -ivh jdk-8u181-linux-x64.rpm 

#将web01tomcat复制到web02
[root@web02 ~]# scp -r 10.0.0.7:/soft /

#复制启动方式到web02
[root@web02 ~]# scp -r 10.0.0.7:/usr/lib/systemd/system/tomcat.service /usr/lib/systemd/system/
[root@web02 ~]# systemctl daemon-reload

#复制代码到web02
[root@web02 ~]# scp -r 10.0.0.7:/code/zrlog /code/

#启动tomcat
[root@web02 ~]# systemctl start tomcat


#windows解析到web02
```



## <font style="color:rgb(51, 51, 51);">静态文件共享到NFS</font>
```plain
1.配置NFS
[root@nfs 20260907]# cat /etc/exports
/data 172.16.1.0/24(rw,sync,all_squash)
/zrlog 172.16.1.0/24(rw,sync,all_squash)

[root@nfs ~]# mkdir /zrlog
[root@nfs ~]# chown nobody.nobody /zrlog/
[root@nfs ~]# systemctl restart nfs

2.拷贝完整的静态数据到nfs
[root@web01 ROOT]# scp -r attached/image/ 10.0.0.31:/zrlog/

#NFS执行-R授权
[root@nfs ~]# chown -R nobody.nobody /zrlog/

3.挂载(WEB01和WEB02)
[root@web02 ~]# showmount -e 172.16.1.31
Export list for 172.16.1.31:
/zrlog 172.16.1.0/24
/data  172.16.1.0/24
[root@web02 ~]# mount -t nfs 172.16.1.31:/zrlog /code/zrlog/ROOT/attached/
[root@web02 ~]# df -h
Filesystem             Size  Used Avail Use% Mounted on
devtmpfs               459M     0  459M   0% /dev
tmpfs                  475M     0  475M   0% /dev/shm
tmpfs                  475M   49M  426M  11% /run
tmpfs                  475M     0  475M   0% /sys/fs/cgroup
/dev/mapper/klas-root   47G  4.8G   43G  11% /
/dev/sda1             1014M  169M  846M  17% /boot
tmpfs                   95M     0   95M   0% /run/user/0
172.16.1.31:/data       47G  3.9G   44G   9% /code/wordpress/wp-content/uploads
172.16.1.31:/zrlog      47G  3.9G   44G   9% /code/zrlog/ROOT/attached

```



## <font style="color:rgb(51, 51, 51);">接入负载均衡</font>
```plain
[root@lb01 conf.d]# cat zrlog.conf
upstream zrlog {
	server 10.0.0.7:8080;
	server 10.0.0.8:8080;
	keepalive 16;
}
server {
	listen 443 ssl;
	server_name www.zrlog.com;
	ssl_certificate   ssl_key/server.crt;
	ssl_certificate_key  ssl_key/server.key;
	 # 配置 SSL 会话缓存，提高性能
           ssl_session_cache shared:SSL:1m;
	 # 设置 SSL 会话超时时间
    	   ssl_session_timeout 5m;
	   ssl_protocols TLSv1.2 TLSv1.3;
	 # 优先使用服务端指定的加密套件
         ssl_prefer_server_ciphers on;

	location / {
	proxy_pass http://zrlog;
	include proxy;
	}
}

server {
        listen 80;
        server_name www.zrlog.com;
        return 302 https://$server_name$request_uri;
}

[root@lb01 conf.d]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@lb01 conf.d]# systemctl restart nginx
[root@lb01 conf.d]# 

```



## <font style="color:rgb(51, 51, 51);">tomcat部署会话保持</font>
### 配置web01
```plain
[root@web01 conf]# cat server.xml
...
      <Host name="www.session.com"  appBase="/code/session"
            unpackWARs="true" autoDeploy="true">

        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="session_access" suffix=".log"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>
...

[root@web01 conf]# systemctl restart tomcat
[root@web01 conf]# mkdir /code/session/ROOT

[root@web01 conf]# cat /code/session/ROOT/index.jsp
<body>
        <%
        //HttpSession session = request.getSession(true);
        System.out.println(session.getCreationTime());
        out.println("<br> web01 SESSION ID:" + session.getId() + "<br>");
        out.println("Session created time is :" + session.getCreationTime()
        + "<br>");
        %>
</body>

#windows解析
10.0.0.7  www.session.com
```

### 配置web02
```plain
配置WE02
[root@web02 conf]# vim server.xml
...
      <Host name="www.session.com"  appBase="/code/session"
            unpackWARs="true" autoDeploy="true">

        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="session_access" suffix=".log"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>

...
[root@web02 conf]# systemctl restart tomcat
[root@web02 conf]# cd /code/session/
[root@web02 session]# mkdir ROOT

[root@web02 ROOT]# cat index.jsp
<body>
        <%
        //HttpSession session = request.getSession(true);
        System.out.println(session.getCreationTime());
        out.println("<br> web02 SESSION ID:" + session.getId() + "<br>");
        out.println("Session created time is :" + session.getCreationTime()
        + "<br>");
        %>
</body>


#windows解析到web02
10.0.0.8 www.session.com

```



### <font style="color:rgb(51, 51, 51);">接入负载均衡</font>
```plain
[root@lb01 conf.d]# cat s.conf
upstream s {
	server 10.0.0.7:8080;
	server 10.0.0.8:8080;
}
server {
	listen 80;
	server_name www.session.com;

	location / {
	proxy_pass http://s;
	include proxy;
	}
}

[root@lb01 conf.d]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@lb01 conf.d]# systemctl restart nginx

#windows解析测试
10.0.0.5 www.session.com
```



## <font style="color:rgb(51, 51, 51);">配置WEB接入redis</font>
```plain
#删除插件
[root@web01 ~]# ll
total 13772
-rw-r--r-- 1 root root 13178703 Aug 13 03:55 apache-tomcat-9.0.121.tar.gz
-rw-r--r-- 1 root root   921429 Sep  7 11:41 tomcat-cluster-redis-session-manager.zip
[root@web01 ~]# unzip tomcat-cluster-redis-session-manager.zip 

#拷贝jars到tomcat的/lib目录中
[root@web01 ~]# cp tomcat-cluster-redis-session-manager/lib/* /soft/tomcat/lib/


#拷贝conf下的redis.properties文件,到tomcat的conf文件
[root@web01 ~]# cp tomcat-cluster-redis-session-manager/conf/redis-data-cache.properties /soft/tomcat/conf/


#将配置文件中连接redis地址修改为如下地址即可
[root@web01 ~]# vim /soft/tomcat/conf/redis-data-cache.properties
redis.hosts=172.16.1.51:6379
redis.password=123456

#添加如下两行至tomcat/conf/context.xml  (添加在</Context> 上一行 )
[root@web01 ~]# vim /soft/tomcat/conf/context.xml
<Valve className="tomcat.request.session.redis.SessionHandlerValve" />
<Manager className="tomcat.request.session.redis.SessionManager" />


#使用--delete同步，工作中慎重使用。
[root@web01 ~]# rsync -avz --delete /soft/tomcat/ 10.0.0.8:/soft/tomcat/


[root@web02 ROOT]# systemctl restart tomcat
```



# <font style="color:rgb(51, 51, 51);">context用法</font>
```plain
 #用户访问www.session.com/tt 实际路径是 /code/tt ,如果/code/tt不创建 tomcat起不来
 <Host name="www.session.com"  appBase="/code/session"
            unpackWARs="true" autoDeploy="true">
     <Context docBase="/code/tt" path="/tt" reloadable="true" />
	
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="session_access" suffix=".log"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>

```

