#docker-compose
用于微服务镜像的构建启动

## 1. 离线安装
前往github[下载](https://github.com/docker/compose/releases)，重命名解压到/usr/local/bin

```sh
sudo mv docker-compose-Linux-x86_64 /usr/local/bin/docker-compose
chmod +x docker-compose
```

## 2. 配置镜像的构建与启动
```yml
version: "3" #有1/2/3三个版本
services:
  eureka: #服务名称
    image: 102.168.0.202:5000/eureka:20190917 #镜像tag
    hostname: xxx
    networks:
      - dockernet
    build:  #构建镜像的地址
      context: ./eureka #地址
      dockerfile: Dockerfile #地址下的dockerfile
    ports: #挂载出的端口
      - "26250:26250"
    volumes: #挂载出的文件目录
      - /var/log/eureka:/app/logs
    container_name: eureka20190917 #容器名称
    command: [ "./start-image.sh" ] #容器启动的命令
    restart: always #重启
  gateway:
    image: 102.168.0.202:5000/gateway:20190917
    build:
      context: ./gateway
      dockerfile: Dockerfile
    ports:
      - "16250:16250"
    volumes:
      - /var/log/gateway:/app/logs
    depends_on: #容器启动依赖
      - eureka
    container_name: gateway20190917
    command: [ "./wait-for-it.sh", "eureka:26250", "--", "./start-image.sh" ]
    restart: always
  platform:
    image: 102.168.0.202:5000/platform:20190917
    build:
      context: ./platform
      dockerfile: Dockerfile
    ports:
      - "46250:46250"
    volumes:
      - /var/log/platform:/app/logs
    depends_on:
      - gateway
    container_name: platform20190917
    command: [ "./wait-for-it.sh", "gateway:16250", "--", "./start-image.sh" ]
    restart: always

networks:
  dockernet:
    driver: overlay #swarm创建网桥
    # external: true 非manager节点
```
## 3. 常用命令
```sh
docker-compose -f xxx.yml up #构建docker镜像并启动容器
dokcer-compose -f xxx.yml start #启动容器
docker-compose logs #查看日志
docker-compose logs -f #实时查看日志
```
## 4. 官网文档
[yml配置项文档v3](https://docs.docker.com/compose/compose-file/)
[CLI控制台命令](https://docs.docker.com/compose/reference/overview/)