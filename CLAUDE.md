
.000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000# panzhirui-workspace

学员个人 workspace 仓库，对应 `jianjiange-site` 组织下 dating 后端项目的个人开发空间。
包含 `dating-server`（Java 微服务 monorepo）、`ai-chat`（Python 数字人服务）、`proto`（gRPC 接口契约）三个子目录。

## 个人隔离前缀

共享开发环境（`38.76.188.242`）多人共用，以下资源必须按 `panzhirui` 前缀隔离，避免覆盖他人数据。

> 命名规则来自 `docs/dev-onboarding.md` 各章节的隔离说明：§2.3（PG）、§3.3（Redis）、§4.2（Nacos）、§5.3（MinIO）、§6.5（RocketMQ）、§7.6（OpenIM）、§8.1（Proto）。

| 资源 | 取值 | 参考章节 |
|------|------|----------|
| PG 库 | `dating_dev_panzhirui` | dev-onboarding §2.3 |
| Redis key 前缀 | `panzhirui:<service>:<domain>:<id>` | dev-onboarding §3.3 |
| Nacos namespace | `dev-panzhirui` | dev-onboarding §4.2 |
| Nacos Data ID | `<service>.yaml` / `<service>-dev.yaml`（**不带前缀**，namespace 已隔离） | dev-onboarding §4.3 |
| MinIO bucket | `dating-panzhirui` | dev-onboarding §5.3 |
| RocketMQ topic / group | `dev_panzhirui_<xxx>` | dev-onboarding §6.5 |
| OpenIM userID | `panzhirui_<id>` | dev-onboarding §7.6 |
| Proto 包坐标（Java） | `com.dating.panzhirui.proto:<service>-proto:<version>` | dev-onboarding §8.1 |
| Proto 包坐标（Python） | `dating-proto-panzhirui-<service>==<version>` | dev-onboarding §8.1 |
| Spring `application.name` | `<service>`（纯服务名，如 `post-service`） | dev-onboarding §1 |
| docker 容器 / image | `dating-<service>` | dev-onboarding §8.4 |

> ⚠️ 其他学员把 `panzhirui` 换成自己的拼音即可。

## 关联文档（权威，先读这些）

- [`docs/student-dev-guide.md`](docs/student-dev-guide.md) — 后端开发总规范，技术栈、Git、目录结构、红线全在这里
- [`docs/dev-onboarding.md`](docs/dev-onboarding.md) — 接入共享基建（远端 `38.76.188.242`）的连接配置 + 凭据
- [`docs/im-service-design.md`](docs/im-service-design.md) — IM 服务设计文档
- [`docs/match-service-prd-tech.md`](docs/match-service-prd-tech.md) — 匹配服务 PRD 与技术设计
- [`docs/post-service-design.md`](docs/post-service-design.md) — 朋友圈服务设计
- [`docs/user-service-design.md`](docs/user-service-design.md) — 用户服务设计
- [`docs/payment-service-design.md`](docs/payment-service-design.md) — 支付服务设计
- [`docs/mobile-gateway-design.md`](docs/mobile-gateway-design.md) — BFF 网关设计
- [`docs/local-infra-setup.md`](docs/local-infra-setup.md) — 在本机用 Docker 起一套自用中间件
- [`docs/development-roadmap.md`](docs/development-roadmap.md) — 服务间依赖关系与推荐开发顺序

## 硬约束（违反一票否决）

来自 `student-dev-guide §10 红线`，写代码前先对一遍：

1. **生产环境**密码 / AK SK / Token **绝不进 git**。学员**共享 dev 凭据**（`38.76.188.242` 那套）为方便快速上手，允许写入 workspace `nacos/<service>-<env>.yaml` 配置模板和 `docs/dev-onboarding.md` 速查表；**业务代码 / `application*.yml` / `.env*` 等运行时配置仍走 `${ENV}` 占位**，真值放 Nacos 或环境变量。
2. 持久层 **禁多表 JOIN**，跨表在 service 层多次单表查 + 内存拼装。
3. 服务间 **禁 HTTP 互调**，只用 gRPC；REST 仅对外（App / H5 / 第三方）。
4. 跨服务 **禁直连别人家的 DB / Redis key / 对象桶**，要数据就调对方 gRPC。
5. **时间一律 UTC**：DB 列 `TIMESTAMPTZ`、JVM `TZ=UTC`、连接 session `SET TIME ZONE 'UTC'`，代码/SQL 禁止写死 `Asia/Shanghai`。
6. **业务服务不直连 OpenIM / LiveKit**，IM / 音视频能力统一经 `im-service` gRPC，禁自建 WebSocket 长连。
7. **`.proto` 走 Nexus 包**：proto 文件放 `proto/`，发布坐标必须带 `panzhirui` 前缀（`com.dating.panzhirui.proto:*`），业务工程通过 `<dependency>` 拉取，不直接读 proto 文件路径。
8. **不要新增中间件**（ES / Mongo / ZK 等）未经评审。RocketMQ 已纳入基础组件清单，可直接使用。
9. **跨服务事务**：禁止 XA / Seata / 本地事务挂远程调用，跨服务一致性用消息 / 重试 / 对账。
10. **对外 API 暴露内部自增 `id`**：出参用业务主键（`user_id` / `post_id` / `order_id` 等），内部 `id` 仅在存储/索引层面使用。

## 技术栈

- Java 21 / Spring Boot 3.3.5 / MyBatis-Plus
- PostgreSQL 16（唯一关系库，禁 MySQL）
- Redis 7 单实例（必带 TTL + 学员前缀 `panzhirui:<service>:<domain>:<id>`）
- MinIO（S3 兼容，统一经 `dating-common` 的 `ObjectStorage` 接口，`path-style-access: true`）
- Nacos 2.4（Config + Discovery 同 namespace `dev-panzhirui`）
- RocketMQ 5.3.1（异步事件 / 写扩散 / 解耦，公网客户端必带 AK/SK，见 `dev-onboarding §6`）
- gRPC 1.68.1 + Protobuf 4.28.3
- Python ≥ 3.13（ai-chat 部分）
- OpenIM + LiveKit（IM / 1v1 音视频，统一经 `im-service` 编排）

## 服务内部分层

```
controller/  →  service  →  manager  →  mapper
   ↓              ↓            ↓          ↓
grpc/         client/       entity/    XML
```

- **调用方向严格单向**，禁止反向/跨层（如 Controller 直接调 Mapper）
- Controller / grpc 只做参数校验 + 编排 Service，**不做业务**
- Service 是事务边界，可调多个 Manager / 远程 gRPC client
- Manager 包装多次 Mapper 调用 + 缓存读写
- Mapper 一张表一个，**禁多表 JOIN**
- 跨服务调用经 `client/` 目录的 gRPC stub 封装

## Git 工作流

- `master` 受保护，只接 `dev → master` 的 PR
- `dev` 集成主线，个人分支 PR 合入
- 个人分支命名：`panzhirui/<topic>`，例 `panzhirui/post-feed-rank`
- Commit message 走 Conventional Commits：`<type>(<service>): <概要>`

  | type | 用途 |
  |------|------|
  | `feat` | 新功能 |
  | `fix` | Bug 修复 |
  | `refactor` | 重构（不改外部行为） |
  | `perf` | 性能优化 |
  | `docs` | 文档 |
  | `test` | 仅加测试 |
  | `chore` | 构建/CI/依赖 |

## 编码要求

### 注释（学员强制）

- **每个方法必须有 Javadoc**：功能描述 + 入参 + 出参 + 异常（public 方法必写完整，private 至少一行说明用途）
- **函数内每一逻辑块用单行注释划清**：让人先看注释串起流程，再看实现
- 写 **WHY** 不写 WHAT：不写套话，要写为什么这么做、前置条件、调用方注意什么

### 异常处理

- 业务异常继承 `BizException(code, message)`，全局 `@RestControllerAdvice` 兜底
- 不把堆栈抛给前端，系统异常返回通用 500 文案

### 事务

- `@Transactional(rollbackFor = Exception.class)` 加在 Service 方法
- 禁止跨服务事务（远程调用不参与本地事务）

### 日志

- SLF4J，禁止 `System.out.println`
- 日志走 stdout（禁写文件），ERROR 必带堆栈
- 关键链路打 traceId / userId（MDC 透传）
- 禁止打印敏感字段

### DTO ↔ Entity 转换

- 手写转换或用 **MapStruct**，禁用 `BeanUtils.copyProperties`

### 缓存

- Cache Aside：先写库，再删缓存，禁双写
- 分布式锁统一 Redisson，必须设 `leaseTime`

## 协作偏好

- 回答简洁，直接给结论 / diff，不要长段铺垫总结。
- 我是后端初学者，解释时兼顾"怎么做"和"为什么这么做"。
- 涉及连接配置、密码、隔离前缀时，**先核对 `docs/` 里的权威文档**，不要凭记忆生成。
- 触发本文写的红线时，必须明确指出并拒绝，不要默默"兼容"实现。
- 引导我养成后端好习惯：写注释、处理异常、注意时区、不暴露内部 id、不给 Redis 设永久 key。
- 生成代码时优先考虑可读性和教学价值而非炫技。
