# 使用Docker搭建citus实验环境

## 前言

搭建citus集群需要多台机器，如果仅用于功能验证而手上又没有合适机器，使用Docker搭建是个不错的选择。
下面是步骤示例（安全，性能调优相关的配置全部忽视）。

## 环境

### host环境
- CentOS 7.3
- Docker 17.12.1-ce

### guest环境
- CentOS 7.4
- PostgreSQL 10
- citus 7.2

## 制作citus镜像

### 启动centos容器

	docker run -it --name citus centos bash

### 在容器中安装citus所需软件

	yum install https://download.postgresql.org/pub/repos/yum/10/redhat/rhel-7-x86_64/pgdg-redhat10-10-2.noarch.rpm
	yum install postgresql10 postgresql10-server postgresql10-contrib
	yum install pgbouncer
	yum install citus_10

	ln -sf /usr/pgsql-10 /usr/pgsql
	cat - >~postgres/.pgsql_profile <<EOF
	
	if [ -f /etc/bashrc ]; then
	. /etc/bashrc
	fi
	export PATH=/usr/pgsql/bin:$PATH
	export PGDATA=/pgsql/data
	EOF

### 创建citus镜像

	docker commit citus citus:0.1

## 创建citus容器

### 为每一个节点创建一个volume

	docker volume create cituscn
	docker volume create cituswk1
	docker volume create cituswk2

### 创建专有子网

	docker network create --subnet=172.18.0.0/16 citus-net

### 创建并运行容器

	docker run --mount source=cituscn,target=/pgsql --network citus-net --ip 172.18.0.100 --name cituscn --hostname cituscn --expose 5432 -it citus:0.1 bash
	docker run --mount source=cituswk1,target=/pgsql --network citus-net --ip 172.18.0.201 --name cituswk1 --hostname cituswk1 --expose 5432 -it citus:0.1 bash
	docker run --mount source=cituswk2,target=/pgsql --network citus-net --ip 172.18.0.202 --name cituswk2 --hostname cituswk2 --expose 5432 -it citus:0.1 bash

在每个容器上，分别执行下面的命令创建数据库

	mkdir /pgsql/data
	chown postgres:postgres /pgsql/data
	chmod 0700 /pgsql/data
	
	su - postgres
	initdb -k -E UTF8 -D /pgsql/data
	
	echo "listen_addresses = '*'" >> /pgsql/data/postgresql.conf
	echo "wal_level = logical" >> /pgsql/data/postgresql.conf
	echo "shared_preload_libraries = 'citus'" >> /pgsql/data/postgresql.conf
	echo "host    all             all             172.18.0.0/16            trust">> /pgsql/data/pg_hba.conf
	
	pg_ctl start
	psql -c "CREATE EXTENSION citus;"


在cituscn容器上，执行下面的命令添加worker。

	psql -c "SELECT * from master_add_node('cituswk1', 5432);"
	psql -c "SELECT * from master_add_node('cituswk2', 5432);"

## 容器detach/attach

如果要退出容器的终端(detach)，可以按CTL+pq

之后需要再次交互执行命令时再attach这个容器

	docker attach cituscn

## 容器启停

以cituscn容器为例，worker容器类似。

### 停止cituscn容器

	docker stop cituscn

### 启动cituscn容器

	docker start -ai cituscn

然后在容器中启动PostgreSQL

	su - postgres
	pg_ctl start

## 测试

在cituscn容器上，测试分片表

	create table tb1(id int primary key, c1 int);
	select create_distributed_table('tb1','id');
	insert into tb1 select id,random()*1000 from generate_series(1,100)id;

执行测试SQL

	postgres=# explain select * from tb1;
	                                  QUERY PLAN                                  
	------------------------------------------------------------------------------
	 Custom Scan (Citus Real-Time)  (cost=0.00..0.00 rows=0 width=0)
	   Task Count: 32
	   Tasks Shown: One of 32
	   ->  Task
	         Node: host=cituswk1 port=5432 dbname=postgres
	         ->  Seq Scan on tb1_102008 tb1  (cost=0.00..32.60 rows=2260 width=8)
	(6 rows)





