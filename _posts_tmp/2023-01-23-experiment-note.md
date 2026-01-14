---
layout: post
title: "[]"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - 
---

## PART 1

### 下载 IMDB数据集
wget ftp://ftp.fu-berlin.de/misc/movies/database/frozendata/*gz

### 使用 cinemagoer(imdbpy2sql)
因为项目改名了，现在叫cinemagoer

### 导入（包含21张表
检查一下是否遗漏

```
./imdbpy2sql.py -d ~/imdb/ -u postgresql://batina:10+10=100@localhost/batina_base
```

查看 数据库 占用空间
select pg_size_pretty(pg_database_size('batina_base'));

这里有一些 CSV，包含一个创建和导入csv的脚本，可以看一下没成功创建的 movie_info_idx
https://github.com/danolivo/jo-bench

目前猜测是，因为磁盘空间不足，脚本提前终止，没有创建这个表


这里有一份 dump 的数据，已经下好了，看看 要不要导入 pg 试一试
https://dataverse.harvard.edu/dataset.xhtml?persistentId=doi:10.7910/DVN/2QYZBT

```
# 导入 imdb 数据集
pg_restore -U batina -d test -h gp-bp11q414y016s3950o-master.gpdb.rds.aliyuncs.com  imdb_pg11  --clean --create --verbose


# 复制数据库（同一个服务器内
CREATE DATABASE new_db WITH TEMPLATE old_db;
```

// 导入好了，没那么大；1GB左右的dump文件，导入后实际占用11GB磁盘空间；可以在自己电脑上搞


该命令似乎是查看 psql 的日志目录
pg_lsclusters

### 


- 在 python 调用 jar 时，依赖找不到

  解决方法：将本地jar安装到本地maven目录，然后在 pom.xml 中引入依赖，打包时用assembly插件连同依赖一起打入jar包

  mvn install:install-file -Dfile=/home/batina/Downloads/z3-4.8.10-x64-ubuntu-18.04/bin/com.microsoft.z3.jar -DgroupId=com.microsoft -DartifactId=z3 -Dversion=1.0.0 -Dpackaging=jar


- no libz3java in java.library.path

  LD_LIBRARY_PATH=/home/batina/Downloads/z3-4.8.10-x64-ubuntu-18.04/bin java -cp equitas-1.0-jar-with-dependencies.jar:. Generator.HelloWorld

LD_LIBRARY_PATH=/home/batina/Downloads/z3-4.8.10-x64-ubuntu-18.04/bin java -cp /home/batina/sources/spes/target/equitas-1.0-jar-with-dependencies.jar:. Generator.HelloWorld



### 
explain analyze （会实际执行 sql

analyze （不带参数：更新当前数据库中所有表的统计数据；指定table：则只更新该表


### stable-baselines 3 Add gymnasium support
https://github.com/DLR-RM/stable-baselines3/pull/1327
解决类似 `AssertionError: The algorithm only supports <class 'gym.spaces.box.Box'> as action spaces but Box(-1.0, 1.0, (3,), float32) was provided` 问题。

```
pip install git+https://github.com/DLR-RM/stable-baselines3@feat/gymnasium-support
pip install git+https://github.com/Stable-Baselines-Team/stable-baselines3-contrib@feat/gymnasium-support
```


