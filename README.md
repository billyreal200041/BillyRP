# 生产环境 EC2 服务与上下游关系清单（AI Ops List.xlsx）

以下为 **逐台 EC2** 的“实例名称 / 服务角色 / 主要上游 / 主要下游”清单（仅生产环境；已排除测试/临时用途条目）。

| 实例名称 | 服务角色 | 主要上游 | 主要下游 |
| --- | --- | --- | --- |
| aie1p-maker-02-copy | 做市服务 | Nacos；业务API（经 APISIX/NGINX） | Aurora makerdb；Kafka（可选） |
| aie1p-maker-02 | 做市服务 | Nacos；业务API（经 APISIX/NGINX） | Aurora makerdb；Kafka（可选） |
| aie1p-maker-03-copy | 做市服务 | Nacos；业务API（经 APISIX/NGINX） | Aurora makerdb；Kafka（可选） |
| aie1p-maker-03 | 做市服务 | Nacos；业务API（经 APISIX/NGINX） | Aurora makerdb；Kafka（可选） |
| aie1p-maker-nacos-01 | 做市 Nacos 配置中心 | 做市服务器（maker-*） | 配置数据（本机/磁盘） |
| aie1p-morph-narwhal-r52xlarge-8c64g-v132-a-Node | 期货撮合（Narwhal）节点池（EKS） | Narwhal Access | Kafka；Aurora futuresdb；Redis |
| aie1p-morph-narwhal-r52xlarge-8c64g-v132-a-Node | 期货撮合（Narwhal）节点池（EKS） | Narwhal Access | Kafka；Aurora futuresdb；Redis |
| aie1p-morph-narwhal-access-c5large-2c4g-v132-a-Node | 期货接入层（Narwhal Access）节点池（EKS） | APISIX/NGINX/客户端 | Narwhal 核心；Kafka；futuresdb/Redis |
| aie1p-morph-futuresweb-c5large-2c4g-v132-a-Node | 期货 Web 节点池（EKS） | APISIX/NGINX | Aurora futuresdb/futuresnewdb；Redis |
| aie1p-apisix-c5large-2c4g-v132-a-Node | APISIX 网关节点池（EKS） | ALB apisix-public/internal；客户端 | EKS 后端服务；ETCD |
| aie1p-apisix-c5large-2c4g-v132-a-Node | APISIX 网关节点池（EKS） | ALB apisix-public/internal；客户端 | EKS 后端服务；ETCD |
| aie1p-spotapi-c5large-2c4g-v132-a-Node | 现货 Spot API 节点池（EKS） | APISIX/NGINX | Aurora spotdb；Valkey spot-redis；Kafka（按需） |
| aie1p-devops-c5large-2c4g-v132-a-Node | DevOps/监控节点池（EKS） | 运维人员/系统 | K8S API；监控/日志系统 |
| aie1p-brokercron-c5large-2c4g-v132-a-Node | 业务定时作业节点池（EKS） | 调度/内部触发 | 各业务库；Redis；Kafka |
| aie1p-ces-access-c5large-2c4g-v132-a-Node | CES 接入层节点池（EKS） | APISIX/NGINX/客户端 | CES 核心；Kafka；spotdb/spot-redis |
| aie1p-spotws-c5xlarge-4c8g-v132-a-Node | 现货 Spot WebSocket 节点池（EKS） | APISIX/NGINX | Aurora spotdb；Valkey spot-redis |
| aie1p-ces-c52xlarge-8c16g-v132-a-Node | CES 撮合/核心节点池（EKS） | CES 接入层 | Kafka；spotdb；spot-redis |
| aie1p-jenkins | CI 构建服务器 | 开发者；GitLab Webhook | 制品库（ECR/S3）；ArgoCD/Helm |
| aie1p-gitlab | 代码仓库服务器 | 开发者 | Jenkins/Runner |
| aie1p-etcd-01 | ETCD（网关/服务发现配置） | APISIX/内部系统 | —— |
| aie1p-kafka-server-01 | Kafka 消息队列 | Narwhal/CES/业务服务（生产者） | 历史/统计/推送等（消费者） |
| aie1p-elasticsearch-01 | 日志检索（Elasticsearch） | 应用/网关日志 | 观测/分析 |
| aie1p-morph-kafkanew-0425 | Kafka 消息队列 | Narwhal/CES/业务服务（生产者） | 历史/统计/推送等（消费者） |
| aie1p-etcdnew-morph-0430 | ETCD（网关/服务发现配置） | APISIX/内部系统 | —— |
| aie1p-devops-c5large-2c4g-v132-a-Node | DevOps/监控节点池（EKS） | 运维人员/系统 | K8S API；监控/日志系统 |
| aie1p-morph-narwhal-access-c5large-2c4g-v132-a-Node | 期货接入层（Narwhal Access）节点池（EKS） | APISIX/NGINX/客户端 | Narwhal 核心；Kafka；futuresdb/Redis |
| aie1p-superset-server-1 | BI 报表服务器（Superset） | 分析人员 | Aurora 只读实例（spotdb 等） |
| aie1p-morph-narwhal-access-c5large-2c4g-v132-a-Node | 期货接入层（Narwhal Access）节点池（EKS） | APISIX/NGINX/客户端 | Narwhal 核心；Kafka；futuresdb/Redis |

## 拓扑（Mermaid）

> 说明：图中灰色节点为上下游**类别/托管资源**（非EC2），白色节点为本清单中的 EC2 实例。

```mermaid
graph LR
    ALB_APISIX["ALB apisix-public/internal"]:::cat
    ALB_NGINX["ALB nginx-public"]:::cat
    DEV["开发者/运维人员"]:::cat
    ANALYST["分析人员"]:::cat
    ARGO["ArgoCD/Helm（部署）"]:::cat
    AURORA_SPOT["Aurora spotdb"]:::cat
    AURORA_FUT["Aurora futuresdb/futuresnewdb"]:::cat
    AURORA_MAKER["Aurora makerdb"]:::cat
    VALKEY_SPOT["Valkey spot-redis"]:::cat
    REDIS_FUT["Redis（futures域）"]:::cat
    BACKENDS["EKS 后端服务（landry/morph）"]:::cat
    OBS["监控/日志系统"]:::cat
    CONSUMERS["下游消费者（历史/统计/推送等）"]:::cat
    n1["aie1p-maker-02-copy\n做市服务"]
    n2["aie1p-maker-02\n做市服务"]
    n3["aie1p-maker-03-copy\n做市服务"]
    n4["aie1p-maker-03\n做市服务"]
    n5["aie1p-maker-nacos-01\n做市 Nacos 配置中心"]
    n7["aie1p-morph-narwhal-r52xlarge-8c64g-v132-a-Node\n期货撮合（Narwhal）节点池（EKS）"]
    n28["aie1p-morph-narwhal-access-c5large-2c4g-v132-a-Node\n期货接入层（Narwhal Access）节点池（EKS）"]
    n9["aie1p-morph-futuresweb-c5large-2c4g-v132-a-Node\n期货 Web 节点池（EKS）"]
    n11["aie1p-apisix-c5large-2c4g-v132-a-Node\nAPISIX 网关节点池（EKS）"]
    n12["aie1p-spotapi-c5large-2c4g-v132-a-Node\n现货 Spot API 节点池（EKS）"]
    n25["aie1p-devops-c5large-2c4g-v132-a-Node\nDevOps/监控节点池（EKS）"]
    n14["aie1p-brokercron-c5large-2c4g-v132-a-Node\n业务定时作业节点池（EKS）"]
    n15["aie1p-ces-access-c5large-2c4g-v132-a-Node\nCES 接入层节点池（EKS）"]
    n16["aie1p-spotws-c5xlarge-4c8g-v132-a-Node\n现货 Spot WebSocket 节点池（EKS）"]
    n17["aie1p-ces-c52xlarge-8c16g-v132-a-Node\nCES 撮合/核心节点池（EKS）"]
    n18["aie1p-jenkins\nCI 构建服务器"]
    n19["aie1p-gitlab\n代码仓库服务器"]
    n20["aie1p-etcd-01\nETCD（网关/服务发现配置）"]
    n21["aie1p-kafka-server-01\nKafka 消息队列"]
    n22["aie1p-elasticsearch-01\n日志检索（Elasticsearch）"]
    n23["aie1p-morph-kafkanew-0425\nKafka 消息队列"]
    n24["aie1p-etcdnew-morph-0430\nETCD（网关/服务发现配置）"]
    n27["aie1p-superset-server-1\nBI 报表服务器（Superset）"]
    ALB_APISIX --> n1
    ALB_NGINX --> n1
    n5 --> n1
    n1 --> AURORA_MAKER
    n1 --> n21
    ALB_APISIX --> n2
    ALB_NGINX --> n2
    n5 --> n2
    n2 --> AURORA_MAKER
    n2 --> n21
    ALB_APISIX --> n3
    ALB_NGINX --> n3
    n5 --> n3
    n3 --> AURORA_MAKER
    n3 --> n21
    ALB_APISIX --> n4
    ALB_NGINX --> n4
    n5 --> n4
    n4 --> AURORA_MAKER
    n4 --> n21
    n7 --> AURORA_FUT
    n7 --> REDIS_FUT
    n7 --> n23
    ALB_APISIX --> n28
    ALB_NGINX --> n28
    n28 --> REDIS_FUT
    n28 --> n23
    ALB_APISIX --> n9
    ALB_NGINX --> n9
    n9 --> AURORA_FUT
    n9 --> REDIS_FUT
    n11 --> n20
    n11 --> n24
    ALB_APISIX --> n12
    ALB_NGINX --> n12
    n12 --> AURORA_SPOT
    n12 --> VALKEY_SPOT
    n12 --> n21
    DEV --> n25
    n25 --> OBS
    n14 --> REDIS_FUT
    n14 --> n21
    ALB_APISIX --> n15
    ALB_NGINX --> n15
    n15 --> n21
    ALB_APISIX --> n16
    ALB_NGINX --> n16
    n16 --> AURORA_SPOT
    n16 --> VALKEY_SPOT
    n17 --> n21
    DEV --> n18
    n19 --> n18
    n18 --> ARGO
    DEV --> n19
    n19 --> n18
    ALB_APISIX --> n20
    n22 --> OBS
    ALB_APISIX --> n24
    ANALYST --> n27
    DEV --> n19
    DEV --> n18
    ALB_APISIX --> BACKENDS
    ALB_NGINX --> BACKENDS
classDef cat fill:#eee,stroke:#999,stroke-width:1px;
```
