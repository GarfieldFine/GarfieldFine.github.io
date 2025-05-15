---
title: CentOS 系统使用 Docker 安装 Elasticsearch 和 Kibana 详细教程
date: 2025-04-24 15:10:08
top_group_index: 1
categories: Linux
tags: [Linux, Docker, Elasticsearch, Kibana]
cover: https://background-img-1.oss-cn-beijing.aliyuncs.com/054A321B234F9C1FA93D6EED6969A271.jpeg
---
# **CentOS 系统使用 Docker 安装 Elasticsearch 和 Kibana 详细教程**

本教程将指导你在 CentOS 系统上使用 Docker 部署 Elasticsearch 7.17.28 和 Kibana 7.17.28，并安装中文分词插件 IK Analysis。

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

---

## **2. 部署 Elasticsearch**

### **2.1 运行 Elasticsearch 容器**

```
docker run -d \
  --name es \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  -e "discovery.type=single-node" \
  -v es-data:/usr/share/elasticsearch/data \
  -v es-plugins:/usr/share/elasticsearch/plugins \
  --privileged \
  -p 9200:9200 \
  -p 9300:9300 \
  elasticsearch:7.17.28
```

#### **参数解释**

| 参数                                             | 说明                             |
| :----------------------------------------------- | :------------------------------- |
| `-d`                                             | 后台运行容器                     |
| `--name es`                                      | 容器命名为 `es`                  |
| `-e "ES_JAVA_OPTS=-Xms512m -Xmx512m"`            | 设置 JVM 堆内存为 512MB          |
| `-e "discovery.type=single-node"`                | 单节点模式（适合开发环境）       |
| `-v es-data:/usr/share/elasticsearch/data`       | 挂载数据卷，持久化存储数据       |
| `-v es-plugins:/usr/share/elasticsearch/plugins` | 挂载插件目录                     |
| `--privileged`                                   | 授予容器特权模式（某些插件需要） |
| `-p 9200:9200`                                   | 暴露 REST API 端口（HTTP）       |
| `-p 9300:9300`                                   | 暴露集群通信端口（TCP）          |
| `elasticsearch:7.17.28`                          | 指定 Elasticsearch 版本          |

------

## **3. 部署 Kibana**

### **3.1 运行 Kibana 容器**

```
docker run -d \
  --name kibana \
  -e ELASTICSEARCH_HOSTS=http://{ip}:9200 \
  -p 5601:5601 \
  kibana:7.17.28
```

#### **参数解释**

| 参数                                    | 说明                                          |
| :-------------------------------------- | :-------------------------------------------- |
| `-d`                                    | 后台运行容器                                  |
| `--name kibana`                         | 容器命名为 `kibana`                           |
| `-e ELASTICSEARCH_HOSTS=http://es:9200` | 连接 Elasticsearch 服务（`es` 是容器名）      |
| `-p 5601:5601`                          | 暴露 Kibana Web 界面端口                      |
| `kibana:7.17.28`                        | 指定 Kibana 版本（与 Elasticsearch 版本一致） |

------

## **4. 安装 IK 中文分词插件**

### **4.1 进入 Elasticsearch 容器**

```
docker exec -it es bash
```

### **4.2 安装 IK 插件**

```
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.17.28/elasticsearch-analysis-ik-7.17.28.zip
```
> 上述这个方式安装不成功的则需要去GitHub上手动安装 地址：https://release.infinilabs.com/analysis-ik/stable/
> 找到对应版本下载到本地，解压后，然后使用可视化工具finalshell直接将文件拖拽到 服务器的虚拟机的/var/lib/docker/volumes/es-plugins/_data这个目录下，然后使用docker restart es这个命令重新启动即可
#### **参数解释**

| 命令                                 | 说明                                  |
| :----------------------------------- | :------------------------------------ |
| `docker exec -it es bash`            | 进入 Elasticsearch 容器的 shell       |
| `./bin/elasticsearch-plugin install` | 安装 Elasticsearch 插件               |
| `https://...ik-7.17.28.zip`          | IK 分词器插件下载地址（版本必须匹配） |

### **4.3 重启 Elasticsearch**

```
docker restart es
```

------

## **5. 验证安装**

### **5.1 检查 Elasticsearch**

```
curl http://localhost:9200
```

正常输出示例：

```
{
  "name" : "node-1",
  "cluster_name" : "elasticsearch",
  "version" : {
    "number" : "7.17.28",
    "build_flavor" : "default",
    "build_type" : "docker",
    "lucene_version" : "8.11.1"
  }
}
```

### **5.2 检查 Kibana**

访问 `http://<服务器IP>:5601`，如果看到 Kibana 界面，说明安装成功。

### **5.3 验证 IK 分词器**

```
curl -X POST "http://localhost:9200/_analyze" -H 'Content-Type: application/json' -d'
{
  "analyzer": "ik_max_word",
  "text": "你好世界"
}'
```

输出示例（正确分词）：

```
{
  "tokens" : [
    { "token" : "你好", "start_offset" : 0, "end_offset" : 2 },
    { "token" : "世界", "start_offset" : 2, "end_offset" : 4 }
  ]
}
```

------

## **6. 常见问题**

### **Q1: 无法访问 Elasticsearch/Kibana？**

- 检查防火墙是否开放端口：

  ```
  sudo firewall-cmd --add-port=9200/tcp --permanent
  sudo firewall-cmd --add-port=5601/tcp --permanent
  sudo firewall-cmd --reload
  ```

### **Q2: IK 插件安装失败？**

- 确保下载的插件版本与 Elasticsearch 版本完全一致（这里是 `7.17.28`）。
- 如果网络问题，可以手动下载 ZIP 文件后复制到容器内安装。

### **Q3: 如何卸载？**

```
# 删除容器
docker stop es kibana
docker rm es kibana

# 删除数据卷
docker volume rm es-data es-plugins

# 删除镜像
docker rmi elasticsearch:7.17.28 kibana:7.17.28
```

------

## **7. 总结**

| 组件              | 关键配置                 | 访问方式                 |
| :---------------- | :----------------------- | :----------------------- |
| **Elasticsearch** | 单节点模式，512MB 堆内存 | `http://IP:9200`         |
| **Kibana**        | 连接 Elasticsearch       | `http://IP:5601`         |
| **IK 分词器**     | 版本必须匹配             | 通过 `_analyze` API 测试 |

**🎉 现在你已成功搭建 Elasticsearch + Kibana 环境，并支持中文分词！** 🚀
