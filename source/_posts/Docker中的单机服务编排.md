---
title: Docker中的单机服务编排
date: 2020-08-24 17:59:41
categories:
- 运维
tags: 
- docker-compose
---

#### Docker核心概念

##### 镜像

- 可以站在OOP(面向对象)的角度理解Docker镜像与容器，镜像对应OOP中的类，通过Dockerfile定义一个应用提供的各类功能，FROM关键字对应OOP里面的extend，这样可以在现有镜像的基础上扩展功能让运维工作变得可复用可扩展。镜像还有一个很重要的版本概念，镜像拥有版本之后可以让运维工作更加的规范化，最直观的好处就是可以应用上线以及版本回退更加工程化规范化。

##### 容器

- 接着上面使用OOP的思维方式来描述镜像，容器可以理解为OOP中的对象，镜像只是规范与模板容器才是真正会占用硬件资源并提供服务的运维单元。

##### 数据卷

- 有些应用是有状态的应用，比如说各类数据库，但是容器是有生命周期的可以通过镜像创建容器，同样也可以销毁容器，但是应用的状态数据不应该随着容器销毁而丢失。所以这个状态数据可以通过数据卷的方式挂载到宿主机的硬盘之中。

##### 网络

- Docker中的网络与虚拟机的网络的概念有点类似，可以通过创建虚拟网络实现服务间的网络隔离。

#### 实际项目中Docker的局限性

- 部署xxl-job

  ```shell
  # 下载镜像
  docker pull xuxueli/xxl-job-admin
  docker pull mysql
  
  # 运行镜像
  docker run --name xxl-job-mysql mysql 
  # 运行xxl-job容器并通过docker网络link到mysql容器
  docker run -p 8080:8080 -v /tmp:/data/applogs --name xxl-job-admin --link xxl-job-mysql -d xuxueli/xxl-job-admin:{指定版本}
  
  # 指定mysql地址配置
  docker run -e PARAMS="--spring.datasource.url=jdbc:mysql://127.0.0.1:3306/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai" -p 8080:8080 -v /tmp:/data/applogs --name xxl-job-admin --link xxl-job-mysql -d xuxueli/xxl-job-admin:{指定版本}
  ```

#### docker-compose的功能与实际价值 

  通过部署xxl-job我们会发现即使是部署只有两个容器的服务，也需要人工进行大量的命令行操作，首先是大量的容器配置数据难以维护，再者无法发挥Docker简单高效的优势。

  ```yaml
  version: '3.7'
  services:
  admin:
      build:
        context: .
        dockerfile: Dockerfile-admin
      ports:
        - 8080:8080
      links:
        - db:db
      depends_on:
        - db
      environment:
        PARAMS: |
          --spring.datasource.url=jdbc:mysql://db:3306/xxl_job?Unicode=true&characterEncoding=UTF-8
          --spring.datasource.username=root
          --spring.datasource.password=asdb
          --xxl.job.login.username=admin
          --xxl.job.login.password=admin
    
    db:
      build:
        context: .
        dockerfile: Dockerfile-db
      environment:
        MYSQL_ROOT_PASSWORD: asdb
      volumes:
        - ./data_dir:/var/lib/mysql
  
    job:
      build:
        context: .
        dockerfile: Dockerfile-job
      links:
        - admin:admin
      depends_on:
        - admin
      command: |
        --xxl.job.admin.addresses=http://admin:8080/xxl-job-admin
  ```

  有了上面的编排描述文件之后，只需要**docker-compose up -d**一条命令就可以做到整个应用的一键部署运行，docker-compose会自动帮我们创建一个默认的network，可以通过depends_on设定容器的以来顺序，可以将docker run时指定的参数固定到文件之中一劳永逸。引入服务编排工具之后，才能发挥出docker的真正实力，一个Dockerfile只能实现单一的功能，而多个Dockerfile组合在一起之后则可以实现复杂系统应用。

#### 真正通用的服务编排该是什么样子

docker-compose虽然可以的基础的服务编排能力，也最最基础的服务伸缩能力，但是也只是局限在单机环境下。随着应用规模的进一步扩大单台机器已经无法承载复制的应用系统，在量子计算去与硬件存储技术没有突破的情况下高可用的多机部署是唯一方案。从容器化技术兴起到现今云计算已经成为多家顶尖科技公司的核心业务。在这一演进过程中有无数技术诞生与没落，Kubernetes却依然占有统治性的地位，这与它在服务编排方面做出的贡献密切相关。