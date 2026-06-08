---
title: "WSL Kind Kubernetes + CloudNative-PG 部署实战"
date: 2025-06-09
author: "Samunroyu"
tags: ["Kubernetes", "PostgreSQL", "DevOps", "WSL"]
description: "从零搭建 K8s 集群到 PostgreSQL 高可用部署的全链路记录"
---

# 背景
最近在公司需要部署 spark 等相关组件，在使用 k8s 时遇见了一些问题，希望能够有一个环境能够测试下，因此决定在 wsl 部署测试。这里记录下从零搭建、避坑调优到最终验证的全链路闭环。

# 基础环境

- **CPU**: Intel(R) Core(TM) Ultra 9 275HX
- **内存**: 32GB（分给 WSL 16GB）
- **K8s 基础组件**：
  - kind：v0.20.0
  - kubectl：v1.36.1
  - docker：v29.1.3

## 安装命令
``` shell
# 下载最新稳定版的 Kind 二进制文件 
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64 
# 赋予执行权限 
chmod +x ./kind 
# 移动到系统 path 目录下，方便全局调用 
sudo mv ./kind /usr/local/bin/kind


# 下载最新稳定版的 kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
# 赋予执行权限并移动到全局目录
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl


# docker 安装
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
## 一、 基础架构声明与避坑 YAML

kind 启动 k8s 节点
```yaml {filename="kind-prod.yaml"}
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane # 主节点（Brain）
- role: worker        # 计算节点 1
- role: worker        # 计算节点 2
```

``` shell
# 启动节点
kind create cluster --config kind-prod.yaml --name dev-cluster

# 部署完成后使用查看版本
kubectl version 

# 查看对应节点
kubectl get nodes

# 查看 k8s 节点资源
kubectl describe node dev-cluster-worker
```

接下来开始部署 postgre。在 K8s 中部署有状态服务，首要解决的是 API 版本对齐与存储、资源的规划。以下为适配当前最新版规范（以 1.29.1+ 为主）的完全体配置文件 `pg-cluster.yaml`。

### 1. 避坑关键点

- **存储字段重构**：老版本中的 `spec.storage.storageClassName` 已在最新版中废弃，**必须缩写为 `storageClass`**，否则会触发 K8s 严审机制的 `BadRequest / strict decoding error`。
    
- **本地盘平替**：在 Kind 虚拟集群中，如果不确定默认的 StorageClass 命名，推荐在测试期省略 `storageClass` 字段，让其自动匹配底层唯一的默认本地盘，防止驱动挂载陷入 `Pending` 死锁。
    

### 2. 生产级高可用清单 (`pg-cluster.yaml`)

```yaml {filename="pg-cluster.yaml"}
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: my-ha-postgres
  namespace: default
spec:
  instances: 3 # 声明一主两从（1 Primary + 2 Replicas）的工业级高可用架构

  # 存储配置
  storage:
    size: 5Gi
    # storageClass: standard # 在本地 Kind 测试时可注释掉，动用默认本地盘

  # 资源边界划分（完美契合 16G/32G 内存池宿主机）
  resources:
    limits:
      cpu: "1"
      memory: 1Gi
    requests:
      cpu: "500m"
      memory: 512Mi
```

## 二、 网络环境解决

国内直连 `ghcr.io`（GitHub Container Registry）极易发生几百兆镜像拉取长达 10 分钟、甚至因控制台超时导致 `ErrImagePull` / `ImagePullBackOff` 的惨剧。

由于 Kind 的分布式节点本质上是彼此隔离的 Docker 容器，**只在宿主机 pull 或者只给其中一个 Worker 灌入镜像，都会引发多节点“时空错配”**（例如 Pod 随机调度到了没有镜像的 `worker2` 上，再度触发联网拉取）。

### 1. 标准“两阶段”离线灌注命令

一旦遇到网络卡壳，应当立刻切断无谓的等待，启动“宿主机拦截 + 强灌全集群”连招：

Bash

```
# ==================== 阶段 A：拉取并灌入 Operator 机器人镜像 ====================
# 1. 宿主机暴力拉取
docker pull ghcr.io/cloudnative-pg/cloudnative-pg:1.29.1

# 2. 一键渗透分发给 Kind 集群下的所有节点
kind load docker-image ghcr.io/cloudnative-pg/cloudnative-pg:1.29.1 --name dev-cluster

# ==================== 阶段 B：拉取并灌入 PostgreSQL 内核镜像 ====================
# 1. 宿主机拉取真正运行的数据库内核
docker pull ghcr.io/cloudnative-pg/postgresql:18.3-system-trixie

# 2. 再次全量分发（彻底喂饱 worker 和 worker2 节点）
kind load docker-image ghcr.io/cloudnative-pg/postgresql:18.3-system-trixie --name dev-cluster
```

> **💡 运维进阶思考（横向走私流）** 若遇到某节点内部已有镜像而另一节点缺失，且宿主机无缓存时，可在宿主机利用 Linux 管道直接对两台打工人容器实施“隔空物理传送”： `docker save ghcr.io/.../postgresql:18.3-system-trixie -o /tmp/pg.tar` `docker exec -i dev-cluster-worker ctr -n k8s.io images import - < /tmp/pg.tar`

## 三、 集群启动、自愈演练与密室破案

### 1. 重新发车与状态监控

``` Bash
# 部署高可用战车
kubectl apply -f pg-cluster.yaml

# 盯紧进度，围观一主两从节点的诞生
kubectl get pods -w
```

### 2. 拔电源演练（Failover 自动容灾验证）

强行秒杀坐镇中央的 1 号主库，模拟物理机房突发断电断网：

```Bash
kubectl delete pod my-ha-postgres-1 --force --grace-period=0
```

**自愈收敛机制**：

- **毫秒级**：Operator 拦截读写流量，在剩余从库中发起仲裁，执行 `pg_ctl promote` 将 2 号或 3 号**秒级晋升为新主库**。
- **秒级**：依据声明式自治原则，大脑自动补齐副本数，重新投胎拉起新 1 号 Pod，并将其降格为“新兵从库”，拉通流复制（Streaming Replication）管道。

### 3. 双重密码密室破案

CNPG 默认在初始化时，会生成两套完全不同的安全凭证。若遇到 `password authentication failed`，通常是误拿了钥匙：

- **钥匙 A：普通应用用户 `app` 的密码（强烈建议业务、测试使用）**
``` shell
kubectl get secret my-ha-postgres-app -o jsonpath="{.data.password}" | base64 --decode; echo
```

- **钥匙 B：超级管理员 `postgres` 的密码**
``` shell
kubectl get secret my-ha-postgres-superuser -o jsonpath="{.data.password}" | base64 --decode; echo
```

## 四、 生产级专属流量路由与安全连入

CNPG 在底层自动铺设了极其优雅的 Service 矩阵，业务层严禁直接连接 Pod 的物理 IP，必须通过以下专属通道进行通信：

### 1. 路由网络通道清单

- `my-ha-postgres-rw`：**绝对的主库通道（Read-Write）**。无论底层发生何种主从切换，此域名永远精准指向唯一的 Primary 读写节点。
- `my-ha-postgres-ro`：**只读通道（Read-Only）**。流量在所有从库（Replica）之间自动负载均衡，跑大数据分析、报表查询时专走此路，绝不影响主库。

### 2. 终极无感连入方案（两步走常驻 Debug 模式）

为了规避交互式单行命令因控制台握手或网络超时引发的 `timed out waiting for the condition` 崩溃，建议采用正统的“两步走”秘密通道：
#### 步骤一：在后台拉起一个免联网、无限期站岗的侦察机 Pod

``` Bash
kubectl run pg-client-debug \
  --image=ghcr.io/cloudnative-pg/postgresql:18.3-system-trixie \
  --image-pull-policy=IfNotPresent \
  --overrides='{
    "spec": {
      "containers": [
        {
          "name": "pg-client-debug",
          "image": "ghcr.io/cloudnative-pg/postgresql:18.3-system-trixie",
          "command": ["sleep", "infinity"],
          "imagePullPolicy": "IfNotPresent"
        }
      ]
    }
  }'
```

#### 步骤二：肉身潜入容器，直击高可用主库服务

``` Bash
# 使用 app 用户连入 app 数据库
kubectl exec -it pg-client-debug -- psql -h my-ha-postgres-rw -U app -d app
```

_此时贴入专属密码，直接秒进。_

#### 步骤三：真身巡检 SQL

进库后可敲入以下命令，实时复核当前握手节点的健康状况与物理 IP：

``` SQL
SELECT 
   inet_server_addr() AS "当前物理节点IP",
   CASE WHEN pg_is_in_recovery() THEN '从库 (ReadOnly)' ELSE '主库 (ReadWrite)' END AS "当前节点角色",
   version() AS "内核版本";
```

#### 步骤四：阅后即焚清理

``` Bash
# 测试完毕后，一键拔掉 debug 侦察机，保持集群内功纯净
kubectl delete pod pg-client-debug
```

- 关联双链： [[Kubernetes]], [[PostgreSQL]], [[分布式系统]], [[架构设计]] 