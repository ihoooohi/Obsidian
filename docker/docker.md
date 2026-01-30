
## Image & Container

👉 **容器是由镜像创建出来的**

`镜像  --->  docker run  --->  容器`

一张镜像可以创建 **多个容器**：

```bash
docker run nginx
docker run nginx
docker run nginx //这里的nginx是镜像名

```

每个都是**独立的容器**，互不影响。

## docker命令

docker network create = 给容器拉一条“内部局域网”

```bash
docker network create myne

docker run -d --name redis --network myne redis
docker run -d --name app --network myne myapp

docker exec -it app ping redis

```