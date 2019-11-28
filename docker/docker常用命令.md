# Docker常用命令

## 1. 容器命令
```sh
docker ps // 查看所有正在运行容器 
docker stop containerId // containerId 是容器的ID 
docker ps -a // 查看所有容器 $ docker ps -a -q // 查看所有容器ID 
docker stop $(docker ps -a -q) //  stop停止所有容器 
docker rm $(docker ps -a -q) //   remove删除所有容器
docker rm -f [容器id] // 删除运行中的容器
docker docker run --restart always --detach \
  --name gateway$TAG --publish 16250:16250 \
  --volume /var/log/gateway:/app/logs \
  192.168.0.202:5000/gateway:$TAG // 启动docker镜像，创建容器
```

## 2. 镜像命令
```sh
docker images // 查看所有镜像  
docker images -a // 查看所有容器 $ docker ps -a -q // 查看所有镜像 
docker rmi [镜像id] // 删除镜像
docker build -t name . //创建镜像
```

## 3. 查看日志
```sh
docker attach --sig-proxy=false mynginx //实时查看日志
docker logs --tail="10" mytest // 查看后10条日志
``` 

## 4. Dockerfile
```sh
FROM 192.168.0.202:5000/openjdk:8
LABEL appname="gateway" version="0.0.1" author="Pengzhi Xie"
COPY gateway-0.0.1.jar /app/gateway.jar
WORKDIR /app
EXPOSE 16250
VOLUME /var/log/gateway:/app/logs
CMD java -jar -Dspring.cloud.config.profile=prod gateway.jar
```

## 5. 进入docker内部镜像
```sh
docker exec -it eureka20190904 /bin/bash
```

## 6. docker查看容器cpu占用率
```sh
docker stats
```

## 7. docker network
```sh
docker network ls
```

## 8. docker导入导出
```sh
docker save > xxx.tar IMAGE
docker load -i xxx.tar
``` 


## 9. docker 启动redis
```sh
docker run -p 6379:6379 -v $PWD/data:/data --name="redis" -d redis:3.2 redis-server --appendonly yes

-p 6379:6379 : 将容器的6379端口映射到主机的6379端口

-v $PWD/data:/data : 将主机中当前目录下的data挂载到容器的/data

redis-server --appendonly yes : 在容器执行redis-server启动命令，并打开redis持久化配置
```