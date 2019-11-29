## 一、问题背景
&emsp;&emsp;微服务开发时，前端往往会有一些公共组件，我们希望这些组件对内公用，但是又不能对外发布到`npm`仓库，
此时我们需要构建一个私有的前端`npm`仓库。
## 二、搭建sinopia前端私有仓库
&emsp;&emsp;[sinopia](https://github.com/xpzsoft-maker/sinopia)是开源的`npm`私有仓库，私有仓库没有的包，
将会从远程`npm`仓库中拉取，本地发布的包将默认发布到私有仓库，而不会发布到远程`npm`仓库。
1. 安装`node.js`、`npm`和`tsc`，并配置淘宝镜像源
```sh
npm config set registry https://registry.npm.taobao.org
npm config list #查看npma配置
```
2. 全局安装sinopia
```sh
npm install -g sinopia
```
3. 测试启动`sinopia`
```sh
sinopia # 启动sinopia，启动成功，会在当前目录下生成 config.yaml配置文件、htpasswd文件和storage文件夹
        # config.yaml用于配置sinopia的启动参数
        # storage文件夹中存放发布和下载的npm包
```
4. 修改`config.yaml`，添加对`@types`、`@angular`等`scope package`的支持
```yaml
#
# This is the default config file. It allows all users to do anything,
# so don't use it on production systems.
#
# Look here for more config file examples:
# https://github.com/rlidwka/sinopia/tree/master/conf
#

# path to a directory with all packages
storage: ./storage

auth:
  htpasswd:
    file: ./htpasswd
    # Maximum amount of users allowed to register, defaults to "+inf".
    # You can set this to -1 to disable registration.
    #max_users: 1000

# a list of other known repositories we can talk to
uplinks:
  npmjs:
    url: https://registry.npm.taobao.org #设置为淘宝镜像

packages:
  '@*/*':
    # scoped packages
    access: $all
    publish: $authenticated
    proxy: npmjs #添加对@types、@angular等scope package的支持

  '*':
    # allow all users (including non-authenticated users) to read and
    # publish all packages
    #
    # you can specify usernames/groupnames (depending on your auth plugin)
    # and three keywords: "$all", "$anonymous", "$authenticated"
    access: $all

    # allow all known users to publish packages
    # (anyone can register by default, remember?)
    publish: $authenticated

    # if package is not available locally, proxy requests to 'npmjs' registry
    proxy: npmjs #普通包下载代理

# log settings
logs:
  - {type: stdout, format: pretty, level: http}
  #- {type: file, path: sinopia.log, level: info}

listen: 0.0.0.0:4873 #默认没有，只能本机以localhost访问，添加后可远程访问
```
5. 重启`sinopia`，通过`http://ip:4873`访问私有库
6. 测试私有仓库
```sh
ssh 客户机 #进入一个测试用的客户机
npm config set registry http://ip:4873 #修改npm远程仓库地址为sinopia私有仓库地址
npm install jquery #测试从私有库中安装jquery，由于私有库中没有jquery，所有sinopia会从proxy:npmjs远程拉取jquery

ssh sinopia服务器
cd sinopia-home/storage #进入sinopia的启动目录，jquery已经在storage中
```
7. `sinopia`有[docker镜像](https://hub.docker.com/r/keyvanfatehi/sinopia/)