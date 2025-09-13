# 业务数据流转拓扑图（生产环境）

> 依据《脚本整理和梳理.pdf》《ALLIN 升级 morph 基础架构部署方案.pdf》《ALLIN nginx-ingress 配置详解 allinx.io》中的路由、Ingress/LB、服务清单与中间件配置整理。仅覆盖生产环境与对外入口。

---

## 1. 自顶向下的流量路径（概览图）

```mermaid
%%{init: {"flowchart": {"htmlLabels": false, "useMaxWidth": true, "nodeSpacing": 6, "rankSpacing": 10}}}%%
flowchart TB

%% ============ Single vertical chain ============

u[Client]; dns[DNS]; cf[CloudFront]; alb_pub[ALB apisix-public]; alb_int[ALB apisix-internal]; apisix[APISIX];
u-->dns; dns-->cf; cf-->alb_pub; alb_pub-->apisix; dns-->alb_int; alb_int-->apisix;

%% NLB direct path (still on the same chain)
nlb_awf[NLB awf-spa-nlb 80 443 external]; cf-->nlb_awf; 
m_awf_spa[morph-awf-spa  svc8080]; nlb_awf-->m_awf_spa;

%% APISIX routing entries, chained one-by-one (host -> path -> service)
r_www_host[www.allinpro.com]; apisix-->r_www_host;
r_www_path[/*]; r_www_host-->r_www_path; r_www_path-->m_awf_spa;

r_api_root_host[api.allinpro.com]; m_awf_spa-->r_api_root_host;
r_api_root_path[/*]; r_api_root_host-->r_api_root_path;
l_spotapi[landry-spotapi-web  svc8080]; r_api_root_path-->l_spotapi;

r_api_fweb_host[api.allinpro.com]; l_spotapi-->r_api_fweb_host;
r_api_fweb_path[/futures/web/api/*]; r_api_fweb_host-->r_api_fweb_path;
m_fweb[morph-futuresweb-web  svc8080]; r_api_fweb_path-->m_fweb;

r_api_fopen_host[api.allinpro.com]; m_fweb-->r_api_fopen_host;
r_api_fopen_path[/futuresopen/*]; r_api_fopen_host-->r_api_fopen_path;
m_fopen[morph-futuresopen-web  svc8080]; r_api_fopen_path-->m_fopen;

r_api_fws_host[api.allinpro.com]; m_fopen-->r_api_fws_host;
r_api_fws_path[/futures/ws*]; r_api_fws_host-->r_api_fws_path;
m_fws[morph-futuresws-app  svc8080]; r_api_fws_path-->m_fws;

r_api_nexs_host[api.allinpro.com]; m_fws-->r_api_nexs_host;
r_api_nexs_path[/moth-nexs-gateway/*]; r_api_nexs_host-->r_api_nexs_path;
n_gw[moth-nexs-gateway  svc8080]; r_api_nexs_path-->n_gw;

r_user_host[user.allinpro.com]; n_gw-->r_user_host;
r_user_path[/*]; r_user_host-->r_user_path;
l_user[landry-userserver-web  svc8080]; r_user_path-->l_user;

r_ws_host[ws.allinpro.com]; l_user-->r_ws_host;
r_ws_path[/ws*]; r_ws_host-->r_ws_path;
l_spotws[landry-spotws-web  svc8080]; r_ws_path-->l_spotws;

r_broker_host[brokerserver.allinpro.com]; l_spotws-->r_broker_host;
r_broker_path[/*]; r_broker_host-->r_broker_path;
l_broker_svr[landry-brokerserver-web  NodePort 8080-30864]; r_broker_path-->l_broker_svr;

r_mack_api_host[mackerel.aie.prod]; l_broker_svr-->r_mack_api_host;
r_mack_api_path[/api/*]; r_mack_api_host-->r_mack_api_path;
l_anemone[landry-anemone-web  svc8080]; r_mack_api_path-->l_anemone;

r_mack_fe_host[mackerel.aie.prod or mackerel.allinpro.com]; l_anemone-->r_mack_fe_host;
r_mack_fe_path[/*]; r_mack_fe_host-->r_mack_fe_path;
l_mack_spa[landry-mackerel-spa  svc8080]; r_mack_fe_path-->l_mack_spa;

r_internal_host[*.aie.prod internal]; l_mack_spa-->r_internal_host;
r_internal_path[/*]; r_internal_host-->r_internal_path;
m_accesshttp[morph-narwhal-accesshttp  svc8080  agent8888]; r_internal_path-->m_accesshttp;

%% Landry catalog (still chaining vertically)
l_apa_spa[landry-apa-spa  svc8080]; m_accesshttp-->l_apa_spa;
l_awftest[landry-awftest-spa  svc8080]; l_apa_spa-->l_awftest;
l_spottask[landry-spottask-web  svc8080]; l_awftest-->l_spottask;
l_gateway[landry-gateway-web  svc8080]; l_spottask-->l_gateway;

l_ces_access[landry-ces-accesshttp  svc8080]; l_gateway-->l_ces_access;
l_ces_match[landry-ces-matchengine  svc7316]; l_ces_access-->l_ces_match;
l_ces_mktpr[landry-ces-marketprice  svc7416]; l_ces_match-->l_ces_mktpr;
l_ces_histw[landry-ces-historywriter  no-svc]; l_ces_mktpr-->l_ces_histw;
l_ces_histr[landry-ces-historyreader  svc7516]; l_ces_histw-->l_ces_histr;
l_ces_cache[landry-ces-cachecenter  svc7810 7811 7812 7813 7802 7803]; l_ces_histr-->l_ces_cache;
l_ces_mon[landry-ces-monitorcenter  svc5555]; l_ces_cache-->l_ces_mon;
l_ces_sum[landry-ces-tradesummary  svc7519]; l_ces_mon-->l_ces_sum;

l_clairvoy[landry-clairvoy-web  NodePort 8080-30906 9000-31816]; l_ces_sum-->l_clairvoy;
l_cobocb[landry-cobocb-web  svc8080]; l_clairvoy-->l_cobocb;
l_cobogw[landry-cobogw-web  svc8080]; l_cobocb-->l_cobogw;
l_trans[landry-trans-web  NodePort 8080-31603 9000-31562]; l_cobogw-->l_trans;
l_tg_bot[landry-tgsggd-bot  no-port]; l_trans-->l_tg_bot;

%% Morph catalog (continuing down the chain)
m_awftest[morph-awftest-spa  svc8080]; l_tg_bot-->m_awftest;
m_fadmin[morph-futuresadmin-web  svc8080]; m_awftest-->m_fadmin;
m_fsched[morph-futuresschedule-app  svc8080]; m_fadmin-->m_fsched;
m_fmkt[morph-futuresmarket-app  svc8080]; m_fsched-->m_fmkt;
m_sub[morph-sub-web  svc8080]; m_fmkt-->m_sub;
m_cond[morph-cond-web  svc8080]; m_sub-->m_cond;

m_mon[morph-narwhal-monitorcenter  svc5555  agent8888]; m_cond-->m_mon;
m_alert[morph-narwhal-alertcenter  svc4444  agent8888]; m_mon-->m_alert;
m_oplog[morph-narwhal-operlogcompact  no-svc]; m_alert-->m_oplog;

m_mktidx[morph-narwhal-marketindex  svc7901  agent8888]; m_oplog-->m_mktidx;
m_mktpr[morph-narwhal-marketprice  svc7416  agent8888]; m_mktidx-->m_mktpr;
m_match[morph-narwhal-matchengine  svc7316 8316  agent8888]; m_mktpr-->m_match;
m_cache[morph-narwhal-cachecenter  svc7810 7802 7803  agent8888]; m_match-->m_cache;
m_histw[morph-narwhal-historywriter  no-svc  agent8888]; m_cache-->m_histw;
m_histr[morph-narwhal-historyreader  svc7516  agent8888]; m_histw-->m_histr;
m_sum[morph-narwhal-tradesummary  svc7519  agent8888]; m_histr-->m_sum;

%% Moth NEXS catalog
n_mkt[moth-nexs-market  svc9090]; m_sum-->n_mkt;
n_trd[moth-nexs-trade  svc9090]; n_mkt-->n_trd;
n_uc[moth-nexs-usercenter  svc9090]; n_trd-->n_uc;

%% Data layer at bottom
spotdb[spotdb.aie.prod  RDS]; n_uc-->spotdb;
spotredis[spotredis.aie.prod  Redis];
kafka[kafka.aie.prod 9092  Kafka];
futdb[futuresnewdb.aie.prod  RDS];
futredis[futuresnewredis.aie.prod 6379  Redis];
kafkanew[kafkanew.aie.prod 9092  Kafka];
etcdnew[etcdnew.aie.prod 2379  Etcd];

%% Dependencies (downwards only)
l_spotapi-->spotdb; l_spotapi-->spotredis; l_spotapi-->kafka; l_spotws-->spotredis; 
m_fweb-.->futdb; m_fweb-.->futredis; m_fws-.->futredis; m_fopen-.->futdb; m_fadmin-.->futdb; m_fsched-.->futdb;
m_accesshttp-.->etcdnew; m_accesshttp-.->kafkanew; 
n_gw-.->futdb; n_mkt-.->futdb; n_trd-.->futdb; n_uc-.->futdb;

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

### 2.1 入口补充（NLB：awf-spa-nlb）

> 在 morph 命名空间发现类型为 **LoadBalancer** 的 `awf-spa-nlb`，外部入口：`aie1p-nlb-awfspa-*.elb.ap-southeast-1.amazonaws.com`，对外端口 80/443，对内指向 `app=spa`（即 `morph-awf-spa:8080`）。这意味着 **CloudFront 的某些 Origin 可能直接指向该 NLB**（而不是经 APISIX）。已在下图增加备选路径，建议后续核对 CloudFront 分发的 Origin 配置以最终确认。

```mermaid
flowchart LR
  U[用户] --> CF[CloudFront]
  subgraph PathA[路径A]
    CF --> ALB[ALB apisix-public] --> APISIX[APISIX] --> AWF[morph-awf-spa:8080]
  end
  subgraph PathB[路径B]
    CF --> NLB[awf-spa-nlb 80/443] --> AWF
  end
```

\--- | --- | --- |
\| [www.allinpro.com](http://www.allinpro.com) | /\* | morph/morph-awf-spa:8080 |
\| api.allinpro.com | /\* | landry/landry-spotapi-web:8080 |
\| api.allinpro.com | /futures/web/api/\* | morph/morph-futuresweb-web:8080 |
\| api.allinpro.com | /futuresopen/\* | morph/morph-futuresopen-web:8080 |
\| api.allinpro.com | /futures/ws\* | morph/morph-futuresws-app:8080 |
\| api.allinpro.com | /moth-nexs-gateway/\* | moth/moth-nexs-gateway:8080 |
\| user.allinpro.com | /\* | landry/landry-userserver-web:8080 |
\| ws.allinpro.com | /ws\* | landry/landry-spotws-web:8080 |
\| brokerserver.allinpro.com | /\* | landry/landry-brokerserver-web:8080 |
\| mackerel.aie.prod | /api/\* | landry/landry-anemone-web:8080 |
\| mackerel.aie.prod / mackerel.allinpro.com | /\* | landry/landry-mackerel-spa:8080 |
\| *.aie.prod（内部） | /* | morph/morph-narwhal-accesshttp:8080 等 |

> 注：上表只列出与用户路径相关的核心条目，完整清单见文档原始路由导出。

---

## 3. 服务清单（生产）

> 以你提供的 `kubectl get pod` 为基线，补齐并按命名空间分组；角色为基于命名与既有文档的合理归类，待后续用查询指令核验端口、依赖与路由。

### 3.1 Landry（现货域）

| Pod 名称                                    | 角色/用途（推断）       | 备注                                       |
| ----------------------------------------- | --------------- | ---------------------------------------- |
| landry-anemone-web                        | Mackerel 后端 API | svc: 8080/TCP                            |
| landry-apa-spa                            | 前端 SPA          | svc: 8080                                |
| landry-awftest-spa                        | 前端 SPA（测试）      | svc: 8080                                |
| landry-brokercron-broker-daily-report     | 定时任务            | 无对外端口                                    |
| landry-brokercron-rebate                  | 定时任务            | 无对外端口                                    |
| landry-brokercron-rebate-daily-settlement | 定时任务            | 无对外端口                                    |
| landry-brokercron-snap-user-balance       | 定时任务            | 无对外端口                                    |
| landry-brokercron-stat-user-trade         | 定时任务            | 无对外端口                                    |
| landry-brokercron-sync-match-fee          | 定时任务            | 无对外端口                                    |
| landry-brokercron-sync-position           | 定时任务            | 无对外端口                                    |
| landry-brokercron-sync-user-match         | 定时任务            | 无对外端口                                    |
| landry-brokercron-sync-user-relationship  | 定时任务            | 无对外端口                                    |
| landry-brokerserver-web                   | 券商服务后端          | **svc: NodePort 8080→30864**             |
| landry-ces-accesshttp                     | CES 接入/HTTP     | svc: 8080                                |
| landry-ces-cachecenter                    | CES 缓存中心        | svc: 7810/7811/7812/7813/7802/7803       |
| landry-ces-historyreader                  | CES 历史读取        | svc: 7516                                |
| landry-ces-historywriter                  | CES 历史写入        | （无独立 svc，依业务链路内用）                        |
| landry-ces-marketprice                    | CES 行情价格        | svc: 7416                                |
| landry-ces-matchengine                    | CES 撮合引擎        | svc: 7316                                |
| landry-ces-monitorcenter                  | CES 监控中心        | svc: 5555                                |
| landry-ces-tradesummary                   | CES 成交汇总        | svc: 7519                                |
| landry-clairvoy-web                       | 工具/控制台          | **svc: NodePort 8080→30906, 9000→31816** |
| landry-cobocb-web                         | 后端/工具           | svc: 8080                                |
| landry-cobogw-web                         | 网关/工具           | svc: 8080                                |
| landry-gateway-web                        | 业务网关            | svc: 8080                                |
| landry-mackerel-spa                       | Mackerel 前端     | svc: 8080                                |
| landry-spotapi-web                        | 现货 API          | svc: 8080                                |
| landry-spottask-web                       | 现货任务中心          | svc: 8080                                |
| landry-spotws-web                         | 现货 WebSocket    | svc: 8080                                |
| landry-tgsggd-bot                         | 机器人/告警          | 无对外端口（env: a,b,c...）                     |
| landry-trans-web                          | 交易/转账或工具        | **svc: NodePort 8080→31603, 9000→31562** |
| landry-userserver-web                     | 用户服务            | svc: 8080                                |

### 3.2 Morph（合约域）

| Pod 名称                       | 角色/用途（推断）    | 备注                                       |
| ---------------------------- | ------------ | ---------------------------------------- |
| ingress-nginx-controller     | Ingress 控制器  | svc: 80/443, admission 443               |
| morph-awf-spa                | 前端 SPA       | svc: 8080；另有 **NLB: awf-spa-nlb 80/443** |
| morph-awftest-spa            | 前端 SPA（测试）   | svc: 8080                                |
| morph-cond-web               | 条件单/策略等      | svc: 8080                                |
| morph-futuresadmin-web       | 合约后台         | svc: 8080                                |
| morph-futuresmarket-app      | 合约行情         | svc: 8080                                |
| morph-futuresopen-web        | 合约开放接口       | svc: 8080                                |
| morph-futuresschedule-app    | 合约调度         | svc: 8080                                |
| morph-futuresweb-web         | 合约 Web/API   | svc: 8080                                |
| morph-futuresws-app          | 合约 WebSocket | svc: 8080                                |
| morph-narwhal-accesshttp     | 撮合域接入        | svc: 8080（副容器 monitoragent:8888）         |
| morph-narwhal-alertcenter    | 告警中心         | svc: 4444（+8888）                         |
| morph-narwhal-cachecenter    | 缓存中心         | svc: 7810/7802/7803（+8888）               |
| morph-narwhal-historyreader  | 历史读取         | svc: 7516（+8888）                         |
| morph-narwhal-historywriter  | 历史写入         | （无 svc 端口；容器+8888）                       |
| morph-narwhal-marketindex    | 指数           | svc: 7901（+8888）                         |
| morph-narwhal-marketprice    | 行情价格         | svc: 7416（+8888）                         |
| morph-narwhal-matchengine    | 撮合引擎         | svc: 7316/8316（+8888）                    |
| morph-narwhal-monitorcenter  | 监控中心         | svc: 5555（+8888）                         |
| morph-narwhal-operlogcompact | 日志压缩         | （无 svc、端口未暴露）                            |
| morph-narwhal-tradesummary   | 成交汇总         | svc: 7519（+8888）                         |
| morph-sub-web                | 订阅/子账号等      | svc: 8080                                |

### 3.3 Moth（NEXS）

| Pod 名称               | 角色/用途（推断） | 备注        |
| -------------------- | --------- | --------- |
| moth-nexs-gateway    | 网关        | svc: 8080 |
| moth-nexs-market     | 行情        | svc: 9090 |
| moth-nexs-trade      | 交易        | svc: 9090 |
| moth-nexs-usercenter | 用户中心      | svc: 9090 |

### 3.4 现货 CES 子图（细粒度）

```mermaid
flowchart LR
  APISIX --> SPOTAPI
  APISIX --> SPOTWS
  SPOTAPI --> CES_ACCESS[ces-accesshttp]
  CES_ACCESS --> CES_MATCH[ces-matchengine]
  CES_ACCESS --> CES_MKT[ces-marketprice]
  CES_ACCESS --> CES_HISTW[ces-historywriter]
  CES_HISTW --> CES_HISTR[ces-historyreader]
  CES_ACCESS --> CES_CACHE[ces-cachecenter]
  CES_ACCESS --> CES_MON[ces-monitorcenter]
  CES_ACCESS --> CES_SUM[ces-tradesummary]
  SPOTAPI --> SPOTDB[(spotdb)]
  SPOTAPI --> SPOTREDIS[(spotredis)]
  SPOTAPI --> KAFKA[(kafka)]
  SPOTWS --> SPOTREDIS
```

### 3.5 Service 暴露方式与端口（来自 `kubectl get svc -o wide`）

**Landry**

| Service                  | Type         | ClusterIP      | Ports (→NodePort)             |
| ------------------------ | ------------ | -------------- | ----------------------------- |
| landry-anemone-web       | ClusterIP    | 172.20.21.136  | 8080                          |
| landry-apa-spa           | ClusterIP    | 172.20.2.42    | 8080                          |
| landry-awftest-spa       | ClusterIP    | 172.20.247.6   | 8080                          |
| landry-brokerserver-web  | **NodePort** | 172.20.169.156 | 8080→30864                    |
| landry-ces-accesshttp    | ClusterIP    | 172.20.118.119 | 8080                          |
| landry-ces-cachecenter   | ClusterIP    | 172.20.26.118  | 7810,7811,7812,7813,7802,7803 |
| landry-ces-historyreader | ClusterIP    | 172.20.131.200 | 7516                          |
| landry-ces-marketprice   | ClusterIP    | 172.20.250.131 | 7416                          |
| landry-ces-matchengine   | ClusterIP    | 172.20.123.252 | 7316                          |
| landry-ces-monitorcenter | ClusterIP    | 172.20.240.167 | 5555                          |
| landry-ces-tradesummary  | ClusterIP    | 172.20.12.52   | 7519                          |
| landry-clairvoy-web      | **NodePort** | 172.20.130.89  | 8080→30906, 9000→31816        |
| landry-cobocb-web        | ClusterIP    | 172.20.130.39  | 8080                          |
| landry-cobogw-web        | ClusterIP    | 172.20.242.96  | 8080                          |
| landry-gateway-web       | ClusterIP    | 172.20.150.146 | 8080                          |
| landry-mackerel-spa      | ClusterIP    | 172.20.139.47  | 8080                          |
| landry-spotapi-web       | ClusterIP    | 172.20.159.111 | 8080                          |
| landry-spottask-web      | ClusterIP    | 172.20.168.40  | 8080                          |
| landry-spotws-web        | ClusterIP    | 172.20.7.15    | 8080                          |
| landry-trans-web         | **NodePort** | 172.20.49.204  | 8080→31603, 9000→31562        |
| landry-userserver-web    | ClusterIP    | 172.20.67.107  | 8080                          |

**Morph**

| Service                            | Type                  | ClusterIP      | External                                             | Ports          |
| ---------------------------------- | --------------------- | -------------- | ---------------------------------------------------- | -------------- |
| awf-spa-nlb                        | **LoadBalancer(NLB)** | 172.20.222.54  | aie1p-nlb-awfspa-\*.elb.ap-southeast-1.amazonaws.com | 80,443         |
| ingress-nginx-controller           | ClusterIP             | 172.20.114.16  | —                                                    | 80,443         |
| ingress-nginx-controller-admission | ClusterIP             | 172.20.23.159  | —                                                    | 443            |
| morph-awf-spa                      | ClusterIP             | 172.20.3.34    | —                                                    | 8080           |
| morph-awftest-spa                  | ClusterIP             | 172.20.86.95   | —                                                    | 8080           |
| morph-cond-web                     | ClusterIP             | 172.20.227.125 | —                                                    | 8080           |
| morph-futuresadmin-web             | ClusterIP             | 172.20.5.108   | —                                                    | 8080           |
| morph-futuresmarket-app            | ClusterIP             | 172.20.220.215 | —                                                    | 8080           |
| morph-futuresopen-web              | ClusterIP             | 172.20.129.63  | —                                                    | 8080           |
| morph-futuresschedule-app          | ClusterIP             | 172.20.217.62  | —                                                    | 8080           |
| morph-futuresweb-web               | ClusterIP             | 172.20.105.94  | —                                                    | 8080           |
| morph-futuresws-app                | ClusterIP             | 172.20.249.255 | —                                                    | 8080           |
| morph-narwhal-accesshttp           | ClusterIP             | 172.20.88.111  | —                                                    | 8080           |
| morph-narwhal-alertcenter          | ClusterIP             | 172.20.178.192 | —                                                    | 4444           |
| morph-narwhal-cachecenter          | ClusterIP             | 172.20.93.111  | —                                                    | 7810,7802,7803 |
| morph-narwhal-historyreader        | ClusterIP             | 172.20.151.178 | —                                                    | 7516           |
| morph-narwhal-marketindex          | ClusterIP             | 172.20.76.67   | —                                                    | 7901           |
| morph-narwhal-marketprice          | ClusterIP             | 172.20.59.159  | —                                                    | 7416           |
| morph-narwhal-matchengine          | ClusterIP             | 172.20.21.58   | —                                                    | 7316,8316      |
| morph-narwhal-monitorcenter        | ClusterIP             | 172.20.204.222 | —                                                    | 5555           |
| morph-narwhal-tradesummary         | ClusterIP             | 172.20.36.7    | —                                                    | 7519           |
| morph-sub-web                      | ClusterIP             | 172.20.77.134  | —                                                    | 8080           |

**Moth**

| Service              | Type      | ClusterIP     | Ports |
| -------------------- | --------- | ------------- | ----- |
| moth-nexs-gateway    | ClusterIP | 172.20.21.60  | 8080  |
| moth-nexs-market     | ClusterIP | 172.20.70.196 | 9090  |
| moth-nexs-trade      | ClusterIP | 172.20.17.52  | 9090  |
| moth-nexs-usercenter | ClusterIP | 172.20.32.1   | 9090  |

---

## 4. 中间件与数据源（生产）

中间件与数据源（生产）
中间件与数据源（生产）

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
