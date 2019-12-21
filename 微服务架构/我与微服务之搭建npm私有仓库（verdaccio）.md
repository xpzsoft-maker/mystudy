## 一、问题背景
&emsp;&emsp;微服务开发时，前端往往会有一些公共组件，我们希望这些组件对内公用，但是又不能对外发布到`npm`仓库，
此时我们需要构建一个私有的前端`npm`仓库。
## 二、搭建verdaccio前端私有仓库
1. 通过`docker`现在`verdaccio/verdaccio`镜像(本文档是**`v4.3.5`**)
```sh
docker pull verdaccio/verdaccio:4.3.5
```
2. 下载`verdaccio`的`docker-examples`配置项
```sh
git clone git@github.com:verdaccio/docker-examples.git
```
包含了`docker`镜像的本地配置、git配置等等，此处我们只用`docker-local-storage-volume`
3. 将`docker-local-storage-volume`复制到任意目录，作为`verdaccio`的容器配置项和npm包存储位置
```sh
cp docker-local-storage-volume ~/Deploy/verdaccio
```
4. 修改`~/Deploy/verdaccio`中的`docker-compose.yaml`文件与`conf/config.yaml`

```yaml
version: '2.1'
services:
  verdaccio:
    image: verdaccio/verdaccio:4.3.5
    container_name: verdaccio-docker-local-storage-vol
    ports:
      - "4873:4873"
    volumes:
        - "./storage:/verdaccio/storage" # 挂载storage
        - "./conf:/verdaccio/conf" # 挂载配置项
volumes:
  verdaccio:
    driver: local
```

```yaml
#
# This is the config file used for the docker images.
# It allows all users to do anything, so don't use it on production systems.
#
# Do not configure host and port under `listen` in this file
# as it will be ignored when using docker.
# see https://github.com/verdaccio/verdaccio/blob/master/wiki/docker.md#docker-and-custom-port-configuration
#
# Look here for more config file examples:
# https://github.com/verdaccio/verdaccio/tree/master/conf
#

# path to a directory with all packages
storage: /verdaccio/storage

auth:
  htpasswd:
    file: /verdaccio/conf/htpasswd
    # Maximum amount of users allowed to register, defaults to "+inf".
    # You can set this to -1 to disable registration.
    #max_users: 1000
security:
  api:
    jwt:
      sign:
        expiresIn: 60d
        notBefore: 1
  web:
    sign:
      expiresIn: 7d

# a list of other known repositories we can talk to
uplinks:
  npmjs:
    url: https://registry.npm.taobao.org/ # 设置为淘宝镜像

packages:
  '@jota/*':
      access: $all
      publish: $all

  '@*/*':
    # scoped packages
    access: $all
    publish: $all
    proxy: npmjs

  '**':
    # allow all users (including non-authenticated users) to read and
    # publish all packages
    #
    # you can specify usernames/groupnames (depending on your auth plugin)
    # and three keywords: "$all", "$anonymous", "$authenticated"
    access: $all

    # allow all known users to publish packages
    # (anyone can register by default, remember?)
    publish: $all

    # if package is not available locally, proxy requests to 'npmjs' registry
    proxy: npmjs

# To use `npm audit` uncomment the following section
middlewares:
  audit:
    enabled: true

# log settings
logs:
  - {type: stdout, format: pretty, level: trace}
  #- {type: file, path: verdaccio.log, level: info}
```

```sh
# 查看 storage/.sinopia-db.json
cat .sinopia-db.json # 文件中存储了storage中已经存在私有npm包
```

5. 配置`verdaccio-docker-local-storage-vol`访问挂载点的权限
`verdaccio-docker-local-storage-vol`容器默认的用户为`verdaccio`，而宿主机的用户一般为`root`或者自定义的普通账户，因此容器与宿主机的UID与GID不一样。    
+ 宿主机为`root`账户
```sh
# 让容器具备访问宿主机挂载点配置的权限
sudo chown 10001:65533 conf/htpasswd # 10001是容器的UID，65533是容器的GID
sudo chown -R 10001:65533 storage # -R root权限
```

+ 宿主机为普通账户
```sh
# 让容器具备访问宿主机挂载点配置的权限
sudo chown 100:101 conf/htpasswd # 100是容器的UID，101是容器的GID
sudo chown -R 100:101 storage # -R root权限
```

6. 启动`verdaccio`
```sh
docker-compose -f docker-compose up --no-build
```

7. 访问`verdaccio` `http://宿主机IP:4873`

## 三、使用
1. 更改客户机`npm`镜像地址为私有仓库地址
```sh
npm config set registry http://宿主机IP:4873
npm install xxx # 从私有仓库下载npm包
```
2. 添加客户机注册用户
```sh
npm adduser --registry http://宿主机IP:4873
```

3. 发布npm包到私有仓库
```sh
npm publish
```