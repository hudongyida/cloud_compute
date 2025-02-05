# docker 
## docker概念
Linux的轻量化虚拟化技术

docker三要素:
1. 镜像
   容器的存储文件，可以一容器的方式的运行，可以推送到仓库进行存储
2. 容器
   带有隔离性质的程序运行环境，又通过镜像运行后得到
3. 仓库
   存储镜像文件的地方，仓库又可以分成docker官方仓库和私有仓库

容器镜像分成的概念:
容器技术是基于Linux内核的虚拟化技术实现的，及容器的最底层为linux kernel内核
```
bootfs[kernel]
```
docker根据不同的系统，制作不同的基础镜像，这些基础镜像都是给予bootfs制作的
```
rootfs[ubuntu,rhel,centos]
bootfs[kernel]
```
开发人员会在基础镜像上继续封装不同的功能，然后打包成不同的镜像
```
image[vim,java]
rootfs[ubuntu,rhel,centos]
bootfs[kernel]
```
```
container[容器实际操作层]
image[MC]
image[vim,java]
rootfs[ubuntu,rhel,centos]
bootfs[kernel]
```

## docker 安装
安装步骤来自https://docs.docker.com/
1. 设置docker 的yum源地址
```shell
dnf -y install dnf-plugins-core
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
2. 安装下面软件 docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```shell
dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
3. 安装完成后启动docker 
```shell
systemctl enable --now docker
```

## docker 加速仓库
编辑下面文件/etc/docker/daemon.json*没有就创建*
"加速地址"
```json
{
  "registry-mirrors": ["https://docker.1ms.run"]
}
```
## docker 命令体系
### 基础命令
```shell
systemctl Options docker ##重启，查看，停止等系列命令
docker --help ##查看帮助命令
```
### 镜像命令
使用命令对docker 镜像进行操作
增:
```shell
docker pull  ##仓库中下载指定的镜像，tag为版本号，默认为latest
```
删:
```shell
docker rmi -f 镜像名称 ##从本地删除指定的镜像文件，-f为强制删除参数
docker image rm -f 镜像名称 ##功能同上
```

查
```shell
docker images -p ##查看镜像列表,等同于下面的,-p参数只显示容器ID
docker image ls 

docker search 镜像名称 ##从仓库查找指定镜像

docker system df ##查看镜像/容器、/卷所占用的空间
```

**虚悬镜像:**容器名称和TAG均为 none

docker images查看结果
REPOSITORY                      TAG       IMAGE ID       CREATED       SIZE
镜像名称                        标签/版本   镜像ID         编辑时间       大小

### 容器命令
- docker run 系列命令【启动容器命令】
  docker run 【选项】 【参数】镜像/镜像ID 启动一个容器

  在添加和运行容器的时候可通过不同的 参数 选项来事项不同的效果
  docker run 
  选项:
  -d 在后台运行
  -i 交互式操作
  -t 终端
  --name 命名
  -v 挂载数据卷

- docker ps
  docker ps 【选项】【参数】
  查了正在运行的容器
  选项:
  -a 查看正在运行的容器+停止运行的容器
  -l 显示最近创建的容器
  -n 显示最近穿穿件的n个容器
  -q 只显示容器编号

- 进入/退出交互式容器命令
  1. 使用exec -it 容器ID bashshell 可新开一个进程进入容器，并且使用exit退出容器的时也不会导致容器停止
  2. 使用attach 容器ID 可以进入刚刚打开的进程，但是使用exit退出容器的时候可能会导致容器停止
 
  1. 使用exit命令退出，但是该命令可能会导致容器缺乏前台守护进程而且自动退出
  2. 使用ctrl+p+q 退出容器，该命令不会退出当前进程

***docker 在启动时必须要前台守护进程，如果没有前台守护进程则会导致docker启动失败***

- 容器的其他操作
   |---|---|
   |启动已经停止的容器|docker start 容器ID|
   |重启容器|docker restart 容器ID|
   |停止容器|docker stop 容器ID|
   |删除已经停止的容器|docker rm 容器ID |
   |查看容器日志|docker logs 容器ID|
   |查看容器内运行进程|docker top 容器ID|
   |查看容器内部细节|docker inspect 容器ID|
   **删除和停止容器可以在命令后加-f 强制执行**

- 复制容器内的文件到主机上
  docker cp 容器ID:容器内的路劲 目的主机路径

- 导入和导出容器
  将整个容器的内容导出为tar包
  docker export 容器ID > 包名.tar

  将tar包的内容导入到容器内
  cat 包名.tar | docker import - 镜像用户/镜像名称:镜像版本号

- 容器推送命令
  推送到阿里云 dockerHUB等公有仓库
```shell
docker login --username=用户名 仓库地址
docker tag [镜像ID] 仓库地址:[镜像版本号]
docker push 仓库地址:[镜像版本号]
```
  推送到私有自建仓库
```shell
##修改镜像标签
docker tag 镜像名称:版本 地址:端口/镜像名称:版本号
##推送镜像到仓库
docker push 地址:端口/镜像名称:版本号
##curl命令查看
curl -XGET http://仓库地址:5000/v2/_catalog
```
- commit命令
  将正在运行/修改后的容器打包成一个新的镜像
  docker commit 选项 容器名/ID 镜像名称

  选项:
  -a 提交的作者信息
  -c 使用Dockerfile指令来创建镜像
  -m 提交时的说明文字
  -p 提交镜像前暂停容器【默认为true】


## docker 卷
在docker启动的时候，在docker run 的时候在后面添加-v选项
docker run --privileged=true -v /宿主机路径:/容器内路径:权限 镜像名称
--privileged给予容器特殊权限，几乎全部的访问权限，允许SELinux放行容器的全部
权限有ro rw两种写法，rw为默认

docker 挂载数据卷有两种挂载方式
默认卷挂载 在宿主机路径 不加/
非默认挂载 在宿主机路径 采用绝对路径的编写格式

卷规则继承:
及容器2可以继承容器1的卷挂载规则
docker run --privileged=true --volumes-from 父容器名 --name 子容器/新容器名称 镜像名称


## docker file
已知构建镜像可以使用commit命令实现，但若是对容器操作过多并不方便我们快速构建镜像
故我们可以采用dockerfiel文件来构建镜像

示例:
```dockerfile
FROM centos
LABEL version="1.0" description="centos7" by="测试"
ENV MYPATH /usr/local
WORKDIR $MYPATH

RUN yum -y install java-1.8.0-openjdk#设置容器时间与宿主机时间同步
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' >/etc/timezone
EXPOSE 80

CMD echo "------success------OK------"
```
|指令|描述|
|---|---|
|FROM|基础镜像|
|LABEL|描述信息|
|ENV|设置环境变量|
|ARG|使用构建时的变量|
|WOEKDIR|设置工作目录，使用/bin/bash进入容器后所在的默认目录|
|RUN|在构建容器的时候使用的命令，默认为shell解析器|
|EXPOSE|暴露的端口信息|
|USER|设置用户和组ID|
|VOLUME|创建挂载卷|
|ADD|添加本地或远程目录到容器内|
|COPY|复制文件和目录到容器内|
|CMD|指定默认命令|
|ENTRYPOINT|指定可执行的文件|

ENTRYPOINT参数可以和CMD命令配合使用，同时存在ENTRYPOINT和CMD命令的时候CMD命令会做完ENTRYPOINT命令的参数在后面执行

CMD命令可以以shell和exec命令的格式来制定CMD命令的执行

## docker 网络
### 网络类型
***注意网络的工作模式只有1-4种***
1. bridge网桥模式【容器的默认模式】
   在创建容器的时候会自动分配一个IP，并且和   docker0 网络镜像链接
2. host 主机模式
   在创建容器的时候不会自动分配IP地址，而是使用宿主机的IP和端口
3. none 
   有独立的网络信息，但是没用网络配置，需要我们自己去进行网络配置
4. container 
   新创建的容器不会创建自己的网卡配置，而是和指定的容器共享IP
5. 自定义

## 网络工作细节
bridge：
在容器创建的时候会自动加入bridge网络，该网络不带DNS解析服务，并且会自动分配IP，每次分配的IP地址都是随机的
容器内的eth0会和主机中的veth进行对应
容器bridge的中的概念等同于网络中的NAT的概念，存在端口映射和地址转换
```
 B  B  B
 E  E  E
 |  |  |
*V**V**V**
* bridge *
****I*****
```

host:
在创建容器的时候不会配置网络信息，并且不存在NAT和地址转换，所有的网络配置同主机
如果配置改网络，在run 容器的时候使用-p参数则会出现告警

```
 H  B  B
 E  E  E
 |  |  |
*|**V**V**
*|bridge *
*I********
```

none:
在创建容器的时候仅会出现lo 网络127.0.0.1环回网络

```
 B  N  B
 E  E  E
 |     |
*V*****V**
* bridge *
****I*****
```

container: 
在创建容器的时候需要指定绑定的容器，网络IP地址配置和需要指定绑定的容器相同，但若需要指定绑定的容器的被删除，或停止工作时，则容器网络中断
```
 B  B  C
 E  E--E
 |  |  
*V**V**V**
* bridge *
****I*****
```

自定义网络:
同bridge网络，但和bridge网络有所区别
1. 自定义网络有自动 DNS 解析
2. 可以将多个不同的网络段进行隔离
3. 网桥上可以配置共享环境变量

## docker compose
**更多请参考 https://docs.dockerd.com.cn/reference/compose-file/**

docker compose安装
在已经安装好了docker的基础上执行下面命令
```shell
yum install -y docker-compose-plugin
```

1. 创建一个工作目录
2. 在工作目录中定义一套服务/compose文件
3. 编写compose.yaml
4. docker compse up   
示例文件  
```yaml
# yaml 配置实例
version: '3'
services:
  web:
    build: .
    ports:
    - "5000:5000"
    volumes:
    - .:/code
    - logvolume01:/var/log
    links:
    - redis
  redis:
    image: redis
volumes:
  logvolume01: {}
```

docker compose参数表
### 版本/名称 顶级元素:
|名称|含义|
|---|---|
|version|已弃用属性，该属性用于向后兼容版本|
|name|设置显示未设置项目名称的项目|
```yaml
version: '3'
name: myapp

services: 
  foo: 
    image: busybox
    command: echo "Im runnning"
```

### 服务顶级元素
示例:
```yaml
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"

  db:
    image: postgres:13
    environment:
      POSTGRES_USER: example
      POSTGRES_DB: exampledb
```
|名称|含义|
|---|---|
|build|从指定的源代码创建容器镜像的构建配置|
|command|同dockerfile的CMD|
|container_name|指定自定义容器名称|
|expose|从容器公开的端口|
|network_mode|网络模式|
|networks|网络|
|aliases|别名|
|ipv4_address、ipv6_address|加入网络的时候指定的IP地址|
|ports|用于容器的公开端口|
|volumes|挂载卷的类型|
|working_dir|工作目录 等于Dockerfile 中的WORKDIR|


### 网络顶级元素
```yaml 
services:
  proxy:
    build: ./proxy
    networks:
      - frontend
  app:
    build: ./app
    networks:
      - frontend
      - backend
  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
    # Specify driver options
    driver: bridge
    driver_opts:
      com.docker.network.bridge.host_binding_ipv4: "127.0.0.1"
  backend:
    # Use a custom driver
    driver: custom-driver
```
|名称|含义|
|---|---|
|driver|指定网络使用哪个驱动程序|
|name|自定义网络名称|

### 卷顶级元素
示例:
```yaml
services:
  backend:
    image: example/database
    volumes:
      - db-data:/etc/data

  backup:
    image: backup-service
    volumes:
      - db-data:/var/lib/backup/data

volumes:
  db-data:
```
|名称|含义|
|---|---|
|driver|指定卷的驱动程序|
|labels|标签|
|name|自定义卷名称|
