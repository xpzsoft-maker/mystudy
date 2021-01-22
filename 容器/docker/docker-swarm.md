# docker-swarm

## 1. Node
```sh
//创建manager节点
docker swarm int --advertise-addr 192.168.0.xxx
```
```sh
//添加节点
docker swarm join --token SWMTKN-1-52r633lzxwaziqu4tvrwdoh7g0wj5n7p0nvxuxrfsb84d5lei6-92k632ku7jf49358ua3ejhcee 192.168.0.233:2377
```
```sh
//删除节点
docker swarm leave --force
```
## 2. 引用文献
1. [基于SpringCloud开发的微服务在Docker Swarm集群中跨Host主机通信的一种解决方案](https://blog.csdn.net/alaska_bibi/article/details/78414922?locationNum=4&fps=1
)
2. [Docker - 使用Swarm和compose部署服务](https://blog.51cto.com/lookingdream/2087077)