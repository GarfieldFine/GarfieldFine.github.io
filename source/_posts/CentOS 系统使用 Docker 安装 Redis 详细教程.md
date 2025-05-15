---
title: CentOS 系统使用 Docker 安装 Redis 详细教程
date: 2025-04-23 20:37:53
top_group_index: 1
categories: Linux
tags: [Linux, Docker, Redis]
cover: https://background-img-1.oss-cn-beijing.aliyuncs.com/0B3B23DEE145DE8077A20489700EE0DD.jpeg
---
# **CentOS 系统使用 Docker 安装 Redis 详细教程**

Redis 是一个高性能的键值数据库，广泛应用于缓存、消息队列等场景。本教程将指导你在 CentOS 系统上使用 Docker 安装并配置 Redis 6.2.6 版本。

------

## **1. 准备工作**

### **1.1 安装 Docker**

如果你的系统尚未安装 Docker，请先执行以下命令安装：

```
# 卸载旧版本（如有）
yum remove docker \
    docker-client \
    docker-client-latest \
    docker-common \
    docker-latest \
    docker-latest-logrotate \
    docker-logrotate \
    docker-engine \
    docker-selinux 
```

------

首先要安装一个yum工具

```Bash
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```



安装成功后，执行命令，配置Docker的yum源（已更新为阿里云源）：

```Bash
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

sudo sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
```



更新yum，建立缓存

```Bash
sudo yum makecache fast
```



最后，执行命令，安装Docker

```Bash
yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```



启动和校验

```Bash
# 启动Docker
systemctl start docker

# 停止Docker
systemctl stop docker

# 重启
systemctl restart docker

# 设置开机自启
systemctl enable docker

# 执行docker ps命令，如果不报错，说明安装启动成功
docker ps
```

## **2. 创建 Redis 数据目录和配置文件**

### **2.1 创建目录**

Redis 需要存储数据和配置文件，我们先创建对应的目录：

```
mkdir -p /root/redis/conf   # 存放 Redis 配置文件
mkdir -p /root/redis/data   # 存放 Redis 持久化数据
```

- **`mkdir -p`**：递归创建目录，即使父目录不存在也会自动创建。
- **`/root/redis/conf`**：存放 Redis 配置文件（`redis.conf`）。
- **`/root/redis/data`**：存放 Redis 的 AOF/RDB 持久化数据。

### **2.2 创建 Redis 配置文件**

```
touch /root/redis/conf/redis.conf
```

- **`touch`**：创建一个空文件（`redis.conf`），稍后可以编辑它来自定义 Redis 配置。

------

## **3. 使用 Docker 运行 Redis**

### **3.1 拉取 Redis 6.2.6 镜像**

```
docker pull redis:6.2.6
```

- **`docker pull`**：从 Docker Hub 下载指定版本的 Redis 镜像（这里选择 `6.2.6`）。

### **3.2 启动 Redis 容器**

```
docker run \
  --restart=always \
  --log-opt max-size=100m \
  --log-opt max-file=2 \
  -p 6379:6379 \
  --name redis \
  -v /root/redis/conf/redis.conf:/etc/redis/redis.conf \
  -v /root/redis/data:/data \
  -d redis:6.2.6 \
  redis-server /etc/redis/redis.conf \
  --appendonly yes \
  --requirepass 123456
```

#### **逐条解释：**

| 参数                                                       | 作用                                                         |
| :--------------------------------------------------------- | :----------------------------------------------------------- |
| **`--restart=always`**                                     | 容器崩溃或服务器重启时自动重新启动 Redis                     |
| **`--log-opt max-size=100m`**                              | 限制单个日志文件最大 100MB                                   |
| **`--log-opt max-file=2`**                                 | 最多保留 2 个日志文件（避免日志占用过多磁盘）                |
| **`-p 6379:6379`**                                         | 将宿主机的 `6379` 端口映射到容器的 `6379` 端口（Redis 默认端口） |
| **`--name redis`**                                         | 给容器命名为 `redis`，方便管理                               |
| **`-v /root/redis/conf/redis.conf:/etc/redis/redis.conf`** | 挂载自定义配置文件到容器内                                   |
| **`-v /root/redis/data:/data`**                            | 挂载数据目录，确保 Redis 数据持久化                          |
| **`-d redis:6.2.6`**                                       | 指定使用 `redis:6.2.6` 镜像并在后台运行                      |
| **`redis-server /etc/redis/redis.conf`**                   | 启动 Redis 服务，并加载指定的配置文件                        |
| **`--appendonly yes`**                                     | 开启 AOF 持久化（确保数据安全）                              |
| **`--requirepass 123456`**                                 | 设置 Redis 访问密码（建议修改为复杂密码）                    |

------

## **4. 验证 Redis 是否运行成功**

### **4.1 检查容器状态**

```
docker ps
```

```
CONTAINER ID   IMAGE         COMMAND                  STATUS         PORTS                    NAMES
a1b2c3d4e5f6   redis:6.2.6   "docker-entrypoint.s…"   Up 5 minutes   0.0.0.0:6379->6379/tcp   redis
```

- **`STATUS`** 显示 `Up` 表示 Redis 正在运行。

### **4.2 测试 Redis 连接**

```
# 进入 Redis 容器
docker exec -it redis redis-cli

# 输入密码
AUTH 123456

# 测试写入和读取数据
SET test "Hello, Redis!"
GET test
```

- 如果返回 `"Hello, Redis!"`，说明 Redis 正常运行。

------

## **5. 自定义 Redis 配置（可选）**

如果你需要修改 Redis 的默认配置（如最大内存、持久化策略等），可以编辑 `/root/redis/conf/redis.conf`：

```
vim /root/redis/conf/redis.conf
```

常见配置项：

```
bind 0.0.0.0  # 允许远程访问
maxmemory 1gb  # 限制最大内存
save 900 1     # RDB 持久化策略
```

修改后，重启 Redis 容器生效：

```
docker restart redis
```

------

## **6. 总结**

### **关键步骤回顾**

1. **安装 Docker**（如果未安装）。

2. **创建 Redis 配置和数据目录**：

   ```
   mkdir -p /root/redis/{conf,data}
   touch /root/redis/conf/redis.conf
   ```

3. **启动 Redis 容器**（带持久化和密码）：

   ```
   docker run ...（见完整命令）
   ```

4. **验证 Redis 是否正常运行**：

   ```
   docker exec -it redis redis-cli
   AUTH 123456
   SET test "Hello"
   GET test
   ```

### **注意事项**

- **密码安全**：`--requirepass` 建议使用更复杂的密码，而非 `123456`。

- **数据备份**：定期备份 `/root/redis/data` 目录，防止数据丢失。

- **防火墙**：如果远程访问 Redis，需开放 `6379` 端口：

  ```
  sudo firewall-cmd --add-port=6379/tcp --permanent
  sudo firewall-cmd --reload
  ```

------

## **7. 常见问题**

### **Q1: 如何升级 Redis 版本？**

```
# 停止并删除旧容器
docker stop redis
docker rm redis

# 拉取新版本并重新运行
docker pull redis:7.0.0
docker run ...（更新命令中的版本号）
```

### **Q2: 如何查看 Redis 日志？**

```
docker logs redis
```

### **Q3: 如何彻底卸载 Redis？**

```
# 删除容器
docker stop redis
docker rm redis

# 删除镜像
docker rmi redis:6.2.6

# 删除数据目录（谨慎操作！）
rm -rf /root/redis
```
