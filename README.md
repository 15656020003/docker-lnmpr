### 使用dockerfile 部署 lnmp+redis 环境 

### docker 简介
docker 可以运行于任何安装现在linux 内核 的x64主机上，推荐内核版本3.8及以上

### 为什么使用docker
1. 加速本地的开发和构建流程，容器可以在开发环境构建，然后轻松地提交到测试环境，并最终进入生产环境
2. 能够在让独立的服务或应用程序在不同的环境中得到相同的运行结果
3. 创建隔离的环境来进行测试
4. 高性能、超大规划的宿主机部署


### 各环境安装Docker
**windows 7 安装**

https://segmentfault.com/n/1330000008349110

**mac**
 
使用docker toolbox https://github.com/widuu/chinese_docker/blob/master/installation/mac.md 参考使用该文章

**linux**

```
sudo yum update
sudo yum install docker
#安装程序将docker程序安装到/usr/bin⺫⽬目录下，配置⽂文件安装在/etc/sysconfig/docker。安装好docker之后，可以 将docker加⼊入到启动服务组中 
sudo systemctl enable docker.service
#手动启动docker服务器，使⽤用命令 sudo systemctl start docker.service
```

目录结构 

```
docker_lnmpr
├── mysql
│   └── Dockerfile
	└── mysqld.cnf
├── nginx
│   ├── Dockerfile
│   ├── nginx.conf
│   └── vhost
│       └── www.texixi.com.conf
├── php
│   ├── Dockerfile
│   ├── composer.phar
│   ├── php-fpm.conf
│   ├── php.ini
│   ├── redis-3.0.0.tgz
└── redis
    └── Dockerfile
    └── redis.conf
```

### 建立镜像与容器
```
# build
docker build -t centos/nginx:v1.11.5 -v /www:/www -v /data:/data  ./nginx
docker build -t centos/mysql:v5.7 -v /data/mysql:/var/lib/mysql -v /data/logs/mysql:/var/log/mysql  ./mysql
docker build -t centos/php:v7.0.12 -v /www:/www -v /data:/data  ./php
docker build -t centos/redis:v3.2.6 -v /data:/data  ./redis

#备注：这里选取了172.172.0.0网段，也可以指定其他任意空闲的网段
docker network create --subnet=172.171.0.0/16 docker-at

# run
docker run --name mysql57 --net docker-at --ip 172.171.0.9 -d -p 3306:3306 -v /data/mysql:/var/lib/mysql -v /data/logs/mysql:/var/log/mysql -v /data/run/mysqlmysqld:/var/run/mysqld  -e MYSQL_ROOT_PASSWORD=123456 -it centos/mysql:v5.7
docker run --name redis326 --net docker-at --ip 172.171.0.10 -d -p 6379:6379  -v /data:/data -it centos/redis:v3.2.6
docker run --name php7 --net docker-at --ip 172.171.0.8 -d -p 9000:9000 -v /www:/www -v /data:/data --link mysql57:mysql57 --link redis326:redis326 -it centos/php:v7.0.12 
docker run --name nginx11 --net docker-at --ip 172.171.0.7 -p 80:80 -d -v /www:/www -v /data:/data --link php7:php7 -it centos/nginx:v1.11.5 
```


### 常用命令
* docker start 容器名（容器ID也可以）
* docker stop 容器名（容器ID也可以）
* docker run 命令加 -d 参数，docker 会将容器放到后台运行
* docker ps 正在运行的容器
* docker logs --tail 10 -tf 容器名    查看容器的日志文件,加-t是加上时间戳，f是跟踪某个容器的最新日志而不必读整个日志文件
* docker top 容器名 查看容器内部运行的进程
* docker exec -d 容器名 touch /etc/new_config_file  通过后台命令创建一个空文件
* docker run --restart=always --name 容器名 -d ubuntu /bin/sh -c "while true;do echo hello world; sleep 1; done" 无论退出代码是什么，docker都会自动重启容器，可以设置 --restart=on-failure:5 自动重启的次数
* docker inspect 容器名   对容器进行详细的检查，可以加 --format='{(.State.Running)}' 来获取指定的信息
* docker rm 容器ID  删除容器，注，运行中的容器无法删除
* docker rm `docker ps -a -q` 这样可以删除所有的容器
* docker images 列出镜像
* docker pull 镜像名:标签 拉镜像
* docker search  查找docker Hub 上公共的可用镜像 
* docker build -t='AT/web_server:v1'  命令后面可以直接加上github仓库的要目录下存在的Dockerfile文件。 命令是编写Dockerfile 之后使用的。-t选项为新镜像设置了仓库和名称:标签
* docker login  登陆到Docker Hub，个人认证信息将会保存到$HOME/.dockercfg, 
* docker commit -m="comment " --author="AT"  容器ID 镜像的用户名/仓库名:标签 不推荐这种方法，推荐dockerfile
* docker history 镜像ID 深入探求镜像是如何构建出来的
* docker port 镜像ID 端口    查看映射情况的容器的ID和容器的端口号，假设查询80端口对应的映射的端口
* run 运行一个容器，  -p 8080:80  将容器内的80端口映射到docker宿主机的某一特定端口，将容器的80端口绑定到宿主机的8080端口，另 127.0.0.1:80:80 是将容器的80端口绑定到宿主机这个IP的80端口上，-P 是将容器内的80端口对本地的宿主机公开
* http://docs.docker.com/reference/builder/ 查看更多的命令
* docker push 镜像名 将镜像推送到 Docker Hub
* docker rmi 镜像名  删除镜像
* docker attach 容器ID   进入容器

**删除所有容器和镜像的命令**

```
docker rm `docker ps -a |awk '{print $1}' | grep [0-9a-z]` 删除停止的容器
docker rmi $(docker images | awk '/^<none>/ { print $3 }')
```

**进入容器的命令**

```
[root@iZ287mq5dooZ nginx]# docker inspect --format "{{ .State.Pid }}" 54a454b827e5(容器ID)
20426
[root@iZ287mq5dooZ nginx]# nsenter --target 20426 --mount --uts --ipc --net --pid
[root@bcb14764a7a3 /]#
```

### dockerfile 语法
* MAINTAINER  标识镜像的作者和联系方式
* EXPOSE 可以指定多个EXPOSE向外部公开多个端口，可以帮助多个容器链接
* FROM   指令指定一个已经存在的镜像
* \#号代表注释
* RUN 运行命令,会在shell 里使用命令包装器 /bin/sh -c 来执行。如果是在一个不支持shell 的平台上运行或者不希望在shell 中运行，也可以 使用exec 格式 的RUN指令
* ENV REFRESHED_AT 环境变量 这个环境亦是用来表明镜像模板最后的更新时间
* VOLUME 容器添加卷。一个卷是可以 存在于一个或多个容器内的特定的目录，对卷的修改是立刻生效的，对卷的修改不会对更新镜像产品影响，例:VOLUME["/opt/project","/data"]
* ADD 将构建环境 下的文件 和目录复制到镜像 中。例 ADD nginx.conf /conf/nginx.conf 也可以是取url 的地址文件，如果是压缩包，ADD命令会自动解压、
* USER 指定镜像用那个USER 去运行
* COPY 是复制本地文件，而不会去做文件提取（解压包不会自动解压） 例：COPY conf.d/ /etc/apache2/  将本地conf.d目录中的文件复制到/etc/apache2/目录中

### docker-compose.yml 语法说明


### 问题点整理
* 注意挂载目录的权限问题，不然容器成功启动几秒后立刻关闭，例：以下/data/run/mysql 目录没权限的情况下就会出现刚才那种情况
		```
		docker run --name mysql57 -d -p 3306:3306 -v /data/mysql:/var/lib/mysql -v /data/logs/mysql:/var/log/mysql -v /data/run/mysql:/var/run/mysqld -e MYSQL_ROOT_PASSWORD=123456 -it centos/mysql:v5.7
		```

* 需要注意php.ini 中的目录对应  mysql 的配置的目录需要挂载才能获取文件内容，不然php连接mysql失败

		```
		# php.ini
		mysql.default_socket = /data/run/mysql/mysqld.sock
		mysqli.default_socket = /data/run/mysql/mysqld.sock
		pdo_mysql.default_socket = /data/run/mysql/mysqld.sock
		
		# mysqld.cnf
		pid-file       = /var/run/mysqld/mysqld.pid
		socket         = /var/run/mysqld/mysqld.sock
		
		```

* 宿主机 无法访问docker的容器（nginx） https://segmentfault.com/q/1010000008893871，个人提问与回答都在里面

* 使用php连接不上redis 
	```
	# 错误的
	$redis = new Redis;
	$rs = $redis->connect('127.0.0.1', 6379);
	
	```
	
	php连接不上，查看错误日志
	```
	PHP Fatal error:  Uncaught RedisException: Redis server went away in /www/index.php:7
	```
	考虑到docker 之间的通信应该不可以用127.0.0.1 应该使用容器里面的ip，所以查看redis 容器的ip
	```
	[root@iZ287mq5dooZ php]# docker ps 
	CONTAINER ID        IMAGE                  COMMAND                   CREATED             STATUS              PORTS                         NAMES
	5fb4b1904f1c        centos/nginx:v1.11.5   "/usr/local/nginx/sbi"    About an hour ago   Up About an hour    0.0.0.0:80->80/tcp, 443/tcp   nginx11
	2bf7ad9f44f9        centos/php:v7.0.12     "/usr/local/php/sbin/"    About an hour ago   Up About an hour    0.0.0.0:9000->9000/tcp        php7
	4b84858ea4e4        centos/redis:v3.2.6    "/bin/sh -c '\"/usr/lo"   18 hours ago        Up About an hour    0.0.0.0:6379->6379/tcp        redis326
	158c67aa178c        centos/mysql:v5.7      "docker-entrypoint.sh"    6 days ago          Up About an hour    0.0.0.0:3306->3306/tcp        mysql57
	[root@iZ287mq5dooZ php]# docker inspect 4b84858ea4e4
	
	```
	结果是为 192.168.0.4，测试连接，成功
	```
	$redis = new Redis;
	$rs = $redis->connect('192.168.0.4', 6379);
	```
	问题是重启容器ip为动态的，解决该问题
	第一步：创建自定义网络
	```
	#备注：这里选取了172.172.0.0网段，也可以指定其他任意空闲的网段
	docker network create --subnet=172.171.0.0/16 docker-at
	docker run --name redis326 --net docker-at --ip 172.171.0.10 -d -p 6379:6379  -v /data:/data -it centos/redis:v3.2.6
	```
	
	连接redis 就可以配置对应的ip地址了，连接成功
	```
	$redis = new Redis;
	$rs = $redis->connect('172.171.0.10', 6379);
	```