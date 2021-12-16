1、项目总览：

50-server.cnf 是数据库 db1 的配置文件；

docker-compose.yml 是 Docker compose 配置文件；

MDockerfile 用于构建 db1 容器的来源镜像；

SDockerfile 用于构建 db2 容器的来源镜像；

MycatDockerfile用于构建 mycat 容器的来源镜像；

Mycat-server-1.6.7.5.tar.gz 是 Mycat 的 程 序 文 件；

supervisord.conf 是 supervisor 程序的配置文件，其中定义了启动 mycat 服务的指令。由于启动 mycat 的指令运行后会退出，为使容器保持启动，使用supervisor 启动服务，保持容器不退出；

schema.xml 和 server.xml是 mycat 的配置文件，其中前者定义集群中的服务器，后者定义 mycat 用户、密码等信息。

2、50-server.cnf文件内容如下：

```yaml
[server] 
[mysqld] 
user = mysql 
pid-file = /run/mysqld/mysqld.pid 
socket = /run/mysqld/mysqld.sock 
basedir = /usr 
datadir = /var/lib/mysql 
tmpdir = /tmp 
lc-messages-dir = /usr/share/mysql 
query_cache_size = 16M 
log_error = /var/log/mysql/error.log 
server-id = 2 
log_bin = mysql-bin 
expire_logs_days = 10 
binlog_ignore_db = mysql 
character-set-server = utf8mb4 
collation-server = utf8mb4_general_ci 
[embedded] 
[mariadb] 
[mariadb-10.3]
```

50-server.cnf 是 db1 的主配置文件，其中定义了服务的端口、日志文件类型等配置。

3、docker-compose.yml文件内容如下：

```yaml
version: "3.8" 
services: 
  db1: 
    image: ubuntu:db1 
    build: 
      context: . 
      dockerfile: MDockerfile 
    networks: 
      - db_network 
  db2: 
    image: ubuntu:db2 
    build: 
      context: . 
      dockerfile: SDockerfile 
    networks: 
      - db_network 
  mycat: 
    image: ubuntu:mycat 
    build: 
      context: . 
      dockerfile: MycatDockerfile 
    ports: 
      - 8066:8066 
      - 9066:9066 
    networks: 
      - db_network 
networks: 
  db_network: 
    driver: bridge
```

集群中共 3 个服务，db1、db2 和mycat。

其中 db1 为主服务器，服务的来源镜像由 MDockerfile 构建，服务所在的网络是所定义的名为 db_network 的桥接网络。db2 与db1类似。

mycat 是读写分离集群中数据库代理，用户通过访问该服务获得数据库服务，服务的来源镜像是 MycatDockerfile，服务所在的网络是db_network。

8066 端口提供了数据库服务，9066 端口是 mycat 的管理端口。

4、MDockerfile文件内容如下：

```yaml
FROM ubuntu:20.04 
MAINTAINER lxc 
COPY sources.list /etc/apt 
RUN apt-get update && apt-get dist-upgrade -y \ 
    && apt-get install mariadb-server -y 
COPY 50-server.cnf /etc/mysql/mariadb.conf.d/50-server.cnf 
RUN /etc/init.d/mysql start \ 
    && mysql -u root -e "create user 'root'@'%' identified by '123456'" \ 
    && mysql -u root -e "grant all on *.* to 'root'@'%'" \ 
    && mysql -u root -e "create database student_manager" \ 
    && mysql -u root -e "create table student_manager.student (id int primary key auto_increment, name varchar(20))" \ 
    && mysql -u root -e "insert into student_manager.student values(null, 'zhangsan')" 
EXPOSE 3306 
CMD /usr/bin/mysqld_safe
```

MDockerfile 中定义了由 ubuntu 镜像生成所需的 db1 镜像的配置，容器启动后要执行的指令是 /usr/bin/mysqld_safe。

5、SDockerfile文件内容如下：

```yaml
FROM ubuntu:20.04 
MAINTAINER lxc 
COPY sources.list /etc/apt 
RUN apt-get update && apt-get dist-upgrade -y \ 
    && apt-get install mariadb-server -y \ 
    && sed -i s/bind-address/#bind-address/g /etc/mysql/mariadb.conf.d/50-server.cnf \ 
    && /etc/init.d/mysql start \ 
    && mysql -u root -e "change master to master_host='db1', 
master_user='root',master_password='123456'" \ 
    && mysql -u root -e "start slave" 
EXPOSE 3306 
CMD /usr/bin/mysqld_safe
```

SDockerfile 用于构建镜像 db2，其中的操作与 MDockerfile 类似；

不同的是，其中执行了 change master 语句和 start slave 语句，前者的功能是指定主服务器地址及用户密码配置，后者的功能是启动主从服务。

6、MycatDockerfile文件内容如下：

```yaml
FROM ubuntu:20.04 
MAINTAINER lxc 
ADD sources.list /etc/apt 
ADD Mycat-server-1.6.7.5.tar.gz /root 
RUN apt-get update && apt-get dist-upgrade -y \ 
    && apt-get install openjdk-8-jdk -y \ 
    && mkdir /root/mycat/logs 
RUN apt-get install supervisor -y 
ADD schema.xml /root/mycat/conf 
ADD server.xml /root/mycat/conf 
ADD supervisord.conf /etc/supervisord.conf 
CMD ["/usr/bin/supervisord"] 
```

MycatDockerfile 用于构建镜像 mycat。拷贝 Mycat-server-1.6.7.5.tar.gz 到/root 目录下，ADD 拷贝 tar 文件时会自动解压，解压的路径是 /root/mycat/；

由于mycat 依赖于 java，故通过 apt install openjdk-8-jdk -y 安装 java；

mycat 运行时需要有 logs 目录，故使用 mkdir 创建 logs 目录；

最后将 schema.xml、server.xml、supervisord.conf 拷贝到容器的对应路径中。

7、schema.xml文件内容如下：

```xml
<?xml version="1.0"?> 
<!DOCTYPE mycat:schema SYSTEM "schema.dtd"> 
<mycat:schema xmlns:mycat="http://io.mycat/"> 
    <schema name="USERDB" checkSQLschema="true" sqlMaxLimit="100" dataNode="dn1"></schema> 
    <dataNode name="dn1" dataHost="localhost1" database="student_manager" /> 
    <dataHost name="localhost1" maxCon="1000" minCon="10" balance="3" dbType="mysql" dbDriver="native" writeType="0" switchType="1" slaveThreshold="100"> 
        <heartbeat>select user()</heartbeat>
        <writeHost host="hostM1" url="db1:3306" user="root" password="123456">
            <readHost host="hostS1" url="db2:3306" user="root" password="123456" />
        </writeHost>
    </dataHost>
</mycat:schema>
```

schema.xml 中定义了集群中的服务器；

其中USERDB 是 mycat 的一个逻辑库，对应所指定的数据库；

writeHost 节点定义了写服务器的配置，其中 url 是服务器地址，此处为 db1:3306；

user、password 分别为 db1 数据库的用户名、密码；

readHost 指定写服务器的配置，与 writeHost类似。

8、server.xml文件内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?> 
<!DOCTYPE mycat:server SYSTEM "server.dtd"> 
<mycat:server xmlns:mycat="http://io.mycat/"> 
<system>
    <property name="nonePasswordLogin">0</property>
    <property name="ignoreUnknownCommand">0</property> 
    <property name="useHandshakeV10">1</property> 
    <property name="removeGraveAccent">1</property> 
    <property name="useSqlStat">0</property> 
    <property name="useGlobleTableCheck">0</property> 
    <property name="sqlExecuteTimeout">300</property> 
    <property name="sequenceHandlerType">1</property> 
    <property name="sequnceHandlerPattern">(?:(\s*next\s+value\s+for\s*MYCATSEQ_(\w+))(,|\)|\s)*)+</property>
    <property name="subqueryRelationshipCheck">false</property> 
    <property name="sequenceHanlderClass">io.mycat.route.sequence.handler.HttpIncrSequenceHandler</property>
    <property name="processorBufferPoolType">0</property> 
    <property name="handleDistributedTransactions">0</property> 
    <property name="useOffHeapForMerge">0</property> 
    <property name="memoryPageSize">64k</property> 
    <property name="spillsFileBufferSize">1k</property> 
    <property name="useStreamOutput">0</property> 
    <property name="systemReserveMemorySize">384m</property> 
    <property name="useZKSwitch">false</property> 
    <property name="strictTxIsolation">false</property> 
    <property name="parallExecute">0</property>
</system> 
<user name="root">
    <property name="password">123456</property>
    <property name="schemas">USERDB</property> 
    <property name="usingDecrypt">0</property> 
</user> 
</mycat:server>
```

server.xml 中定义 mycat 用户、密码等信息。

其中 user name 为用户连接 mycat 使用的用户，password 为密码，schema 是在 schema.xml 中定义的逻辑库。

9、supervisord.conf文件内容如下：

```yaml
[supervisord] 
nodaemon=true 
[program:mycat] 
command=/bin/bash /root/mycat/bin/mycat start 
```

supervisord.conf 是 supervisor 服务的配置文件，其中定义了启动 mycat 的指令。

supervisor是一个进程管理程序，它可以将一个指令变为服务，并监控服务状态。

将启动 mycat 的指令通过 supervisor 启动后，指令不退出，可以保持容器不关闭。

10、构建镜像，使用指令 docker-compose build 根据配置文件构建镜像。

11、启动服务，使用指令 docker-compose up -d 启动服务并使服务在后台运行。

12、通过访问宿主机的 8066 端口使用数据库服务，查询数据，并插入一条数据。

由于mycat 假定在生产环境中均处于内网，因此不支持 mysql 8 客户端默认加密登录方式，需要添加参数--default-auth=mysql_native_password，方可登录。

```sql
mysql --default-auth=mysql_native_password -uroot -p123456 -P8066 -h127.0.0.1
show databases;
use USERDB;
show tables;
select * from student; 
insert into student values(null, 'lisi');
select * from student;
```

13、通过访问宿主机的 9066 端口查看读写分离集群运行情况。

```sql
mysql --default-auth=mysql_native_password -uroot -p123456 -P9066 -h127.0.0.1 -e "show @@datasource"
```

从运行结果上看，写的负载（WRITE_LOAD）都在 db1 上，读的负载（READ_LOAD）都在 db2 上，说明读写分离集群正常运行。
