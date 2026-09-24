# ack-ruoyi-k8s

在阿里云 **ACK（容器服务 Kubernetes 版，托管集群）** 上部署 **RuoYi-Cloud 3.6.5** 微服务体系的完整学习配置集：业务微服务、中间件、前端接入、监控告警与链路追踪。

> 本仓库是个人学习「阿里云 ACK 上部署 K8s 微服务」过程中沉淀的 Kubernetes 清单与配置文件，环境为按量付费的个人学习环境（单副本、单可用区），非生产配置。

## 内容概览

| 类别 | 内容 |
| --- | --- |
| 业务微服务 | RuoYi-Cloud 6 个服务：`gateway / auth / system / gen / job / file` 的 Deployment + Service |
| 中间件 | Nacos 2.2.3（注册/配置中心）、MySQL 8.0、Redis 7.2 |
| 前端 | Nginx 1.29，`/prod-api` 反向代理到网关 |
| 监控告警 | kube-prometheus 0.18.0 定制版 + JMX exporter（JVM 指标）+ ServiceMonitor + PrometheusRule + Alertmanager → 钉钉机器人 |
| 链路追踪 | SkyWalking 10.4.0 + Elasticsearch 8.14.3（docker-compose 形态） |
| 学习文档 | 企业级部署手册 4 份（架构 / 部署 / 故障 / 总览，供对照阅读） |

## 目录结构

```
.
├── 6wei/                        # 6 个微服务独立清单（Nacos 指向内网 172.26.41.3:8848）
├── ruoyi-deploy/                # 中间件 + 微服务 + 监控采集（主要部署目录）
│   ├── mysql/                   #   MySQL 8.0 Deployment + Service
│   ├── redis/                   #   Redis 7.2.11-alpine Deployment + Service
│   ├── nacos-standalone-pod.yaml  # Nacos 单机 Pod（8848/9848/9849）
│   ├── nacos-service.yaml         # Nacos LoadBalancer Service（含 nodePort）
│   └── weifuwu/                 #   6 个微服务 + 6 个 ServiceMonitor（采集 9404 JMX 指标）
├── ruoyi-k8s/                   # 聚合清单：ruoyi-all.yaml（6 服务一体，Nacos 指向公网 IP）
│   └── ruoyi.yaml               # gateway 单服务示例
├── nginx-deployment/            # 前端 Nginx Deployment + LoadBalancer Service
├── nginx.conf                   # 前端 Nginx 配置（/prod-api → 网关 NodePort 30080）
├── skywalking/                  # SkyWalking OAP + UI + ES 的 docker-compose.yml
├── jmx-exporter/                # JMX Prometheus Javaagent 0.20.0（jar + config.yaml）
├── kube-prometheus-0.18.0-bear/ # kube-prometheus 0.18.0 定制版（manifests/jsonnet/grafana 大盘）
├── dingtalk/                    # 钉钉机器人 webhook 配置与告警消息模板
├── alertmanager-dingtalk.yaml   # AlertmanagerConfig：告警路由到钉钉
├── ruoyi-alert.yaml             # PrometheusRule：微服务 / JVM / 节点告警规则
├── test-alert.yaml              # 钉钉告警测试规则（人为触发）
├── ruoyi-crash-test.yaml        # 故障演练：busybox 故意退出，触发 CrashLoopBackOff 告警
├── ruoyi-enterprise-deploy/     # 学习手册（总览 / 系统架构 / 部署手册 / 故障处理）
└── workspace/                   # 工作目录（当前为空）
```

## 组件与端口

| 组件 | 版本 | 端口 | 说明 |
| --- | --- | --- | --- |
| ruoyi-gateway | 3.6.5 | 8080 | 微服务网关 |
| ruoyi-auth | 3.6.5 | 9200 | 认证服务 |
| ruoyi-modules-system | 3.6.5 | 9201 | 系统服务 |
| ruoyi-modules-gen | 3.6.5 | 9202 | 代码生成 |
| ruoyi-modules-job | 3.6.5 | 9203 | 定时任务 |
| ruoyi-modules-file | 3.6.5 | 9300 | 文件服务 |
| Nacos | 2.2.3 | 8848 / 9848 / 9849 | 注册与配置中心（HTTP / 客户端 gRPC / Raft gRPC） |
| MySQL | 8.0 | 3306 | 数据库 |
| Redis | 7.2 | 6379 | 缓存 |
| JMX exporter | 0.20.0 | 9404 | JVM 指标采集（挂载 javaagent） |
| SkyWalking OAP / UI | 10.4.0 | 11800 / 12800 / 8080 | 链路追踪 |
| Elasticsearch | 8.14.3 | 9200 | 链路/日志存储 |

镜像均来自**阿里云 ACR 个人版**（`crpi-*.cn-hangzhou.personal.cr.aliyuncs.com`），命名空间 `yanwu`（微服务）与 `batt_class_object`（中间件/前端/监控）。

## 部署链路（学习路径）

1. **中间件先行**：MySQL → Redis → Nacos（`ruoyi-deploy/`），确认注册中心可访问；
2. **业务微服务**：依次部署 auth/system/gen/job/file → gateway（`6wei/` 或 `ruoyi-deploy/weifuwu/`），服务在 Nacos 注册成功；
3. **前端接入**：Nginx Deployment + `nginx.conf`，`/prod-api` 代理网关；
4. **监控告警**：kube-prometheus（`kube-prometheus-0.18.0-bear/`）→ 部署 JMX exporter 与 ServiceMonitor → 导入 PrometheusRule → Alertmanager 推送钉钉；
5. **链路追踪**：SkyWalking（`skywalking/docker-compose.yml` 本地形态，K8s 版可参考部署手册）；
6. **故障演练**：`ruoyi-crash-test.yaml` 制造 CrashLoopBackOff，验证告警链路。

> 详细的架构说明、分步部署与故障排查请见 `ruoyi-enterprise-deploy/` 下 4 份文档（00 总览、01 架构、02 部署手册、03 故障手册）。

## 注意事项

- **敏感信息**：`dingtalk/config.yml` 内含钉钉机器人 webhook 的 `access_token` 与 `secret`，属凭据信息。本仓库为私有仓库，若未来转为公开或分享，请先移除并轮换该 token。
- **环境地址**：清单中的 IP 为学习环境地址（内网 `172.26.41.x`、公网 `120.27.247.65`、本机 `192.168.x.x`），部署到新环境时需按实际替换 `SPRING_CLOUD_NACOS_*` 环境变量与 `nginx.conf` 代理地址。
- **镜像仓库**：如需复用镜像，需将 ACR 镜像同步到自己的仓库，并在 Deployment 中配置对应的 `imagePullSecrets`（如 `aliyun-acr-secret`）。
- 学习环境全部单副本，滚动更新会出现短暂 502，属正常现象（非生产配置）。

## 参考

- [RuoYi-Cloud](https://gitee.com/y_project/RuoYi-Cloud)（3.6.5）
- [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus)（0.18.0）
- 阿里云 ACK 托管集群文档
