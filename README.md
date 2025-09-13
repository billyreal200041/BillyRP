# 业务数据流转拓扑图（生产环境）

> 依据《脚本整理和梳理.pdf》《ALLIN 升级 morph 基础架构部署方案.pdf》《ALLIN nginx-ingress 配置详解 allinx.io》中的路由、Ingress/LB、服务清单与中间件配置整理。仅覆盖生产环境与对外入口。

---

## 1. 自顶向下的流量路径（概览图）

```mermaid
flowchart LR
  %% 顶层
  subgraph Internet[Internet]
    U[终端用户/客户端]
  end
  U -->|FQDN| DNS[DNS: Route53/外部DNS]

  subgraph Edge[CDN/边缘]
    CF[CloudFront<br/>TLS 终止]
  end
  DNS --> CF

  subgraph Entry[入口负载均衡]
    ALB_PUB[ALB aie1p-apisix-public<br/>Hosts: www.allinpro.com, *.allinpro.com, *.allinx.io]
    ALB_INT[ALB aie1p-apisix-internal<br/>Hosts: *.aie.prod]
  end

  CF --> ALB_PUB
  DNS --> ALB_INT

  APISIX[APISIX Gateway<br/>(EKS)]

  ALB_PUB --> APISIX
  ALB_INT --> APISIX

  %% --- 业务域 ---
  subgraph Spot[现货域 (landry)]
    SPOTAPI[landry-spotapi-web]
    SPOTWS[landry-spotws-web]
    USERSVR[landry-userserver-web]
    BROKER[landry-brokerserver-web]
    MACK_SPA[landry-mackerel-spa]
    ANEMONE[landry-anemone-web]
    GATEWAY[landry-gateway-web]
  end

  subgraph Futures[合约域 (morph)]
    AWF_SPA[morph-awf-spa]
    FUT_WEB[morph-futuresweb-web]
    FUT_OPEN[morph-futuresopen-web]
    FUT_WS[morph-futuresws-app]
    FUT_ADMIN[morph-futuresadmin-web]
    FUT_MARKET[morph-futuresmarket-app]
    NARW_ACC[morph-narwhal-accesshttp]
  end

  subgraph Nexs[NEXS (moth)]
    NEXS_GW[moth-nexs-gateway]
    NEXS_MKT[moth-nexs-market]
    NEXS_TRD[moth-nexs-trade]
    NEXS_UC[moth-nexs-usercenter]
  end

  %% --- 中间件 ---
  subgraph Middleware[中间件/数据层]
    subgraph LandryMW[现货中间件]
      SPOTDB[(spotdb.aie.prod<br/>RDS)]
      SPOTREDIS[(spotredis.aie.prod<br/>Redis)]
      KAFKA[(kafka.aie.prod:9092<br/>Kafka)]
    end
    subgraph MorphMW[合约中间件]
      FUTDB[(futuresnewdb.aie.prod<br/>RDS)]
      FUTREDIS[(futuresnewredis.aie.prod<br/>Redis)]
      KAFKANEW[(kafkanew.aie.prod:9092<br/>Kafka)]
      ETCDNEW[(etcdnew.aie.prod:2379<br/>Etcd)]
    end
  end

  %% --- APISIX 路由示意（标签文字仅用于说明） ---
  APISIX -->|www.allinpro.com /*| AWF_SPA
  APISIX -->|api.allinpro.com /*| SPOTAPI
  APISIX -->|api.allinpro.com /futures/web/api/*| FUT_WEB
  APISIX -->|api.allinpro.com /futuresopen/*| FUT_OPEN
  APISIX -->|api.allinpro.com /futures/ws*| FUT_WS
  APISIX -->|api.allinpro.com /moth-nexs-gateway/*| NEXS_GW
  APISIX -->|user.allinpro.com /*| USERSVR
  APISIX -->|ws.allinpro.com /ws*| SPOTWS
  APISIX -->|brokerserver.allinpro.com /*| BROKER
  APISIX -->|mackerel.aie.prod /api/*| ANEMONE
  APISIX -->|mackerel.allinpro.com 或 mackerel.aie.prod /*| MACK_SPA
  APISIX -->|*.aie.prod 内部域| NARW_ACC

  %% --- 业务到数据依赖 ---
  SPOTAPI --> SPOTDB
  SPOTAPI --> SPOTREDIS
  SPOTAPI --> KAFKA
  SPOTWS --> SPOTREDIS
  USERSVR --> SPOTDB

  AWF_SPA --> FUT_WEB
  FUT_WEB --> FUTDB
  FUT_WEB --> FUTREDIS
  FUT_OPEN --> FUTDB
  FUT_WS --> FUTREDIS
  FUT_ADMIN --> FUTDB
  FUT_ADMIN --> FUTREDIS
  NARW_ACC --> ETCDNEW
  NARW_ACC --> KAFKANEW

  NEXS_GW --> FUTDB
  NEXS_MKT --> FUTDB
  NEXS_TRD --> FUTDB
  NEXS_UC --> FUTDB
```

---

## 2. 生产对外域名 → 上游服务映射（节选）

| 域名 (Host)                                   | 路径 (Path)             | 上游服务 (Namespace/Service)              |
| ------------------------------------------- | --------------------- | ------------------------------------- |
| [www.allinpro.com](http://www.allinpro.com) | /\*                   | morph/morph-awf-spa:8080              |
| api.allinpro.com                            | /\*                   | landry/landry-spotapi-web:8080        |
| api.allinpro.com                            | /futures/web/api/\*   | morph/morph-futuresweb-web:8080       |
| api.allinpro.com                            | /futuresopen/\*       | morph/morph-futuresopen-web:8080      |
| api.allinpro.com                            | /futures/ws\*         | morph/morph-futuresws-app:8080        |
| api.allinpro.com                            | /moth-nexs-gateway/\* | moth/moth-nexs-gateway:8080           |
| user.allinpro.com                           | /\*                   | landry/landry-userserver-web:8080     |
| ws.allinpro.com                             | /ws\*                 | landry/landry-spotws-web:8080         |
| brokerserver.allinpro.com                   | /\*                   | landry/landry-brokerserver-web:8080   |
| mackerel.aie.prod                           | /api/\*               | landry/landry-anemone-web:8080        |
| mackerel.aie.prod / mackerel.allinpro.com   | /\*                   | landry/landry-mackerel-spa:8080       |
| \*.aie.prod（内部）                             | /\*                   | morph/morph-narwhal-accesshttp:8080 等 |

> 注：上表只列出与用户路径相关的核心条目，完整清单见文档原始路由导出。

---

## 3. 服务清单（生产）

### 3.1 Landry（现货域）

* landry-spotapi-web、landry-spotws-web、landry-userserver-web、landry-brokerserver-web
* landry-gateway-web、landry-mackerel-spa、landry-anemone-web
* CES/撮合与工具：landry-ces-*, landry-trans-web, landry-cobo* 等

### 3.2 Morph（合约域）

* Web/API：morph-awf-spa、morph-futuresweb-web、morph-futuresopen-web、morph-futuresws-app、morph-futuresadmin-web、morph-futuresmarket-app
* Narwhal（撮合/组件）：morph-narwhal-accesshttp 等
* Ingress：ingress-nginx-controller（ClusterIP，ALB 直连 POD IP）

### 3.3 Moth（NEXS）

* moth-nexs-gateway、moth-nexs-market、moth-nexs-trade、moth-nexs-usercenter

---

## 4. 中间件与数据源（生产）

* 现货（Landry）

  * RDS：spotdb.aie.prod
  * Redis：spotredis.aie.prod
  * Kafka：kafka.aie.prod:9092

* 合约（Morph）

  * RDS：futuresnewdb.aie.prod
  * Redis：futuresnewredis.aie.prod
  * Kafka：kafkanew\.aie.prod:9092（EC2 自建）
  * Etcd：etcdnew\.aie.prod

---

## 5. 入口与负载均衡

* 外部域名（allinpro.com / allinx.io）→ CloudFront（HTTPS 终止）→ ALB（HTTP）→ APISIX（HTTP）→ EKS 服务（ClusterIP）
* 内部域名（\*.aie.prod）→ 内部 ALB → APISIX → EKS 服务
* NGINX Ingress（morph）为 ClusterIP，ALB 使用 target-type: ip 直连 POD

---

## 6. 备注

* 本图用于“业务数据流转”视角，平台运维入口（apisix-admin、argocd 等）未展开。
* 若需导出 PNG/SVG 或按子域拆分子图，可在本页继续迭代。
