# Langfuse TKE (Tencent Kubernetes Engine) 部署指南

本目录包含了将 Langfuse 部署到腾讯云 TKE 的原生 Kubernetes YAML 配置文件。
该配置方案移除了对本地数据库、缓存及对象存储组件的依赖，转而使用腾讯云提供的托管云原生服务（PostgreSQL, Redis, ClickHouse, COS），从而实现更高可用性和弹性的生产级部署。

## 架构说明

- **Langfuse Web**: 负责处理前端请求与 API 流量，通过 TKE 的内网 CLB (LoadBalancer) 暴露。
- **Langfuse Worker**: 负责后台异步任务处理。
- **云原生依赖**:
  - **数据库**: 腾讯云云数据库 PostgreSQL
  - **缓存/队列**: 腾讯云云原生 Redis
  - **对象存储**: 腾讯云对象存储 (COS)，替代 MinIO，作为事件和媒体文件的存储。
  - **分析数据库**: 云端 ClickHouse (可使用腾讯云 CDW 或自建)。

## 部署步骤

### 1. 准备云端依赖资源

在部署前，请确保您已经在腾讯云控制台开通并配置好以下资源，并获取它们的连接信息（IP、端口、账号、密码、密钥）：

- **PostgreSQL 数据库**：建议创建专用的数据库（如 `langfuse`）及用户。
- **Redis 实例**：用于缓存和 BullMQ 任务队列。
- **ClickHouse**：分析型数据库。
- **对象存储 COS**：至少创建一个存储桶（例如 `langfuse-events`、`langfuse-media`、`langfuse-exports`），并创建一个拥有这些存储桶读写权限的 API 密钥 (SecretId 和 SecretKey)。

### 2. 配置 ConfigMap 和 Secret

打开 `01-config.yaml` 文件，根据您实际的云服务信息进行替换。

在 `ConfigMap` (`langfuse-config`) 中，需要替换的项包括（但不限于）：
- `NEXTAUTH_URL`: 替换为最终分配的 CLB 内网 IP 或内网域名。
- `DATABASE_URL`: PostgreSQL 的连接 URL。
- `CLICKHOUSE_MIGRATION_URL` & `CLICKHOUSE_URL`: ClickHouse 服务的连接地址。
- `REDIS_HOST`: Redis 的内网地址。
- 各种 `LANGFUSE_S3_*` 配置：配置为您的 COS Bucket 名称、Region 及 Endpoint 地址（注意：通常使用 COS 时 `*_FORCE_PATH_STYLE` 设为 `false`）。

在 `Secret` (`langfuse-secret`) 中，替换所有带有 `# CHANGEME` 注释的凭据：
- `NEXTAUTH_SECRET`: 使用 `openssl rand -base64 32` 生成。
- `SALT`: 自定义盐值。
- `ENCRYPTION_KEY`: 使用 `openssl rand -hex 32` 生成，必须是 64 字符十六进制。
- 数据库和 Redis 的密码。
- COS 的 `ACCESS_KEY_ID` 和 `SECRET_ACCESS_KEY`。

> **注意**: 建议不要将包含明文密码的 `01-config.yaml` 提交到公开的代码仓库。

### 3. 配置内网 CLB (LoadBalancer)

在 `02-langfuse-web.yaml` 中，`Service` 的配置包含如下注解（Annotation）：
```yaml
service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: "subnet-xxxxxx"
```
请将 `subnet-xxxxxx` 替换为您希望创建内网 CLB 的实际子网 ID。

### 4. 应用部署

在确保所有配置修改无误后，通过 `kubectl` 按顺序应用 YAML 文件：

```bash
# 1. 创建配置和密钥
kubectl apply -f 01-config.yaml

# 2. 部署 Langfuse Web 及内网 CLB Service
kubectl apply -f 02-langfuse-web.yaml

# 3. 部署 Langfuse Worker
kubectl apply -f 03-langfuse-worker.yaml
```

### 5. 验证部署

查看 Pod 的运行状态，确保 `langfuse-web` 和 `langfuse-worker` 都正常启动且处于 `Running` 状态：
```bash
kubectl get pods -l app=langfuse
```

查看 Service 获取分配的内网 CLB IP：
```bash
kubectl get svc langfuse-web-service
```
您现在可以通过获取到的 `<EXTERNAL-IP>`（此场景下为 CLB 的内网 IP）在内网环境访问 Langfuse Web 界面。
