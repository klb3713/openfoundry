# OpenFoundry 技术架构与实现

> **一份基于源码逐行核实的架构与实现说明**
>
> - 代码快照：`master` 分支 `96ff1de`
> - 文档版本：v1.0 · 2026-11-09
> - 覆盖范围：49 个 workspace 包（33 个库 + 13 个服务 + 3 个应用），约 114,000 行 TypeScript/TSX
> - 证据口径：关键结论均附 `相对路径:行号`；文档与实现冲突处以代码为准，并用「文档与实现不一致」引用块标注

## 目录

- [第 1 章 文档说明](#第-1-章-文档说明)
- [第 2 章 项目定位与全景](#第-2-章-项目定位与全景)
- [第 3 章 总体架构](#第-3-章-总体架构)
- [第 4 章 契约层与 SDK](#第-4-章-契约层与-sdk)
- [第 5 章 身份、权限与多租户安全](#第-5-章-身份权限与多租户安全)
- [第 6 章 本体与对象数据面](#第-6-章-本体与对象数据面)
- [第 7 章 平台业务服务：动作、函数、自动化与协作](#第-7-章-平台业务服务动作函数自动化与协作)
- [第 8 章 数据与 AI：数据集、存储引擎与 AIP](#第-8-章-数据与-ai数据集存储引擎与-aip)
- [第 9 章 前端应用](#第-9-章-前端应用)
- [第 10 章 工程化与交付](#第-10-章-工程化与交付)
- [第 11 章 横切基础库](#第-11-章-横切基础库)
- [第 12 章 测试体系与质量门禁](#第-12-章-测试体系与质量门禁)
- [第 13 章 架构评估、已知限制与风险](#第-13-章-架构评估已知限制与风险)
- [第 14 章 演进建议](#第-14-章-演进建议)
- [附录](#附录)

## 第 1 章 文档说明

### 1.1 目的与读者

本文档描述 OpenFoundry 的**技术架构与实现方式**：它由哪些进程和库组成、一次请求如何穿过它们、关键数据模型长什么样、哪些设计是有意为之、哪些地方已知不可靠。

面向三类读者：

- **新加入的工程师**：需要在一到两天内建立完整的代码地图，知道改一个功能要动哪些包。
- **架构评审者**：需要判断这套"Foundry 模拟器"的边界在哪里、哪些能力是真实现、哪些是桩。
- **二次开发者与运维**：需要知道启动方式、存储模式切换、端口分配与已知故障模式。

它**不**替代两份既有文档：

| 文档 | 定位 | 与本文的关系 |
|------|------|--------------|
| `README.md` | 快速上手与端点速查 | 本文核实其声明，冲突处以代码为准并标注 |
| `GUIDE.md` | 印尼语用户手册，端到端验证过的产品行为 | 本文引用其"已知限制"作为佐证，不重复产品操作步骤 |
| `docs/foundry-fidelity.md` | 权限模型与 Palantir Foundry 的逐条对照 | 第 5 章在其基础上展开实现细节 |

### 1.2 范围

**在范围内**：进程拓扑与端口、服务模板、网关分发、请求生命周期、领域模型、存储层、鉴权与多租户、契约与代码生成、前端应用、构建与交付、测试体系、已知缺陷与架构风险。

**不在范围内**：具体业务操作的教学（见 `GUIDE.md`）、Palantir Foundry 本身的产品说明、逐字段的 API 参考（见 `README.md` 与 `conjure/*.yml`）。

### 1.3 方法与证据口径

本文的每一条结论都来自对仓库源码的阅读，而不是对 README 的转述。写作遵循以下口径：

1. **快照**：`master` 分支 commit `96ff1de`（"feat(svc-datasets): serve the Foundry connectivity API (18 of 20 operations)"）。
2. **以代码为准**：README、`GUIDE.md`、源码注释都可能滞后。凡发现文档与实现冲突，正文用引用块标注：
   > 文档与实现不一致：……
3. **证据定位**：关键结论后附 `相对路径:行号`，例如 `services/svc-gateway/src/server.ts:191`。
4. **不确定性显式化**：未能通过代码或运行验证的推断标「（未验证）」，不写成事实。
5. **数字口径**：代码行数统计 `.ts`/`.tsx`/`.mts`，排除 `node_modules`、`dist`；包数量按 `pnpm-workspace.yaml` 的四个 glob 统计。

### 1.4 术语约定

| 术语 | 含义 | 在本仓库中的位置 |
|------|------|------------------|
| Ontology（本体） | 一组对象类型、链接类型、动作类型的集合，是 Foundry 的数据建模层 | `packages/ontology-schema`、`services/svc-ontology` |
| Object Type（对象类型） | 带属性 schema 的实体类型，如 `PestControlOrder` | `conjure/ontology-service.yml`、`svc-objects` |
| ObjectSet | 对一组对象的声明式描述（基础集、过滤、并/交/差、searchAround、静态集） | `packages/object-set` |
| Action Type（动作类型） | 对对象做受控写操作的可执行定义 | `services/svc-actions` |
| RID | Foundry 风格资源标识符，形如 `ri.ontology.main.object-type.xxx` | `packages/rid` |
| Marking | 强制访问控制标签，用户必须持有全部 Marking 才能访问 | `packages/markings`、`packages/permissions` |
| Grant / Role | 资源上的角色授予（owner/editor/viewer/discoverer） | `packages/permissions` |
| Branch / Transaction | **数据集**的分支与写入事务；本体没有分支概念 | `services/svc-datasets` |
| Tenant（租户） | 以 `org_rid` 标识的组织，Postgres 模式下由行级安全（RLS）隔离 | `db/migrations`、各 Pg store |

> 特别注意：本仓库中出现的每一个 `branch` 都是**数据集分支**，不是本体分支。控制台的 Scenarios 页面不持久化任何东西，v2 本体端点会显式拒绝 `branch`/`scenarioRid`/`transactionId` 参数（策略集中在 `packages/errors/src/ontology-scoping.ts`）。

### 1.5 全文结构

| 章 | 主题 | 回答的问题 |
|----|------|-----------|
| 1 | 文档说明 | 本文的范围与证据口径 |
| 2 | 项目定位与全景 | 这是什么项目、多大、用什么技术、怎么跑起来 |
| 3 | 总体架构 | 分层、进程边界、一次请求怎么走、存储双模 |
| 4 | 契约层与 SDK | API 形状从哪里来，v1/v2 为什么是两套 API |
| 5 | 身份、权限与多租户安全 | 谁在什么条件下能访问什么 |
| 6 | 本体与对象数据面 | 本体元数据、对象存储、ObjectSet 求值、搜索与聚合 |
| 7 | 平台业务服务 | 动作、函数、自动化、Webhook、媒体、资源树、管理后台 |
| 8 | 数据与 AI | 数据集与事务、数据库与迁移、AIP 与 LLM 策略 |
| 9 | 前端应用 | 控制台、Workshop、文档站与组件库 |
| 10 | 工程化与交付 | 构建、类型、Lint、测试、CI、容器与编排 |
| 11 | 横切基础库 | 错误、RID、分页、日志、追踪、健康、配置、Redis、测试工具 |
| 12 | 测试体系与质量门禁 | 测试怎么分层、怎么跑、当前基线是多少 |
| 13 | 架构评估与已知限制 | 哪些设计好、哪些地方会坏、风险清单 |
| 14 | 演进建议 | 如果要继续做，优先级是什么 |
| 附录 | 端口、包清单、术语、端点 | 速查 |

---

## 第 2 章 项目定位与全景

### 2.1 这是什么

OpenFoundry 是 **Palantir Foundry 的本地模拟器（emulator）**：它实现 Foundry 风格的 HTTP API（`/api/v2` 为主、`/api/v1` 为辅），用内存或本地文件（可选 PostgreSQL）存储数据，让开发者、测试人员和售前演示在没有 Foundry 授权的情况下，也能跑起一套"看起来像 Foundry"的平台。

项目自称的定位写在 `package.json` 的 `description` 里：`"Open-source data platform with ontology-first architecture"`（`package.json:4`）。但更准确的描述来自 README 第一句与代码结构：**ontology-first 是它的领域模型选择，Foundry 兼容是它的产品目标**。

判断它是不是"真 Foundry"，看三个硬指标，它在代码里都做了对应实现：

| 指标 | 实现位置 | 说明 |
|------|----------|------|
| 线上模型形状与 Foundry 一致 | `conjure/*.yml`、`packages/api-types` | 用 Conjure IDL 描述端点与模型，生成 TS 类型 |
| 错误信封与 Foundry 一致 | `packages/errors/src/api-error.ts` | `errorCode`/`errorName`/`errorInstanceId`/`parameters`/`statusCode` |
| 资源标识符与权限模型一致 | `packages/rid`、`packages/permissions`、`docs/foundry-fidelity.md` | `ri.*` RID、Organization → Marking → Role 三层检查 |

### 2.2 三个设计目标与它们的取舍

**目标一：仿真保真度优先于功能完备度。**
它宁可只实现 20 个端点中的 18 个，也不返回形状不对的响应。`services/svc-datasets/src/routes/v1/files.ts` 把未实现的操作记录在"本该在这里"的位置，让调用方拿到 404 而不是错误形状；`services/svc-gateway/src/routes/v1/index.ts` 明确不代理 `attachments` 命名空间，因为"注册一个前缀只会返回 502，而调用方应该拿到 404"。

**目标二：零外部依赖可运行。**
不设 `DATABASE_URL` 时，服务用内存 + `/tmp/openfoundry-data` 下的 JSON 文件；没有 `ANTHROPIC_API_KEY`/`OPENAI_API_KEY` 时，AIP 用内置的 mock LLM；Redis 永远不是必需项（限流退化到进程内令牌桶）。这是"一条命令起全栈"（`bash start.sh`）的基础。

**目标三：单仓可维护。**
13 个服务共享 33 个库包，靠 pnpm workspace + Turborepo 组织。代价是构建拓扑深（`build` 依赖 `^build`），收益是类型与契约只写一遍。

**它不是生产系统**，这一点必须说清楚：没有高可用、没有消息队列、没有分布式事务；内存模式没有租户隔离（`services/svc-gateway/src/server.ts:46-53` 在 production 且无 `DATABASE_URL` 时直接抛错拒绝启动，正是承认这一点）；13 个服务里有 3 个甚至没有被 `start.sh` 启动。

### 2.3 规模与技术栈

截至快照 commit，仓库的客观规模：

| 维度 | 数量 | 统计方式 |
|------|------|----------|
| TypeScript/TSX 源码行数 | 约 114,000 行 | `find` 排除 `node_modules`/`dist` 后 `wc -l` |
| workspace 包 | 49 个 | 33 个库 + 13 个服务 + 3 个前端应用 |
| 后端服务 | 13 个（10 个被 `start.sh` 启动） | `services/*/package.json` 与 `start.sh:118` |
| HTTP 端口 | 8080–8092 | 各服务 `src/config.ts` 的 `PORT` 默认值 |
| 测试文件 | 约 100 个 `.test.ts` | 排除 `node_modules` |
| Conjure 契约 | 8 个 YAML，约 1,950 行 | `conjure/*.yml` |

技术栈：

| 层 | 选型 | 版本/说明 | 证据 |
|----|------|-----------|------|
| 运行时 | Node.js | `>=20`，实测 Node 26 亦可 | `package.json` engines |
| 包管理 | pnpm | 10.27.0，`workspace:*` 协议 | `packageManager`、`pnpm-workspace.yaml` |
| 构建编排 | Turborepo | 2.5.x，任务图与缓存 | `turbo.json` |
| 语言 | TypeScript | 5.5.x，`strict` + `noUnusedLocals/Parameters` | `tsconfig.base.json` |
| HTTP 框架 | Fastify | 5.3.x，每服务一个实例 | `services/*/package.json` |
| 鉴权 | jose + 自签 JWT | OAuth2 授权码 + PKCE、client_credentials | `packages/auth-tokens` |
| 存储 | 内存 / JSON 文件 / PostgreSQL | `pg` 8.13，PGlite 用于测试 | `packages/db` |
| 缓存/限流 | ioredis（可选） | 无 Redis 时进程内令牌桶 | `packages/redis`、`svc-gateway/src/rate-limiter.ts` |
| 日志 | pino（Fastify 内置） | 结构化 JSON，开发态 pino-pretty | `packages/logging` |
| 前端 | React 18 + Vite + Blueprint v5 | `@blueprintjs/*` 5.x，react-router-dom 7 | `apps/app-console/package.json` |
| 测试 | Vitest 3 | 根配置 + 每包 `tests/` + 集成测试配置 | `vitest.config.ts`、`tests/vitest.config.ts` |

### 2.4 仓库布局

```
openfoundry/
├── apps/            3 个前端应用：app-console（控制台）、app-workshop（低代码）、app-docs（文档站）
├── conjure/         8 个 Conjure IDL 契约（ontology/object/action/dataset/compass/admin/auth/common）
├── db/migrations/   PostgreSQL 迁移（含 org_rid 与 RLS）
├── deploy/          docker/（Dockerfile、nginx）、docker-compose*.yml、helm/openfoundry
├── docs/            foundry-fidelity.md（权限模型对照）、本文档
├── packages/        33 个库：契约与 SDK（6）、本体（5）、数据面（4）、平台能力（8）、可观测性（5）、测试（1）…
├── scripts/         启动种子、迁移、备份、演示数据、鉴权辅助
├── services/        13 个 Fastify 服务
├── tests/           integration/ 与 sdk-compat/ 跨服务测试
├── start.sh         本地一键编排（清端口 → 起 Postgres（可选）→ 起服务 → 播种 → 起前端）
└── turbo.json       build/test/typecheck/generate 任务图
```

`packages/` 内部可以再分四组，这个分组对理解依赖方向很重要：

| 分组 | 包 | 特征 |
|------|----|------|
| 契约与客户端 | `conjure-ir`、`conjure-compiler`、`conjure-codegen`、`api-types`、`wire`、`client`、`sdk`（含 sdk-oauth/sdk-react/sdk-cli） | 只依赖彼此，不依赖服务 |
| 领域模型 | `ontology-schema`、`ontology-ir`、`ontology-generator`、`ontology-maker`、`object-set` | 纯数据与算法，零 I/O |
| 平台能力 | `permissions`、`markings`、`auth-tokens`、`rid`、`pagination`、`errors`、`search`、`pipeline`、`db`、`config`、`redis`、`health`、`logging`、`tracing` | 被服务消费，少数依赖 `pg`/`fastify` |
| 前端与测试 | `ui-components`、`ui-icons`、`ui-tokens`、`test-utils` | 只被应用或测试消费 |

### 2.5 运行时拓扑

```
                        ┌──────────────────────────────┐
                        │  app-console  :3000 (Vite)   │
                        │  app-workshop / app-docs     │
                        └──────────────┬───────────────┘
                                       │ HTTP /api/v1, /api/v2, /multipass
                        ┌──────────────▼───────────────┐
                        │   svc-gateway  :8080         │
                        │   CORS/Helmet/限流/JWT/代理   │
                        └──┬───┬───┬───┬───┬───┬───┬───┘
        ┌──────────────────┘   │   │   │   │   │   └──────────────────┐
        ▼                      ▼   ▼   ▼   ▼   ▼                      ▼
  svc-ontology :8081   svc-objects :8082  svc-actions :8083   svc-multipass :8084
  svc-datasets :8085   svc-admin :8087    svc-functions :8088  svc-aip :8092
  svc-sentinel :8091   svc-compass :8086  svc-webhooks :8089   svc-media :8090
        │                      │                    │
        └──────────┬───────────┴────────┬───────────┘
                   ▼                    ▼
        ┌────────────────────┐  ┌──────────────────────┐
        │ 内存 / JSON 文件    │  │ PostgreSQL（可选）     │
        │ /tmp/openfoundry-  │  │ 迁移 + org_rid + RLS  │
        │ data/*.json        │  │                      │
        └────────────────────┘  └──────────────────────┘
```

端口与进程（全部可在各服务 `src/config.ts` 与 `start.sh` 中交叉验证）：

| 端口 | 服务 | 职责 | `start.sh` 启动 |
|------|------|------|------------------|
| 8080 | svc-gateway | 唯一对外入口：鉴权、限流、代理、跨实体搜索 | 是 |
| 8081 | svc-ontology | 本体元数据 CRUD、对象/链接/动作类型 | 是 |
| 8082 | svc-objects | 对象与链接数据、ObjectSet 求值、聚合、语义搜索 | 是 |
| 8083 | svc-actions | 动作校验与执行（写对象） | 是 |
| 8084 | svc-multipass | 认证与令牌签发、用户/组/客户端 | 是 |
| 8085 | svc-datasets | 数据集、分支、事务、文件、connectivity | 是 |
| 8086 | svc-compass | 资源树（空间/项目/文件夹）与血缘 | **否** |
| 8087 | svc-admin | 平台管理：用户、组、审计、资源角色 | 是 |
| 8088 | svc-functions | 用户函数注册与执行 | 是 |
| 8089 | svc-webhooks | Webhook 注册与投递 | **否** |
| 8090 | svc-media | 媒体上传/引用（S3 可选） | **否** |
| 8091 | svc-sentinel | 监控、调度、副作用 | 是（但 `start.sh` 不检查该端口） |
| 8092 | svc-aip | AIP Chat、模型注册、嵌入 | 是 |
| 3000 | app-console | Vite 开发服务器 / 控制台 | 是 |

> 文档与实现不一致：README 称"10 个后端服务"、`packages/` "28 packages"；实际是 13 个服务、33 个库包（49 个 workspace 包）。README 也承认 svc-compass/svc-webhooks/svc-media 未被 `start.sh` 启动，但没有解释原因——代码里它们与其它服务结构相同，只是启动脚本的 `SERVICES` 数组没列（`start.sh:118`）。

### 2.6 三种运行形态

| 形态 | 入口 | 存储 | 适用场景 |
|------|------|------|----------|
| 本地开发 | `bash start.sh`（`--reseed`、`--industry <name>`） | 有 Docker 则起 Postgres 容器，否则内存 + JSON | 开发、演示、调试 |
| Docker Compose | `docker compose up --build`（根目录或 `deploy/`） | 容器内 Postgres + 数据卷 | 单机演示、交付给客户 |
| Kubernetes | `deploy/helm/openfoundry` | 外部 Postgres（迁移 Job） | 多服务部署 |

三者的细节（Dockerfile 选择包的方式、Helm 模板、环境变量）在第 10 章展开。这里只需要记住一个结论：**同一份代码在三种形态下行为一致，差异只来自 `DATABASE_URL` 是否设置**——存储层是所有差异的收敛点。

### 2.7 能力矩阵与"真实现 vs 桩"

README 的能力表（47 个害虫防治对象、7 种 ObjectSet、13 个过滤算子、6 个聚合指标等）基本可信，但读者需要区分三类能力：

| 类型 | 例子 | 判断方法 |
|------|------|----------|
| 真实现 | ObjectSet 求值、过滤算子、聚合、动作副作用、数据集事务、权限模型 | 有对应实现与单元/集成测试 |
| 有实现的模拟 | AIP Chat（无 API key 时用规则 mock）、函数执行、Sentinel 调度 | 行为可观察，但不是真实计算/真实 LLM |
| 仅有路由或未接入 | svc-compass/svc-webhooks/svc-media 未启动；部分 Foundry 端点 404 | 网关返回 502（服务未起）或 404（未实现） |

第 13 章会把"已确认不可用"的清单集中列出，并标注每一条的证据。

---

## 第 3 章 总体架构

### 3.1 分层视图

```
┌──────────────────────────────────────────────────────────────────────┐
│ 客户端层   app-console / app-workshop / app-docs / SDK / 脚本         │
├──────────────────────────────────────────────────────────────────────┤
│ 接入层     svc-gateway：CORS → Helmet → 限流 → 请求 ID → 错误处理      │
│            → JWT 鉴权 → 路由分发（代理 / 路径嗅探 / 网关自实现）        │
├──────────────────────────────────────────────────────────────────────┤
│ 服务层     13 个 Fastify 服务，每个 = config + server + routes + store │
│            ontology · objects · actions · datasets · multipass · admin │
│            functions · aip · sentinel · compass · webhooks · media     │
├──────────────────────────────────────────────────────────────────────┤
│ 领域层     ontology-schema/ir/generator/maker · object-set · search     │
│            permissions · markings · pipeline · rid · pagination       │
├──────────────────────────────────────────────────────────────────────┤
│ 契约层     conjure/*.yml → conjure-ir → conjure-compiler → codegen     │
│            → api-types / wire → 服务序列化器与 SDK                     │
├──────────────────────────────────────────────────────────────────────┤
│ 存储层     内存 Map/数组 → JSON 文件（/tmp/openfoundry-data）          │
│            或 PostgreSQL（迁移 + org_rid + RLS）                       │
├──────────────────────────────────────────────────────────────────────┤
│ 横切层     errors · logging · tracing · health/metrics · config · redis │
└──────────────────────────────────────────────────────────────────────┘
```

这个分层不是目录的简单归纳，而是**依赖方向**：箭头只允许向下。服务层可以 import 领域层与横切层，反之不行；契约层不依赖任何服务（`packages/api-types` 的 `dependencies` 为空，见其 `package.json`），所以服务与 SDK 能各自消费同一份生成物。

### 3.2 进程边界

13 个服务是 13 个独立进程，各自监听端口、各自创建 Fastify 实例、各自持有存储连接。它们之间**没有消息队列、没有服务网格、没有共享进程内状态**，协作方式只有两种：

1. **同步 HTTP 调用**（少数）：如 svc-actions 调用 svc-objects 的 HTTP API 写对象，svc-datasets 调用 svc-ontology 读取类型信息，svc-admin 的统计路由并发调用多个服务。
2. **共享持久化状态**（多数）：多个服务读写同一批 JSON 文件或同一个 Postgres 库。

第 2 种方式是这套架构最容易被误解的地方：服务在**代码上**是独立的，在**数据上**却通过约定好的文件/表共享。第 3.7 节给出完整清单。

### 3.3 服务模板：每个服务长什么样

除网关外，12 个服务共享同一套骨架，以 svc-objects 为例：

```
services/svc-objects/
├── src/index.ts          进程入口：createServer() → listen()
├── src/config.ts         环境变量 → 配置对象（PORT、DATABASE_URL、LOG_LEVEL…）
├── src/server.ts         服务器工厂：CORS、请求 ID、错误处理、鉴权、路由注册
├── src/routes/v2/*.ts    v2 路由（按资源分文件）
├── src/routes/v1/*.ts    v1 路由与序列化器
├── src/store/*.ts        存储实现（内存 + Pg 两套）
└── tests/*.test.ts       单元/路由测试
```

`createServer()` 里注册的中间件顺序几乎是复制粘贴的（`services/svc-objects/src/server.ts:70-160`）：

| 顺序 | 内容 | 作用 |
|------|------|------|
| 1 | `@fastify/cors` | 允许浏览器跨域直连服务端口（开发便利） |
| 2 | `onSend` 钩子 | 回写 `x-request-id` |
| 3 | `setErrorHandler` | 把 `OpenFoundryApiError`、Fastify 校验错误、未知错误映射成统一信封 |
| 4 | `registerAuthorization` | 把权限存储挂到实例上，供 `requirePermission` 使用 |
| 5 | 路由注册 | v2 前缀 `/api/v2`、v1 前缀 `/api/v1` |

**这套模板的代价**：每个服务重复约 40–60 行中间件代码，且没有共享的 `createBaseServer()` 工厂。好处是服务之间零耦合、单服务可以独立读懂。第 14 章会把它列为可演进项。

### 3.4 网关：三种分发模式

网关是唯一对外入口（`:8080`），但它不是简单的反向代理。按实现方式分三类：

**模式一：整段前缀代理。** 大多数资源组用 `app.all("/")` + `app.all("/*")` 把整个前缀转发给对应服务，例如 datasets → svc-datasets（`services/svc-gateway/src/routes/v2/datasets.ts:8-16`）。`proxyRequest` 负责剥离逐跳头、剥离客户端伪造的 `x-user-*`、丢弃 `Content-Length`、注入 `X-Forwarded-For` 与 `X-Request-Id`、上游不可达时返回 502（`services/svc-gateway/src/proxy.ts:50-140`）。

**模式二：路径嗅探分发。** `/api/v2/ontologies/*` 一个前缀背后有三个服务，网关按 URL 正则决定去向（`services/svc-gateway/src/routes/v2/ontologies.ts:24-52`）：

```ts
// /ontologies/{rid}/objects/... 或 .../objectSets/... → svc-objects
if (/\/ontologies\/[^/]+\/objects(\/|$)/.test(url) ||
    /\/ontologies\/[^/]+\/objectSets(\/|$)/.test(url)) {
  return proxyRequest(request, reply, app.config.services.objects);
}
// /ontologies/{rid}/actions/{action}/apply|validate → svc-actions
if (/\/ontologies\/[^/]+\/actions\/[^/]+\/(apply|validate)$/.test(url)) {
  return proxyRequest(request, reply, app.config.services.actions);
}
// 其余 → svc-ontology
```

这是**本仓库最脆弱的耦合点之一**：新增一个 `/ontologies/:rid/...` 端点时，如果正则没覆盖到，请求会静默落到 svc-ontology 并 404，而不是报错。第 13 章把它列为风险项。

**模式三：网关自实现。** 少数端点不代理，直接在网关里实现：跨实体搜索 `/api/v2/search`（网关直连 Postgres 搜索索引并按权限过滤，`services/svc-gateway/src/routes/v2/search.ts`）、健康检查 `/status/health`、指标 `/metrics`、以及 `/multipass/*` 的认证入口。

v1 的处理更保守：只注册 `datasets` 与 `ontologies` 两个命名空间（`services/svc-gateway/src/routes/v1/index.ts:20-36`），`attachments` 故意不注册——未注册前缀返回 Fastify 的 404，比代理到一个不存在的后端返回 502 更诚实。

### 3.5 请求生命周期

#### 3.5.1 读路径：GET 一个对象

```mermaid
sequenceDiagram
    participant B as 浏览器 (app-console)
    participant G as svc-gateway :8080
    participant O as svc-objects :8082
    participant S as 存储 (Map/JSON 或 Postgres)

    B->>G: GET /api/v2/ontologies/{rid}/objects/{type}/{pk}<br/>Authorization: Bearer <JWT>
    G->>G: CORS → Helmet → 限流 → x-request-id
    G->>G: authPlugin 验签（AUTH_PUBLIC_KEY 已设置时）
    G->>G: 路径嗅探：/objects/ → svc-objects
    G->>O: 转发（剥离 x-user-*，写入已验证身份，丢弃 Content-Length）
    O->>O: onRequest：从 x-user-* 读身份
    O->>O: requirePermission(DISCOVER/VIEW)
    O->>S: store.getObject(ontologyRid, type, pk)
    S-->>O: 对象或 null
    O-->>G: 200 + v2 投影 JSON（或 404 错误信封）
    G-->>B: 透传状态码、头、体
```

关键点：

- **网关不做业务校验**，它只负责身份、限流、转发与错误归一化；对象是否存在、权限是否足够由 svc-objects 判断。
- **身份以 `x-user-*` 头传递**，网关从自己验过的 JWT 生成，客户端发来的同名头一律丢弃（`services/svc-gateway/src/proxy.ts:36-46`、`79-84`）。没有验证过的令牌时不写任何身份头，下游看到匿名请求。
- **存储调用必须 `await`**：`ObjectReadWriteStore` 的方法返回 `T | Promise<T>`，内存实现同步、Pg 实现异步。漏掉 `await` 在内存测试里完全正常，在 Postgres 下会把 pending Promise 序列化成 200（第 6 章详述）。

#### 3.5.2 写路径：POST 执行一个动作

```mermaid
sequenceDiagram
    participant B as 浏览器
    participant G as svc-gateway
    participant A as svc-actions :8083
    participant O as svc-objects :8082
    participant S as 存储

    B->>G: POST /api/v2/ontologies/{rid}/actions/{action}/apply
    G->>G: 验签 + 限流 + 路径嗅探（/actions/.../apply → svc-actions）
    G->>A: 转发
    A->>A: 校验动作类型、参数、目标对象
    A->>O: HTTP 调用对象写接口（objects-client.ts）
    O->>S: 写入对象/链接
    S-->>O: 新版本
    O-->>A: 200/4xx
    A-->>G: 动作结果（含创建/修改的对象）
    G-->>B: 透传
```

关键点：

- svc-actions **不直接持有对象存储**，它通过 HTTP 调 svc-objects（`services/svc-actions/src/objects-client.ts:61`）。这意味着一次动作至少两次 HTTP 跳转，且**没有跨服务事务**：对象写入成功、动作日志写入失败时会出现部分状态（第 7 章核实具体边界）。
- 写路径的权限检查发生在 svc-actions（动作资源），而对象层的权限检查发生在 svc-objects；两处都调用同一个 `requirePermission`，但检查的资源不同。

#### 3.5.3 失败分支

| 场景 | 表现 | 来源 |
|------|------|------|
| 令牌缺失或无效 | 401（`AUTH_PUBLIC_KEY` 已设置时）；未设置时全部放行 | 网关 `middleware/auth.ts` |
| 权限不足 | 403 + `PERMISSION_DENIED` 信封 | `packages/permissions` |
| 资源不存在 | 404 + 具体 `errorName`（如 `ObjectNotFound`） | 各服务错误工厂 |
| 请求体非法 JSON | 400 `InvalidRequest`（Fastify 4xx 透传，不包装成 500） | `services/svc-gateway/src/server.ts:139-155` |
| 后端服务未启动 | 502 `BadGateway` | `services/svc-gateway/src/proxy.ts:120-131` |
| 空 body 却带 `Content-Type: application/json` | 400 `FST_ERR_CTP_EMPTY_JSON_BODY` | Fastify 行为，控制台多个页面踩过 |

### 3.6 身份在链路上的位置

| 层 | 拿到什么 | 做什么 | 不做什么 |
|----|----------|--------|----------|
| 浏览器 | 用户名/密码或已有 JWT | 在 `authFetch` 里统一加 `Authorization` | 不判断权限 |
| 网关 | 原始 JWT | 验签、取 claims、生成 `x-user-*`、剥离客户端伪造头 | 不查资源、不查 grant |
| 服务 | `x-user-*`（或匿名） | 用 `requirePermission` 查 grant/policy | 不重新验签 |
| 存储 | `org_rid` 等 | RLS 兜底租户隔离 | 不理解角色 |

这条链的核心假设是：**服务端口不可从公网直达**。一旦攻击者能直连 `:8082`，他就能自己伪造 `x-user-*`。`start.sh` 与本仓库的部署都默认服务只监听本机/集群内网，但这个假设没有被代码强制（第 13 章列为风险）。

### 3.7 存储双模与共享状态

#### 3.7.1 切换条件

`DATABASE_URL` 是唯一开关。每个服务的 `server.ts` 在启动时判断：

```ts
if (options.store) store = options.store;                    // 测试注入
else if (config.databaseUrl) store = new PgObjectStore(pool); // Postgres
else store = new ObjectStore();                               // 内存 + 文件
```

（`services/svc-objects/src/server.ts:55-67`）

网关还额外加了一条生产安全阀：`NODE_ENV=production` 且没有 `DATABASE_URL` 时拒绝启动，理由写在错误信息里——"内存存储没有租户隔离"（`services/svc-gateway/src/server.ts:46-53`）。

#### 3.7.2 无 DATABASE_URL 时的持久化范围（重要）

并非所有服务都会落盘。全仓搜索 `writeFileSync`/`readFileSync` 后可以确认，**只有三类数据会持久化**：

| 文件 | 写入方 | 内容 |
|------|--------|------|
| `/tmp/openfoundry-data/ontology-store.json` | svc-ontology | 本体、对象类型、链接类型、动作类型 |
| `/tmp/openfoundry-data/object-store.json` | svc-objects | 对象数据（按 ontology 作用域，`formatVersion: 2`） |
| `/tmp/openfoundry-data/link-store.json` | svc-objects | 对象链接 |
| `/tmp/openfoundry-data/authorization.json` | `packages/permissions` | 所有服务的 grant 与 policy（共享一个文件） |

其余服务（datasets、admin、aip、functions、compass、sentinel、multipass 的 token 等）在无 `DATABASE_URL` 时是**纯进程内内存**，重启即丢。这带来一个实际后果：

> `start.sh` 只在"没有 `ontology-store.json`"或显式 `--reseed`/切换行业时播种。重启后本体还在、数据集/函数/AIP 会话却清空，而播种不会重跑，控制台对应页面会显示空数据。这是"部分持久化"设计留下的坑，第 13 章给出复现路径。

#### 3.7.3 每个服务的存储双实现

| 服务 | 内存实现 | Postgres 实现 | 无 DB 时落盘 |
|------|----------|----------------|---------------|
| svc-ontology | `OntologyStore` | 同名 Pg 实现（`store/`） | 是 |
| svc-objects | `ObjectStore` + `LinkStore` | `PgObjectStore` | 是 |
| svc-datasets | `DatasetStore`/`FileStore`/`PipelineStore` | `PgDatasetStore`/`PgFileStore`/`PgPipelineStore` | 否 |
| svc-admin | `UserStore`/`GroupStore`/`AuditStore` | `PgUserStore`/`PgGroupStore`/`PgAuditStore` | 否 |
| svc-aip | `ConversationStore`/`EmbeddingStore` | `PgConversationStore`/`PgEmbeddingStore` | 否 |
| svc-functions | `FunctionStore` | `PgFunctionStore` | 否 |
| svc-compass | `ResourceStore`/`LineageStore` | `PgResourceStore`/`PgLineageStore` | 否 |
| svc-actions | `ActionRegistry`/`ActionLog` | `PgActionRegistry`/`PgActionLog` | 否 |
| svc-sentinel | 内存 store | 部分 | 否 |
| svc-multipass | `UserStore`/`ClientStore`/`TokenStore` | `PgUserStore` | 否 |
| svc-media | `MediaStore` | `S3MediaStore`（对象存储而非 PG） | 否 |

这张表解释了两件事：为什么"重启后数据还在"只对本体/对象成立；以及为什么 `DATABASE_URL` 一开，一批路由会立刻坏掉（同步调用异步 store，见第 7、13 章）。

### 3.8 服务间 HTTP 调用

| 调用方 | 被调方 | 位置 | 用途 |
|--------|--------|------|------|
| svc-gateway | 全部后端 | `src/proxy.ts` | 所有 API 转发 |
| svc-gateway | 各服务 `/metrics` | `services/svc-gateway/src/routes/health.ts:118` | 聚合指标 |
| svc-actions | svc-objects | `services/svc-actions/src/objects-client.ts:61` | 动作副作用写对象 |
| svc-datasets | svc-ontology | `services/svc-datasets/src/routes/v2/pipelines.ts:44` | 读取本体与对象类型做 pipeline 输出 |
| svc-aip | svc-ontology | `services/svc-aip/src/llm/real-client.ts:29`、`services/svc-aip/src/llm/mock-client.ts:37` | 让 LLM 理解本体 |
| svc-aip | 外部 LLM | `src/llm/{openai,anthropic,ollama}-client.ts` | 真实推理 |
| svc-admin | 多服务 | `services/svc-admin/src/routes/v2/stats.ts:12-77` | 平台统计（并发抓取，3s 超时） |

可以看到：**没有服务注册中心、没有重试/熔断库**，调用地址全部来自环境变量默认值（`*_SERVICE_URL`）。一个服务挂掉时，调用方要么 502，要么在统计页显示部分数据。

### 3.9 横切约定

| 约定 | 实现 | 说明 |
|------|------|------|
| 错误信封 | `packages/errors/src/api-error.ts` | 所有错误统一为 `{errorCode, errorName, errorInstanceId, parameters, statusCode}`；`parameters` 只含 SafeArg，敏感值在 `toJSON()` 中被剔除 |
| 资源标识 | `packages/rid` | `ri.<service>.<instance>.<type>.<id>`，服务端只接受字符串、不做结构校验的地方也有 |
| 分页 | `packages/pagination` | `pageSize` + `pageToken`/`nextPageToken`；v1 部分端点不分页 |
| 请求追踪 | 网关与各服务 | 网关生成/沿用 `x-request-id`，透传到下游并在响应头回写；日志字段含 `requestId` |
| 日志 | pino | 结构化 JSON；开发态 `pino-pretty`；`no-console` 规则只对三个"以打印为职责"的模块豁免 |
| 健康与指标 | `packages/health` | `/status/health`、`/metrics`；网关聚合下游指标 |
| 安全头 | `@fastify/helmet` | 网关统一加 HSTS；CSP 关闭，因为唯一 HTML 页面（multipass 登录页）自带更严格策略 |
| CORS | `@fastify/cors` | 网关 `origin: true` 反射来源；各服务也各自开了 CORS（便于直连调试） |

错误信封示例（真实形状）：

```json
{
  "errorCode": "NOT_FOUND",
  "errorName": "ObjectNotFound",
  "errorInstanceId": "3f1c9d2e-...",
  "parameters": { "objectType": "PestControlOrder", "primaryKey": "ORD-001" },
  "statusCode": 404
}
```

### 3.10 架构级设计决策

| # | 决策 | 理由（代码/文档证据） | 代价 |
|---|------|----------------------|------|
| 1 | 网关只做接入，不做业务 | 鉴权与限流集中，业务错误由服务产生，避免网关成为第二个领域模型 | 路径嗅探把"路由知识"塞进网关，新增端点容易漏配 |
| 2 | 服务按 Foundry 产品域切分，而非按 DDD 聚合 | 目标是与 Foundry API 对齐，端点归属天然按产品域划分 | 跨域写（动作→对象）必须走 HTTP，无事务 |
| 3 | 内存是一等公民，Postgres 是可选后端 | 一条命令起全栈、测试无需外部依赖 | 两套 store 语义不完全一致（同步/异步、隔离性），部分路由只在一种模式下正确 |
| 4 | 存储按服务分文件，权限 grants 例外共享 | 简单、可读、便于调试 | "部分持久化"造成重启后状态不一致；共享 JSON 文件没有并发控制 |
| 5 | 契约用 Conjure 自描述并生成类型 | 服务与 SDK 共享同一份模型，v1/v2 差异可被显式表达 | 自研编译器/生成器增加维护成本，生成物与 YAML 可能漂移 |
| 6 | 身份用自签 JWT + `x-user-*` 可信跳 | 服务无需每次回查 multipass，横向扩展友好 | 信任边界依赖"服务端口不可直达"，代码没有强制 |

### 3.11 小结

OpenFoundry 的架构可以一句话概括：**一个带鉴权与限流的薄网关，加 13 个同构的 Fastify 服务，加 33 个共享库，数据落在"内存/JSON 或 Postgres"两种可切换的后端上**。

它的复杂度不在进程数量，而在三处"隐式契约"：

1. 网关的路径嗅探规则（第 3.4 节）——路由归属靠正则，不靠注册表。
2. `x-user-*` 可信跳（第 3.6 节）——身份正确性依赖网络边界。
3. 存储双模的语义差异（第 3.7 节）——同一份路由在两种模式下行为不同。

后续章节会逐个展开这三处，并在第 13 章汇总为可执行的改进清单。

---

## 第 4 章 契约层与 SDK

### 4.1 本章导读

契约层回答两个问题：服务对外的请求与响应长什么样，客户端如何把一次方法调用变成一条 HTTP 请求。OpenFoundry 的答案分成三层：`conjure/*.yml` 用 Palantir Conjure 的语法声明 7 个服务、80 个端点；`packages/conjure-ir`、`conjure-compiler`、`conjure-codegen` 组成一条 YAML → IR → TypeScript 的生成流水线；`packages/client` 与 `packages/sdk*` 提供真正被应用调用的客户端。本章只覆盖「声明与客户端」；端点内部如何读写存储见〈本体与对象数据面〉与〈平台业务服务〉，令牌与权限见〈身份、权限与多租户安全〉。需要先说明一个贯穿全章的事实：契约 YAML 与生成流水线在本仓库里并没有接入构建，随包发布的类型和端点描述符是手写的，这既是本章最重要的设计取舍，也是绝大多数已知漂移的来源。

### 4.2 契约层的位置与边界

契约层不在请求路径上。网关把 `/api/v1`、`/api/v2` 的请求按 URL 正则转发给各服务，服务里的路由是手写注册的 Fastify 插件，既不读取 YAML，也不读取 IR（`services/svc-gateway/src/routes/v2/index.ts:43` 起是纯粹的前缀注册）。因此契约在本仓库中的角色是**声明与工具链**，而不是运行时的路由表。真正随代码发布的「契约」是 `packages/api-types` 里的手写类型、`packages/client` 里的端点元组，以及 `packages/sdk/sdk/src/types.ts` 里的手写响应类型。

| 形态 | 代表文件 | 参与运行时 | 仓库内消费者 |
|------|----------|------------|--------------|
| 声明 | `conjure/*.yml` | 否 | 集成测试、`sdk-cli generate types` |
| IR 与编译器 | `packages/conjure-ir`、`packages/conjure-compiler` | 否 | `conjure-codegen`、`sdk-cli` |
| 代码生成器 | `packages/conjure-codegen` | 否 | 仅 `sdk-cli`（见下文） |
| 手写 API 类型 | `packages/api-types` | 否 | 无：没有任何包 import 它 |
| HTTP 客户端 | `packages/client` | 是 | `@openfoundry/sdk`、`@openfoundry/sdk-oauth` |
| SDK 门面 | `packages/sdk/sdk` | 是 | 无仓库内消费者 |
| React Hooks | `packages/sdk/sdk-react` | 是 | 无仓库内消费者 |
| OAuth 助手 | `packages/sdk/sdk-oauth` | 是 | `apps/app-console` 的 `AuthContext` |
| CLI | `packages/sdk/sdk-cli` | 是 | `bin: openfoundry` |

「无消费者」不是猜测：`grep '"@openfoundry/sdk"' --include=package.json` 只命中它自己，`@openfoundry/api-types` 与 `@openfoundry/wire` 同样没有 workspace 依赖者。控制台覆盖约 80 个 `fetch` 调用点的方式是一个拦截器 `apps/app-console/src/lib/authFetch.ts`，而不是 SDK（`AGENTS.md:107`）；控制台唯一使用的 SDK 包是 `@openfoundry/sdk-oauth`（`apps/app-console/src/context/AuthContext.tsx:14`）。

### 4.3 代码结构

| 路径 | 作用 | 关键入口 |
|------|------|----------|
| `conjure/` | 8 个 Conjure YAML 契约 | 7 个服务定义 + 1 个共享类型文件 |
| `packages/conjure-ir/src/ir-types.ts` | IR 的 TypeScript 模型 | `ConjureDefinition`、`Type`、`EndpointDefinition` |
| `packages/conjure-compiler/src/compiler.ts` | YAML → IR | `compileConjureFile`、`compileConjureFiles` |
| `packages/conjure-compiler/src/file-loader.ts` | 目录加载与合并 | `loadConjureDirectory` |
| `packages/conjure-ir/src/ir-validator.ts` | IR 结构校验 | `validateIr` |
| `packages/conjure-codegen/src/codegen.ts` | IR → 文件集合 | `generateFromIr`、`writeGeneratedFiles` |
| `packages/conjure-codegen/src/service-generator.ts` | 每个服务生成一个客户端类 | `generateServiceFile` |
| `packages/api-types/src/` | 手写 API 类型与端点元组 | `index.ts`、`endpoints.ts` |
| `packages/wire/src/` | Conjure 线上格式的序列化约定 | `conjure-wire.ts`、`query-params.ts` |
| `packages/client/src/` | 分层 fetch 客户端 | `createClient`、`foundryPlatformFetch` |
| `packages/sdk/sdk/src/` | 门面与 5 个资源 API | `createFoundryClient` |
| `packages/sdk/sdk-cli/src/commands/generate.ts` | 唯一真正调用生成器的入口 | `openfoundry generate types` |

### 4.4 Conjure YAML 契约

#### 4.4.1 文件结构

8 个文件共用同一套骨架：顶层 `types:` 下是 `definitions:`，`default-package` 决定类型归属的 Java 包名，`objects:` 是一张「类型名 → 类型定义」的表；顶层 `services:` 下是「服务名 → 服务定义」。下面是从 `conjure/object-service.yml` 精简出的片段，逐行说明见其后。

```yaml
types:
  definitions:
    default-package: com.openfoundry.api.objects
    objects:
      OntologyObject:            # 有 fields → 对象类型
        fields:
          rid: Rid               # 引用同包内 alias
          objectType: string     # 内建原语
          properties: PropertyMap
services:
  ObjectService:
    base-path: /api/v2           # 编译时拼到每个端点路径前
    endpoints:
      listObjects:
        http: GET /ontologies/{ontologyRid}/objects/{objectType}
```

- `default-package`：类型与服务的命名空间，编译后写入 `TypeName.package`（`packages/conjure-compiler/src/compiler.ts:97`）。多个文件可以定义同名类型（如 `Rid`），靠包名区分。
- 类型种类由键名判定，顺序是 `alias` → `values` → `union` → `fields`（`packages/conjure-compiler/src/compiler.ts:130`）。`alias: string` 是别名，`values:` 是枚举，`fields:` 是对象；`union` 分支虽已实现（`packages/conjure-codegen/src/type-generator.ts:186`），但 8 个 YAML 里没有使用。
- 字段值可以写成裸类型字符串，也可以写成 `{type, docs}`；`optional<T>`、`list<T>`、`set<T>`、`map<K,V>` 递归解析（`packages/conjure-compiler/src/type-resolver.ts:28`）。小写名字才会命中原语表，首字母大写的 `Rid` 一律当作用户定义引用（`packages/conjure-compiler/src/type-resolver.ts:73`）。
- `http: METHOD /path` 由 `parseHttpString` 拆成方法与路径（`packages/conjure-compiler/src/http-parser.ts:14`），随后与 `base-path` 直接字符串拼接（`packages/conjure-compiler/src/compiler.ts:205`）。路径里的 `{param}` 用正则提取，供参数类型推断。
- `args` 的 `param-type` 是 `path`、`query`、`body`、`header` 之一；缺省时，名字出现在路径模板里就是 path，否则是 query（`packages/conjure-compiler/src/compiler.ts:275`）。
- `returns` 缺失表示无响应体；`docs` 会被 trim 后写入 IR。
- 三个 YAML 语法在编译器里被**静默丢弃**：`types.imports`（`packages/conjure-compiler/src/yaml-parser.ts:35` 有类型、`packages/conjure-compiler/src/compiler.ts:95` 起从不读取）、端点级 `auth:`（`RawEndpoint` 根本没有该字段，`packages/conjure-compiler/src/yaml-parser.ts:46`；端点 auth 一律取服务级 `default-auth`，`packages/conjure-compiler/src/compiler.ts:234`）、顶层 `errors:`（8 个文件都没有，编译器固定输出空数组，`packages/conjure-compiler/src/compiler.ts:114`）。IR 里定义了完整的 `ErrorDefinition`（`packages/conjure-ir/src/ir-types.ts:146`）和错误代码生成器（`packages/conjure-codegen/src/error-generator.ts:12`），但在这份契约上是死代码。

#### 4.4.2 8 个文件与服务映射

| 文件 | 服务名 | base-path | 端点数 | 目标服务（端口） |
|------|--------|-----------|--------|------------------|
| `action-service.yml` | ActionService | `/api/v2` | 5 | svc-actions（8083） |
| `admin-service.yml` | AdminService | `/api/v2` | 14 | svc-admin（8087） |
| `auth-service.yml` | AuthService | `/` | 5 | svc-multipass（8084） |
| `compass-service.yml` | CompassService | `/api/v2` | 8 | svc-compass（8086） |
| `dataset-service.yml` | DatasetService | `/api/v2` | 15 | svc-datasets（8085） |
| `object-service.yml` | ObjectService | `/api/v2` | 10 | svc-objects（8082） |
| `ontology-service.yml` | OntologyService | `/api/v2` | 23 | svc-ontology（8081） |
| `common-types.yml` | 无服务 | — | 0 | 仅类型；被其余文件声明 import，但编译器不解析 |

合计 7 个服务、80 个端点。端点数是按每个文件里 `http:` 行数统计的，与 `grep -c` 一致。`auth-service.yml` 的 base-path 是 `/`，因为它声明的就是 `/multipass/api/oauth2/token`、`/.well-known/openid-configuration` 这类完整路径（`conjure/auth-service.yml:100`）。

这张表里已经有一处漂移：`compass-service.yml` 声明 `GET /resources` 且 base-path 为 `/api/v2`（`conjure/compass-service.yml:65`），编译结果是 `/api/v2/resources`；而网关把 Compass 挂在 `/api/v2/compass`（`services/svc-gateway/src/routes/v2/index.ts:47`），服务端实际路由是 `/api/v2/compass/resources`（`services/svc-compass/src/routes/v2/resources.ts:11`）。契约里的 8 个 Compass 端点没有一个能与真实路由对上。`conjure/` 也没有 v1 的任何声明，v1 的 28 个操作只存在于代码与 README。

#### 4.4.3 IR 模型

编译器不直接产出 TypeScript，而是先产出一个与语言无关的 `ConjureDefinition`。它的形状是 Palantir Conjure IR 的 TypeScript 投影（`packages/conjure-ir/src/ir-types.ts:10`）：

```ts
// packages/conjure-ir/src/ir-types.ts 精简
export interface ConjureDefinition {
  version: number;
  errors: ErrorDefinition[];
  types: TypeDefinition[];      // alias | enum | object | union
  services: ServiceDefinition[];
  extensions: Record<string, unknown>;
}

export type Type =
  | { type: "primitive"; primitive: PrimitiveType }
  | { type: "optional"; optional: { itemType: Type } }
  | { type: "list"; list: { itemType: Type } }
  | { type: "map"; map: { keyType: Type; valueType: Type } }
  | { type: "reference"; reference: TypeName };
```

（`Type` 还有 `set` 与 `external` 两个分支，此处省略。）四个设计点值得注意：`Type` 是判别联合而不是字符串，生成器因此可以用穷尽 `switch` 处理每一种形状（`packages/conjure-codegen/src/type-generator.ts:16`）；`ParamType` 把 path/query/body/header 编码成带参数的联合，header 参数还能带 `paramId`（`packages/conjure-ir/src/ir-types.ts:133`）；`EndpointDefinition` 带 `tags` 与 `markers`，但编译器一律填空数组（`packages/conjure-compiler/src/compiler.ts:236`），因为 YAML 解析器不读这两个键；`ErrorDefinition` 在模型里完整存在，编译器却从不构造它（`packages/conjure-compiler/src/compiler.ts:114`）。`ir-builders.ts` 提供程序化构造 IR 的 helper，主要被 `conjure-ir` 自己的测试使用；`ir-utils.ts` 提供按 `TypeName` 查类型、取全部端点等只读工具。

### 4.5 代码生成流水线

```mermaid
flowchart LR
  Y["conjure/*.yml"] --> C["conjure-compiler"]
  C --> IR["conjure-ir ConjureDefinition"]
  IR --> G["conjure-codegen"]
  G --> F["GeneratedFile 数组 内存"]
  F -.->|"sdk-cli generate types 写盘"| O["任意 outputDir"]
  F -.->|"未接入"| A["packages/api-types"]
  Y -.->|"人工镜像"| A
  Y -.->|"人工镜像"| S["@openfoundry/sdk 类型"]
  A -.->|"无依赖"| S
  S --> CL["@openfoundry/client"]
  CL --> H["HTTP /api/v1 或 /api/v2"]
  SV["svc-* Fastify 路由"] --> H
```

图中实线是代码真实存在的调用关系，虚线是「本应存在但没有接上」的关系。流水线本身是完整的：`compileConjureFiles` 把多个 YAML 编译成各自的 `ConjureDefinition` 再合并（`packages/conjure-compiler/src/compiler.ts:34`），合并时按 `package.name` 查重、重名抛错（`packages/conjure-compiler/src/compiler.ts:56`）；`generateFromIr` 为每个类型、每个错误、每个服务各生成一个文件，最后追加一个 barrel `index.ts`（`packages/conjure-codegen/src/codegen.ts:35`）；`writeGeneratedFiles` 递归建目录后逐个写盘（`packages/conjure-codegen/src/codegen.ts:75`）。

一次生成分四步：

1. **加载**：`loadConjureDirectory` 只收 `.yml`/`.yaml`，文件名 `sort()` 后依次读取（`packages/conjure-compiler/src/file-loader.ts:12`）。排序是确定性的来源，但也意味着文件名字典序决定同名类型的覆盖顺序。
2. **编译**：每个文件独立编译成 IR，`extensions.source` 记录来源文件名（`packages/conjure-compiler/src/compiler.ts:117`）；`mergeDefinitions` 拼接 `types`/`services`/`errors` 并检查跨文件重名。
3. **校验**：`validateIr` 检查类型引用可解析、路径参数与 path 参数一一对应（`packages/conjure-ir/src/ir-validator.ts:28`）。它是纯函数，不会自动被编译或生成流程调用。
4. **生成**：`generateFromIr` 输出 `GeneratedFile[]`；是否写盘、写到哪里由调用方决定。

**生成物不入库**。`.gitignore` 明确忽略 `packages/api-types/src/generated/` 与 `conjure/ir/*.json`（`.gitignore:37`），仓库里也确实不存在 `src/generated` 目录。根 `package.json` 提供了 `generate:api` 脚本，内容是 `turbo run generate --filter=@openfoundry/api-types`（`package.json:19`），`turbo.json` 也声明了 `generate` 任务、输出 `src/generated/**`（`turbo.json:25`）；但静态检查显示没有任何 `package.json` 声明 `generate` 脚本，`@openfoundry/api-types` 的 scripts 只有 `build`/`test`/`typecheck`（`packages/api-types/package.json:18`）。因此这条命令在当前仓库不会执行任何生成任务（未实际运行 turbo，以免写入 `.turbo` 缓存；结论来自脚本与包清单的静态比对）。真正会调用生成器的是 `openfoundry generate types` 子命令：它动态 import 编译器与生成器，把 `conjureDir` 编译后写到 `outputDir`（`packages/sdk/sdk-cli/src/commands/generate.ts:52`），默认输出 `src/generated`（`packages/sdk/sdk-cli/src/config/project-config.ts` 的 `DEFAULTS`）。另一个调用方是集成测试 `tests/integration/conjure-pipeline.test.ts`，它通过相对路径直接 import 源码，验证 YAML → IR → TS 全链路，并断言 query 参数清单（`tests/integration/conjure-pipeline.test.ts:4`、`:92`）。

**确定性与增量**：确定性有，增量没有。输入排序固定、生成函数无随机源、无时间戳，同样的 YAML 必然产出同样的字节；但 `writeGeneratedFiles` 是全量覆盖，没有哈希比对、没有「仅写变化文件」，也不检查输出是否已存在。`generate` 任务在 turbo 层面可以靠 `outputs` 做缓存，前提是某个包真的声明了该脚本——当前没有。

**平坦命名空间的冲突**：代码生成把 Conjure 的包结构压平成一个目录，文件名只取类型名的 kebab-case（`packages/conjure-codegen/src/naming.ts:37`），导入也按名字而非包名（`packages/conjure-codegen/src/service-generator.ts:29`）。我在内存里对 8 个 YAML 跑了完整合并与生成，得到 112 个类型、7 个服务、120 个文件、105 个唯一路径：`rid.ts` 出现 7 次、`page-token.ts` 7 次，`property-value.ts`、`property-map.ts`、`order-by-clause.ts` 各 2 次。`writeGeneratedFiles` 会按顺序写，后写的覆盖先写的；barrel 生成器对每个条目都输出一行 `export *`（`packages/conjure-codegen/src/index-generator.ts:14`），于是 `index.ts` 里出现 7 行相同的 `export * from "./rid"`。这是流水线能跑通、但产物不能直接当作一个 TypeScript 包使用的直接原因。

### 4.6 生成的类型长什么样

下面是 `generateFromIr` 对合并后 IR 的真实输出（本次运行于内存，未写盘、未入库）：

```ts
// Rid 来自 common-types.yml，OntologyObject 来自 object-service.yml
export type Rid = string;

export interface OntologyObject {
  rid: Rid;
  objectType: string;
  primaryKey: string;
  properties: PropertyMap;
}
```

服务文件生成一个 `ConjureClient` 接口和一个类，方法体只做路径插值与参数打包（`packages/conjure-codegen/src/service-generator.ts:122`）。实际输出是一整行，下面按参数换行以便阅读：

```ts
// 实际输出的排版是单行，这里仅为可读性换行
async listObjects(ontologyRid: Rid, objectType: string, options?: {
  pageSize?: number | undefined; pageToken?: PageToken | undefined;
  select?: Array<string> | undefined; excludeRid?: boolean | undefined;
}): Promise<ListObjectsResponse> {
  return this.client.fetch<ListObjectsResponse>("GET",
    `/api/v2/ontologies/${ontologyRid}/objects/${objectType}`,
    { query: { pageSize: options?.pageSize, /* … */ } });
}
```

注意生成的方法把 `base-path` 拼进了路径，得到 `/api/v2/...`；而手写 SDK 的端点元组用 `/v2/...`，由 `foundryPlatformFetch` 再补 `api` 前缀（`packages/client/src/platform-fetch.ts:24`）。两套约定都能到达同一个 URL，但不能混用。

真正入库、被当作「API 类型」引用的是 `packages/api-types`，它是手写的：文件头写着 "Hand-written TypeScript types for all OpenFoundry API endpoints"（`packages/api-types/src/index.ts:4`），每个文件注明「Derived from conjure/xxx.yml」。它的 `OntologyObject` 与生成结果形状一致，但没有任何机制保证这一点：

```ts
// packages/api-types/src/objects.ts:10-15，手写，非生成
export interface OntologyObject {
  rid: Rid;
  objectType: string;
  primaryKey: string;
  properties: PropertyMap;
}
```

`api-types` 还手写了 79 个端点元组，全部是 `/api/v2`、没有 v1（`packages/api-types/src/endpoints.ts:19` 起）。它自己的元组类型是 `[method, path, hasBody, needsAuth, needsPreview]`（`packages/api-types/src/endpoints.ts:7`），而 `@openfoundry/client` 的同名类型是 `[method, path, flags, requestContentType, responseContentType]`（`packages/client/src/types.ts:77`）。两者同名不同义，且 `api-types` 的路径带 `/api` 前缀，直接喂给 `foundryPlatformFetch` 会得到 `/api/api/v2/...`，因此它们不能组合使用；仓库里也确实没有这样的调用点。

### 4.7 wire 包的线上约定

`@openfoundry/wire` 把 Conjure 的 JSON 线上规则独立成包：`ConjureType` 枚举列出 15 种类型标签（`packages/wire/src/conjure-wire.ts:14`），`serializeConjureValue` / `deserializeConjureValue` 按标签递归处理 optional、list、set、map、object（`packages/wire/src/conjure-wire.ts:47`）。几条具体规则：

- DATETIME 在运行时是 `Date`，线上是 ISO 8601 字符串；`toWireDatetime` 固定用 `toISOString()`，`fromWireDatetime` 先用正则校验再解析（`packages/wire/src/datetime.ts:15`）。
- BEARERTOKEN 序列化时替换成 `"{REDACTED}"`，避免令牌进入日志或请求体（`packages/wire/src/conjure-wire.ts:73`）。
- BINARY 在 `Uint8Array` 与 base64 字符串之间转换，优先用 `Buffer`，没有则回退到手写编码（`packages/wire/src/binary.ts:13`）。
- SAFELONG 用 `number` 承载，非整数或超出 ±(2^53−1) 时抛 `RangeError`（`packages/wire/src/safe-long.ts:20`）。
- 请求体用 `JSON.stringify` 的 replacer 把 `Date` 转 ISO，响应体用 `JSON.parse` 的 reviver 把 ISO 字符串还原成 `Date`（`packages/wire/src/request-body.ts:15`）。

这套规则与真实 Conjure 客户端一致，但当前没有消费者：`@openfoundry/client` 只做普通 `JSON.stringify`（`packages/client/src/platform-fetch.ts:43`），SDK 的日期字段按 `string` 声明（`packages/sdk/sdk/src/types.ts:215` 的 `createdTime: string`）。换句话说，wire 包是「为将来准备的规范实现」，不是现有链路的一环。

### 4.8 SDK 分层

| 包 | 职责 | 依赖 | 仓库内消费者 |
|----|------|------|--------------|
| `@openfoundry/client` | 四层 fetch 中间件、URL 构造、错误映射 | errors、auth-tokens | sdk、sdk-oauth（类型） |
| `@openfoundry/sdk` | 门面 `FoundryClient` + 5 个资源 API | client、object-set、pagination、ontology-schema、rid、errors、auth-tokens | 无 |
| `@openfoundry/sdk-oauth` | 授权码 + PKCE、令牌刷新 | client（类型）、auth-tokens | app-console |
| `@openfoundry/sdk-react` | Provider 与 11 个 hooks | 仅 react | 无 |
| `@openfoundry/sdk-cli` | `openfoundry` 命令行 | conjure-compiler、conjure-codegen、conjure-ir、config | 无（作为 bin 使用） |

依赖方向是单向的：`sdk → client`，`sdk-oauth → client`（只 import `TokenProvider` 类型，`packages/sdk/sdk-oauth/src/token-manager.ts:1`），`sdk-react` 谁也不依赖、自己用原生 `fetch`，`sdk-cli` 只依赖 Conjure 三件套与 `config`。没有服务依赖 SDK，也没有 SDK 依赖服务实现；这是分层最干净的一处。

一次 `client.objects.listObjects(...)` 到 HTTP 请求的路径：

1. `FoundryClient` 的 getter 懒创建并缓存命名空间（`packages/sdk/sdk/src/foundry-client.ts:82`）。
2. `createObjectsApi(client)` 返回对象字面量，方法内先 `withPathParams` 把 `/v2/.../:param` 换成真实路径（`packages/sdk/sdk/src/api/objects.ts:76`）。
3. 调 `foundryPlatformFetch`：`new URL("api" + path, baseUrl + "/")` 得到 `{baseUrl}/api/v2/...`，再追加 query（`packages/client/src/platform-fetch.ts:24`）。
4. 进入 `client.fetch`，即第 1 层 `headerMutator`：`await tokenProvider()` 后写入 `Authorization: Bearer ...`（`packages/client/src/header-mutator.ts:13`）。
5. 第 2 层 `retryingFetch` 本应检查状态码后决定是否重试，但它调用的是第 3 层 `fetchOrThrow`，非 2xx 会先被抛成 `OpenFoundryApiError`（见下文）；第 4 层是原生 `fetch`（`packages/client/src/client.ts:24`）。
6. 响应按端点元组的 `responseContentType` 解析：JSON 直接 `.json()`，`204/205` 返回 `undefined`，`application/octet-stream` 原样返回 `Response`（`packages/client/src/platform-fetch.ts:66`）。

错误映射只有一条规则：非 2xx 一律转成 `OpenFoundryApiError`。`fetchOrThrow` 先尝试把响应体解析成 `{errorCode, errorName, errorInstanceId, parameters}` 信封；解析失败就只用状态码构造错误（`packages/client/src/fetch-or-throw.ts:10`）。`fromResponse` 在信封里的 `errorCode` 不是已知枚举值时按状态码兜底映射，`errorName` 缺失则记为 `UnknownError`（`packages/errors/src/api-error.ts:109`）。因此调用方拿到的错误同时有机器可读的 `errorCode` 与用于追踪的 `errorInstanceId`，但 `message` 是拼接出来的 `API error <status>: <name>`，服务端的人类可读消息不会保留。

`createClient` 的文档注释声称顺序是「header → retrying → fetch-or-throw → raw」（`packages/client/src/client.ts:7`），但实现里 `throwingFetch` 先包住 raw，`retryingFetch` 再包住 `throwingFetch`（`packages/client/src/client.ts:28`）。这导致 429/503 在到达重试层之前就被第 3 层抛成异常，重试永远不会发生。我用真实的 `createClient` 配一个计数 fetch 实测：`maxRetries: 3` 下 429 响应只触发了 1 次底层调用，随后抛出 `OpenFoundryApiError`。`packages/client/tests/client.test.ts:231` 那条「retries and then throws on persistent 429」只断言最终抛错，没有断言调用次数，所以测试是绿的。重试层自身是好的（单独使用 `createRetryingFetch` 的测试覆盖了 429/503、指数退避与 `Retry-After`，`packages/client/src/retrying-fetch.ts:65`），坏的是组合顺序。

#### 4.8.1 认证与令牌

`@openfoundry/client` 只认 `TokenProvider`：一个返回 `BearerToken` 的同步或异步函数（`packages/client/src/types.ts:13`）。`BearerToken` 封装原始字符串，提供常量时间比较、`toAuthorizationHeader()`，并让 `toString`/`toJSON`/`inspect` 全部返回 `[REDACTED BearerToken]`，防止令牌进入日志（`packages/auth-tokens/src/bearer-token.ts:11`）。静态令牌与 provider 二选一的包装在 SDK 门面里完成（`packages/sdk/sdk/src/foundry-client.ts:123`）。

生产模式的登录链路由 `@openfoundry/sdk-oauth` 提供，控制台的 `AuthContext` 是唯一消费者（`apps/app-console/src/context/AuthContext.tsx:14`）：

1. `createAuthorizationUrl` 生成 `/multipass/api/oauth2/authorize?...`，默认开 PKCE，`code_challenge_method=S256`（`packages/sdk/sdk-oauth/src/oauth-client.ts:88`）。
2. `generateCodeVerifier` 从随机字节取 RFC 7636 允许的字符集；`generateCodeChallenge` 做 SHA-256 后 base64url 无填充编码（`packages/sdk/sdk-oauth/src/pkce.ts:22`、`:38`）。
3. 回调拿到 code 后 `exchangeCodeForToken` 换令牌；`TokenManager.setToken` 记录绝对过期时间（`packages/sdk/sdk-oauth/src/token-manager.ts:107`）。
4. `getToken()` 在过期前 60 秒触发刷新，`#refreshPromise` 去重并发刷新（`packages/sdk/sdk-oauth/src/token-manager.ts:133`、`:137`）。
5. `toTokenProvider()` 把 `TokenManager` 适配成 client 需要的 `TokenProvider`（`packages/sdk/sdk-oauth/src/token-manager.ts:70`）。

PKCE 与服务端的约束是配套的：svc-multipass 的授权页只接受注册过的客户端、重定向 URI 精确匹配、公共客户端必须用 S256（`conjure/auth-service.yml:130` 的 docs 与 `services/svc-multipass/src/routes/authorize.ts` 的实现一致）。`sdk-cli` 的登录流程复用了 PKCE，但端点路径写错（见已知限制第 9 条）。

**超时**：客户端没有内建超时。`ClientOptions` 只有 `baseUrl`、`tokenProvider`、`fetch`、`maxRetries`、`retryBaseDelayMs`（`packages/client/src/types.ts:18`），单次请求的 `RequestOptions` 只接受调用方传入的 `AbortSignal`（`packages/client/src/types.ts:38`）。想设超时只能在 `tokenProvider` 之外自己包一层 fetch，或用 `AbortSignal.timeout()` 显式传入。

### 4.9 v1 与 v2 双版本

`/api/v1` 与 `/api/v2` 不是同一 API 的两个前缀，而是两套模型。README 的原话是「a separate API, not the same one behind another prefix」（`README.md:231`），AGENTS.md 进一步列出四处必分歧字段（`AGENTS.md:86`）。代码里的分歧比文档列的更多：

| 维度 | v1 | v2 | 证据 |
|------|----|----|------|
| 路由前缀 | `/api/v1` | `/api/v2` | 各服务 `server.ts` 两次 `app.register` |
| `Branch` 标识 | `branchId` | `name` | `services/svc-datasets/src/routes/v1/serializers.ts:51` |
| `OntologyObject` | `{properties, rid}` 信封 | store 的完整对象，含 `rid/objectType/primaryKey/properties/createdAt/updatedAt` | `services/svc-objects/src/routes/v1/serializers.ts:25`；`services/svc-objects/src/routes/v2/objects.ts:366` |
| `ObjectType.primaryKey` | `string[]` | `primaryKeyApiName: string` | `services/svc-ontology/src/routes/v1/serializers.ts:42` |
| `ApplyActionResponse` | `{}`，无字段 | `{rid, status, result}` | `services/svc-actions/src/routes/v1/actions.ts:102`；`services/svc-actions/src/routes/v2/actions.ts:113` |
| `File` | `path/transactionRid/sizeBytes/updatedTime` | 同左，但仍单独投影 | `services/svc-datasets/src/routes/v1/serializers.ts:100`；`services/svc-datasets/src/routes/v2/files.ts:31` |
| 列表投影参数 | `properties` | `select` | `services/svc-objects/src/routes/v1/objects.ts:191` |
| 搜索过滤语法 | `query`，14 个变体，`contains` 测数组成员 | `where`，28 个变体，`contains` 测子串 | `services/svc-objects/src/routes/v1/search-query.ts:1` |
| 聚合结果 | `{name, value}` | 额外带 `type`、`field` | `services/svc-objects/src/routes/v1/objects.ts:393` |
| 分页 | `GET /ontologies` 不分页 | 分页 | `services/svc-ontology/src/routes/v1/ontologies.ts:55` |
| 创建状态码 | 200（创建路由未显式设置） | 201（创建路由显式设置） | `services/svc-datasets/src/routes/v1/datasets.ts:46`；`services/svc-datasets/src/routes/v2/datasets.ts:47` |

状态码差异同样属于契约：v1 的创建接口不设置 201，调用方读到的是 200。

#### 4.9.1 v1 的服务面

v1 由四个路由插件组成，合计 28 个操作：svc-ontology 8 个（ontologies、objectTypes、outgoingLinkTypes、actionTypes、queryTypes），svc-objects 5 个（列表、单对象、链接、search、aggregate），svc-actions 3 个（apply、applyBatch、validate），svc-datasets 12 个（dataset、branches、transactions、files）。注册点在各服务的 `server.ts`：`services/svc-ontology/src/server.ts:136`、`services/svc-objects/src/server.ts:157`、`services/svc-actions/src/server.ts:150`、`services/svc-datasets/src/server.ts:168`。网关侧不是简单透传：v1 的 ontology 前缀按 URL 三分到 svc-objects / svc-actions / svc-ontology（`services/svc-gateway/src/routes/v1/ontologies.ts:22`）。

#### 4.9.2 序列化投影策略

三个服务各自维护 `routes/v1/serializers.ts`：`svc-objects`（39 行）、`svc-ontology`（247 行）、`svc-datasets`（126 行）。`svc-actions` 没有单独的 serializers 文件，v1 形状直接内联在 `routes/v1/actions.ts` 里。投影的输入一律是 store 的存储类型，不是 v2 路由的输出。这一点在代码注释里写得很直白：

> v1 and v2 are not the same API with a different prefix. … These functions project the store's own shapes onto the v1 models declared in `foundry_sdk/v1/datasets/models.py`, so nothing here depends on what the v2 routes happen to emit.

（`services/svc-datasets/src/routes/v1/serializers.ts:4`）

投影不是字段改名，还要处理存储无法表达的模型约束。`toV1Dataset` 在 `parentFolderRid` 缺失时抛 500，而不是发一个空字符串或编造的根目录 RID（`services/svc-datasets/src/routes/v1/serializers.ts:55`）；`toV1ObjectType` 的 `primaryKey` 包成单元素列表，因为 v1 允许复合主键而 OpenFoundry 只支持单键（`services/svc-ontology/src/routes/v1/serializers.ts:162`）；`entityRid` 用确定性哈希给没有 RID 列的实体造一个稳定 RID（`services/svc-ontology/src/routes/v1/serializers.ts:88`）。v1 的 `File` 与 v2 四个字段完全相同，仍然单独写一份投影，理由是「the two versions agree today by coincidence, not by contract」（`services/svc-datasets/src/routes/v1/serializers.ts:104`）。

`svc-objects` 的 v1 搜索没有复用共享查询引擎，而是另写了一个 173 行的求值器：v1 `contains` 是「数组包含某值」，v2 的 `contains` 是「字符串包含子串」，把 v1 语法翻译成 v2 子句会答错（`services/svc-objects/src/routes/v1/search-query.ts:1`、`:136`）。v1 的 `orderBy` 也不同于 v2：v1 是 `{fields: [{field, direction}]}` 的列表，v2 是单个 `{field, direction}` 子句（`services/svc-objects/src/routes/v1/objects.ts:67`）。

#### 4.9.3 未服务操作如何 404

v1 声明了但 OpenFoundry 不服务的操作有两个：`POST /api/v1/datasets/{rid}/files:upload` 和整个 `attachments` 命名空间。两者的 404 机制不同：

- **upload** 走网关的 `/api/v1/datasets/*` 通配代理（`services/svc-gateway/src/routes/v1/index.ts:30`）到 svc-datasets；svc-datasets 的 v1 文件路由只注册了列表、读取、删除（`services/svc-datasets/src/routes/v1/files.ts:83` 起），没有注册 upload。Fastify 默认的 not-found 处理器返回 404，网关把上游状态码原样转发（`services/svc-gateway/src/proxy.ts:136`），所以调用方看到 404。为什么不实现写在注释里：v1 上传必须指定 `transactionRid` 或 `transactionType`，而文件存储一个路径只有一份记录，后续写入会静默覆盖，无法把文件放进某个事务（`services/svc-datasets/src/routes/v1/files.ts:74`）。
- **attachments** 在网关层就没有注册前缀。注释说明了原因：注册一个没有后端的前缀会得到 502，而调用方应当得到「这个操作不存在」的 404（`services/svc-gateway/src/routes/v1/index.ts:17`）。仓库里没有任何 `setNotFoundHandler`，所以 404 体是 Fastify 的默认形状。

另有一类「声明了但拒绝」而不是 404：v1 文件路由声明了 `branchId`、`startTransactionRid`、`endTransactionRid`、`transactionRid`，收到非空值一律 400 `invalidArgument`，因为文件记录没有分支/事务维度，静默忽略等于用唯一一份状态回答一个带范围的请求（`services/svc-datasets/src/routes/v1/files.ts:50`）。v1 还拒绝 `DELETE` 事务类型、`fuzzy` 词项匹配、`ranges`/`fixedWidth`/`duration` 分组（`services/svc-datasets/src/routes/v1/datasets.ts:20`、`services/svc-objects/src/routes/v1/search-query.ts:113`、`services/svc-objects/src/routes/v1/objects.ts:437`）。

### 4.10 设计决策与理由

**1. 自研 Conjure 编译器，而不是官方 `conjure-typescript`。** 理由：整条链是纯 TypeScript，不需要 JVM、Gradle 或 Conjure 的 Java 工具链，`pnpm install` 之后即可在 vitest 里直接 import 编译器与生成器做断言（`tests/integration/conjure-pipeline.test.ts:4`）。取舍：只实现仓库用到的子集——不解析 `imports`、不生成错误类型、丢弃端点级 `auth`、把包结构压平成单目录，因此产物不能当作通用 Conjure 实现使用，同名类型冲突要靠调用方自己避开。

**2. 生成物不入库，`api-types` 手写。** 理由：避免生成代码进入 review diff，避免「改一行 YAML 产生几十个文件变更」，包发布只需要 `tsc`（`packages/api-types/package.json:18`）。取舍：契约与类型之间没有任何自动校验，YAML 改了没人重新生成，`api-types` 与 YAML 靠人工同步；`generate:api` 与 turbo 的 `generate` 任务成了空配置，给读者「有生成流程」的错觉。

**3. v1 不共享 v2 序列化器。** 理由：v1 客户端不能被 v2 的字段增删影响。`File` 是反例中的正例——两版字段今天完全一致，仍然分开投影（`services/svc-datasets/src/routes/v1/serializers.ts:100`）。取舍：同一实体要维护两份投影，且两份都可能与 store 漂移；v1 的 `OntologyObject` 只保留 `properties` 与 `rid`，其余 store 字段被显式丢弃，这个「丢弃清单」也是手写的。

**4. SDK 不依赖服务实现，只依赖共享包。** 理由：SDK 可以被外部项目单独安装，服务可以替换实现；`@openfoundry/sdk` 的依赖只有 client、object-set、pagination、ontology-schema、rid、errors、auth-tokens（`packages/sdk/sdk/package.json:23`）。取舍：wire 类型只能手写，于是出现 SDK 按 Foundry 模型写、服务按自己的 store 模型发的情况（见下节）；SDK 也拿不到服务端的路由清单，覆盖面靠人工维护。

**5. 四层 fetch 中间件。** 理由：鉴权、重试、错误映射、传输各管一件事，每层都能单独测试（`packages/client/tests/client.test.ts` 分层覆盖）。取舍：组合顺序是隐式契约，错了不会被类型系统发现——当前 `retryingFetch` 包在 `throwingFetch` 外面，重试实际失效（`packages/client/src/client.ts:28`）。

**6. 契约只声明已实现的操作。** 理由：避免发布一个调用必然 404 的 API；`conjure/` 里没有 `attachments`，没有 v1，Compass 也只列了 8 个资源操作。取舍：YAML 不能当作完整 API 目录——`ontology-service.yml` 23 个端点，实际服务还提供 `outgoingLinkTypes`、`queryTypes`、`sharedPropertyTypes` 等未声明路由；v1 的 28 个操作完全不在契约里。

### 4.11 已知限制与坑

**1. v1/v2 的兼容成本落在每个服务上。** 同一个实体两套投影、两套路由、两套查询参数；v1 的必填字段在 store 里可能是可选的，只能在序列化时抛 500（`services/svc-datasets/src/routes/v1/serializers.ts:55`）。新增一个 v2 字段不会自动出现在 v1，反过来也不会；`svc-actions` 甚至没有 serializer 文件，v1 形状内联在路由里（`services/svc-actions/src/routes/v1/actions.ts:21`）。

**2. 生成物与 YAML 漂移没有任何检测。** 已知三处：Compass 的 base-path 拼不出真实路由（`conjure/compass-service.yml:65` vs `services/svc-gateway/src/routes/v2/index.ts:47`）；`dataset-service.yml` 的 `Branch` 声明了 `datasetRid`（`conjure/dataset-service.yml:38`），store 的 `Branch` 没有这个字段（`services/svc-datasets/src/store/dataset-store.ts:11`），v2 路由原样返回 store 对象；`ontology-service.yml` 的 `ObjectTypeV2` 用 `primaryKey`（`conjure/ontology-service.yml:39`），而类型化的 `ObjectTypeDefinition` 用 `primaryKeyApiName`（`packages/ontology-schema/src/object-type.ts:44`）。最后这处最麻烦：v2 创建路由把请求体原样存进 store（`services/svc-ontology/src/routes/v2/object-types.ts:66`、`services/svc-ontology/src/store/ontology-store.ts:188`），而 demo 种子是用 `"primaryKey"` 创建的（`scripts/demo/seed-healthcare.sh:36`），于是同一个 GET 端点可能返回 `primaryKey`，也可能返回 `primaryKeyApiName`；v1 serializer 必须同时读两种拼写（`services/svc-ontology/src/routes/v1/serializers.ts:116`），SDK 的 `ObjectType` 只认 `primaryKeyApiName`（`packages/sdk/sdk/src/types.ts:24`）。

**3. `generate` 任务与 `generate:api` 是空配置。** 没有任何包声明 `generate` 脚本，`turbo run generate` 匹配不到任务；`packages/api-types/src/generated/` 被 gitignore 且目录不存在（`.gitignore:37`）。唯一可用入口是 `openfoundry generate types`，默认写到使用者的 `src/generated`，不是 `api-types`。

**4. 代码生成器压平命名空间会静默覆盖。** 实测 120 个文件里 105 个唯一路径，`rid.ts` 与 `page-token.ts` 各重复 7 次，`order-by-clause.ts` 有两个不同内容（`common-types` 版带 `OrderDirection` 枚举，`object-service` 版是 `direction?: string`），写盘时后者覆盖前者，而 barrel 里会出现重复的 `export *`。生成器也没有 `errors` 输入（8 个 YAML 都没有 `errors:`），`error-generator.ts` 在这份契约上不会产出任何文件。

**5. 组合后的客户端不重试。** 见上文实测：`createClient({maxRetries: 3})` 对持续 429 只调用底层 fetch 一次。`packages/client/src/client.ts:7` 的注释、`apps/app-docs` 的「Retrying fetch」说明都与实际行为不符。修法是把 `createRetryingFetch` 放在 `createFetchOrThrow` 内层，但那会改变 4xx 是否重试的语义，属于真实的行为取舍，目前没人动。

**6. 没有内建超时。** 只有调用方传入的 `AbortSignal`（`packages/client/src/types.ts:38`）；`fetch` 本身不会超时，一个挂住的请求会一直占用调用栈。

**7. SDK 的响应类型与服务端 wire 形状不一致。** `@openfoundry/sdk` 的 `OntologyObject` 是 Foundry v2 的裸 map：`__apiName`、`__primaryKey`、`__rid` 加索引签名（`packages/sdk/sdk/src/types.ts:81`），而服务端 v2 返回的是嵌套 `properties` 的完整对象（`services/svc-objects/src/routes/v2/objects.ts:366`）。SDK 测试用裸 map 造响应（`packages/sdk/sdk/tests/sdk.test.ts:325`），所以测试通过；`tests/integration/sdk-compat.test.ts` 只断言 `data` 是数组，不检查元素形状，也发现不了。同类问题还有：`ActionResult` 声明 `{edits?, validation?}`（`packages/sdk/sdk/src/types.ts:95`）而 v2 返回 `{rid, status, result}`；`Branch` 声明 `branchId`（`packages/sdk/sdk/src/types.ts:203`，v1 模型）但 SDK 根本没有 branches API；`ListObjectsParams.properties`（`packages/sdk/sdk/src/types.ts:260`）发给 v2 路由后无人读取，v2 读的是 `select`（`services/svc-objects/src/routes/v2/objects.ts:271`）。

**8. SDK 覆盖面窄。** 只有 5 个命名空间（ontologies、objects、actions、datasets、admin），没有 branches/transactions/files/links/compass/search；datasets 只有 4 个方法（`packages/sdk/sdk/src/api/datasets.ts:97`）。`Branch`、`Transaction` 类型被导出却没有任何 API 使用（`packages/sdk/sdk/src/index.ts:42`）。

**9. `sdk-cli auth login` 走的路由不存在。** CLI 把授权地址拼成 `new URL("/oauth/authorize", baseUrl)`、令牌地址拼成 `/oauth/token`（`packages/sdk/sdk-cli/src/commands/auth.ts:40`），但 svc-multipass 只注册了 `/multipass/api/oauth2/authorize` 与 `/multipass/api/oauth2/token`（`services/svc-multipass/src/routes/authorize.ts:189`、`services/svc-multipass/src/routes/oauth.ts:72`），网关也没有 `/oauth` 前缀。CLI 用的 client id 是 `openfoundry-cli`（`packages/sdk/sdk-cli/src/commands/auth.ts:10`），而开发客户端清单里只有 `openfoundry-dev`、`openfoundry-console`、`admin`、`developer`（`services/svc-multipass/src/store/client-store.ts:48`）。这条路径没有测试覆盖（`packages/sdk/sdk-cli/tests/sdk-cli.test.ts` 只测参数解析、凭据存储与帮助文本），属于静态可确认的断链（未做运行时验证）。

**10. `sdk-react` 与 `sdk`/`client` 脱节。** 它的依赖只有 react，hooks 自己拼 `/api/v2/...` 并用原生 `fetch`（`packages/sdk/sdk-react/src/use-fetch.ts:70`、`packages/sdk/sdk-react/src/use-objects.ts:39`），既不经过 `client` 的重试与错误映射，也不经过 SDK 的资源 API。仓库里没有消费者。

**11. v2 的 `applyBatch` 经网关不可达。** 网关 v2 的 ontology 代理只用 `/(apply|validate)$` 匹配动作路由（`services/svc-gateway/src/routes/v2/ontologies.ts:35`），`/applyBatch` 不匹配，被转发给 svc-ontology，而那里没有该路由，最终 404。v1 的代理正则补上了 `applyBatch`，并在注释里点名这是 v2 的缺陷（`services/svc-gateway/src/routes/v1/ontologies.ts:10`）。

> 文档与实现不一致：AGENTS.md 说「v1 `OntologyObject` nests values under `properties` while v2 is a bare property map」（`AGENTS.md:86`），提交 9032c45 的说明也这么写，`services/svc-objects/tests/v1-api.test.ts:6` 的注释同样重复了这句。但 v2 路由从未把对象拍平成裸 map：它直接返回 store 的 `StoredObject`（`services/svc-objects/src/routes/v2/objects.ts:366`），测试断言的是 `body.properties.name`（`services/svc-objects/tests/objects-service.test.ts:79`），`GUIDE.md:626` 的运行示例也是「v2 - objek utuh dengan metadata」。真正的裸 map（`__apiName` 等）只存在于 SDK 的 `types.ts`，服务端没有实现。v1 与 v2 确实必须分开投影，但理由是 v1 只保留 `{properties, rid}`、v2 带元数据，而不是「v2 没有 properties」。

> 文档与实现不一致：`apps/app-docs` 的 Ontology 指南让用户执行 `pnpm generate:api` 并声称「The generated code is placed in packages/api-types/」（`apps/app-docs/src/pages/OntologyGuide.tsx:258`）。实际上该命令不执行生成任务，`api-types` 也不是生成目录。同一套文档的 SDK 指南称 `foundryPlatformFetch()` 会「Adds platform headers (trace ID, user agent)」（`apps/app-docs/src/pages/SdkGuide.tsx:89`），实现里没有这两个 header（`packages/client/src/platform-fetch.ts:16` 起只设 Content-Type/Accept）；指南里的 CRUD 示例请求 `/api/objects/v1/objects/...`（`apps/app-docs/src/pages/SdkGuide.tsx:101`），这个前缀在仓库里不存在；React 示例 import `FoundryProvider` from `@openfoundry/sdk/react`（`apps/app-docs/src/pages/SdkGuide.tsx:332`），实际包名是 `@openfoundry/sdk-react`、导出名是 `OpenFoundryProvider`（`packages/sdk/sdk-react/src/context.tsx:19`），且 `useObjects` 的签名是 `(ontologyRid, objectType, options)`（`packages/sdk/sdk-react/src/use-objects.ts:25`），不是示例里的 `useObjects("Employee", {...})`。

> 文档与实现不一致：`packages/wire` 的查询参数注释写「Array: comma-separated values」（`packages/wire/src/query-params.ts:10`），实现是对数组逐个重复 key 并用 `&` 连接（`packages/wire/src/query-params.ts:23`），与 `foundryPlatformFetch` 的数组处理一致（`packages/client/src/platform-fetch.ts:101`），但与注释相反。另外 `@openfoundry/wire` 当前没有任何 workspace 依赖者，Conjure 的 datetime/base64/safelong 规则由它定义却未被 client 使用——client 只做 `JSON.stringify`（`packages/client/src/platform-fetch.ts:43`）。

**12. `AGENTS.md:90` 的对照方法仍然有效，但需要外部仓库。** 代码注释大量引用 `foundry_sdk/v1/*/models.py`、`foundry_sdk/v2/*/models.py` 作为模型来源（例如 `services/svc-datasets/src/routes/v1/serializers.ts:8`），但 `foundry_sdk` 不在本仓库中；要复核 v1/v2 模型必须另取官方 Python SDK。本仓库内可用的替代证据是 `@osdk/foundry`（根 `package.json` 依赖 `^2.56.0`）与 `tests/integration/sdk-compat.test.ts`，但后者只验证端点可达，不验证响应元素形状（`tests/integration/sdk-compat.test.ts:244`），所以模型漂移不会被它拦住。

**13. `@openfoundry/api-types` 没有调用方，也不能驱动 `client`。** 它的 79 个端点元组全是 `/api/v2` 前缀，五元组是 `[method, path, hasBody, needsAuth, needsPreview]`（`packages/api-types/src/endpoints.ts:7`）。
而 `client` 的五元组是 `[method, path, flags, requestContentType, responseContentType]`（`packages/client/src/types.ts:77`），且 `foundryPlatformFetch` 会在路径前再拼一个 `api`（`packages/client/src/platform-fetch.ts:24`）。两者同名不同义，当前只能当类型字典使用；要接上必须先统一元组形状与路径前缀，仓库内没有任何 import 点（静态结论）。

**14. `conjure/` 没有 v1 声明，也没有 `attachments` 声明。** 想从契约判断某个 v1 操作是否存在，只能读 `services/*/src/routes/v1/index.ts` 与 README；契约与实现的完整性靠人工维持。

**15. 应用内架构文档把 `api-types` 画成 SDK 的依赖。** `apps/app-docs/src/pages/Architecture.tsx:212` 写 `sdk/ --> client, api-types, object-set, ontology-schema`，但 `packages/sdk/sdk/package.json` 的 dependencies 没有 `@openfoundry/api-types`；SDK 用的是自己手写的 `src/types.ts`。

### 4.12 关键代码位置

- `packages/conjure-compiler/src/compiler.ts:25` — `compileConjureFile`，YAML → IR 的入口。
- `packages/conjure-compiler/src/compiler.ts:91` — `compileRaw`：只读 `types.definitions` 与 `services`，`imports`、`errors` 被忽略。
- `packages/conjure-compiler/src/compiler.ts:205` — `fullPath = basePath + httpPath`，契约路径的实际拼接点。
- `packages/conjure-codegen/src/codegen.ts:35` — `generateFromIr`，类型/错误/服务/barrel 四类产物的总装。
- `packages/conjure-codegen/src/service-generator.ts:122` — 生成方法体：路径插值 + `client.fetch` 调用。
- `packages/conjure-codegen/src/naming.ts:37` — `toFileName`，包结构被压平成 kebab-case 文件名的位置。
- `packages/api-types/src/index.ts:4` — 「Hand-written TypeScript types」：`api-types` 不是生成物的自述。
- `package.json:19` — `generate:api` 指向一个没有脚本的 turbo 任务。
- `packages/client/src/client.ts:28` — 四层中间件的实际组合顺序，重试失效的根因。
- `packages/client/src/platform-fetch.ts:24` — `new URL("api" + path, baseUrl + "/")`，SDK 端点路径的约定。
- `services/svc-objects/src/routes/v1/serializers.ts:3` — v1/v2 对象形状分歧的注释与投影函数。
- `services/svc-datasets/src/routes/v1/files.ts:74` — 未服务的 v1 上传操作及其原因记录处。

---

## 第 5 章 身份、权限与多租户安全

### 5.1 本章导读

OpenFoundry 的每个请求都要回答两个问题：调用者是谁，以及它能否对这个资源做这件事。本章讲这两件事的实现——`svc-multipass` 作为 OAuth2 授权服务器签发 JWT，`svc-gateway` 在边缘校验令牌并把身份转发给下游，`@openfoundry/permissions` 在服务侧做 Organization → Marking → Role 三层判定，PostgreSQL 的 `org_rid` 与行级安全（RLS）则提供更粗的租户边界。前端登录交互见〈前端应用〉章，审计日志的内容见〈平台业务服务〉章，密钥与部署拓扑见〈工程化与交付〉章。

### 5.2 职责与边界

这一层是「横切」的：它不属于任何单个业务服务，而是被所有服务共享，并由网关统一入口。分工可以一句话概括——**网关只证明身份，服务自己决定权限，数据库兜底租户**。

| 组件 | 位置 | 负责 | 不负责 |
|------|------|------|--------|
| `svc-multipass` | `services/svc-multipass`，默认 :8084 | 用户目录、密码校验、OAuth2 授权码与 client_credentials、JWT 签发、legacy 角色映射 | 任何资源级鉴权；它没有 `registerAuthorization` |
| `svc-gateway` | `services/svc-gateway`，默认 :8080 | 令牌校验、限流、`x-user-*` 剥离与重写、反向代理；自己的 `/api/v2/search` 做可见性过滤 | 业务资源的角色判定（只把请求转给服务） |
| `@openfoundry/permissions` | `packages/permissions` | 三层判定、路由守卫、grants/policy 存储抽象 | 用户与令牌；它只读 `x-user-*` 或 claims |
| `@openfoundry/markings` | `packages/markings` | Marking 与密级类型、`MarkingEvaluator` | 未接入任何服务（只被自身测试使用），实际 Marking 判定在 permissions 里 |
| PostgreSQL RLS | `db/migrations/009_multi_tenancy.sql` | `org_rid` 列、策略、`FORCE ROW LEVEL SECURITY` | 不替代权限模型；两者是分层关系 |

### 5.3 威胁模型与信任边界

系统里有一个明确的信任分界：**网关是唯一验证令牌的地方，服务端口假定只有内网可达**。这张表把各区域的可达者、信任假设和风险列在一起；后面的所有设计都围绕它展开。

| 区域 | 可达者 | 信任假设 | 主要风险 | 现有控制 |
|------|--------|----------|----------|----------|
| 浏览器与外部客户端 → 网关 :8080 | 任意 | 不可信 | 伪造 `x-user-*`、伪造/重放 JWT、拦截授权码、开放重定向 | `services/svc-gateway/src/proxy.ts` 剥离 `x-user-*`；ES256 + `iss` + `aud` 校验；PKCE S256；闭客户端注册与精确 redirect URI |
| 网关进程内 | 网关自身 | 唯一令牌验证点 | 中间件顺序错位、`AUTH_PUBLIC_KEY` 为空 | `authPlugin` / `rateLimitPlugin` 直接挂在根实例；启动日志告警；测试钉住 `/metrics` |
| 服务端口 :8081–:8092 | 假定仅内网 | 无条件信任 `x-user-*` | 端口一旦公网可达，认证与剥离全部失效 | 无网络层控制（部署责任）；`ENFORCE_PERMISSIONS` 默认 false |
| PostgreSQL | 服务进程与运维 | `app.org_rid` 决定可见行 | 连接角色绕过 RLS、未设置会话变量 | RLS + `FORCE`；但 `deploy/` 以超级用户连接，策略实际不生效 |
| 种子/同步脚本 | 仓库运维 | 直连服务端口，不带令牌 | 绕过网关即绕过认证 | 凭据只从环境读取（`scripts/lib/auth.sh`）；端口不得公网暴露 |

几个具体的攻击面值得单独点出：

- **身份伪造**：下游服务只读 `x-user-*`，因此客户端若能把这些头送到服务，就等价于自称管理员。网关对 `x-user-` 前缀一律丢弃（`services/svc-gateway/src/proxy.ts:42`、`:83`），并有测试断言「即使网关没有 claims 也丢弃」（`services/svc-gateway/tests/trusted-identity.test.ts:94-106`）。
- **令牌伪造**：JWT 用 ES256 非对称签名，验证方只需要公钥；`validateToken` 把算法固定为 `["ES256"]`，默认 60 秒时钟容差（`packages/auth-tokens/src/token-validator.ts:166-171`）。
- **授权码拦截**：`/authorize` 对公开客户端强制 PKCE S256，且拒绝 `plain`（`services/svc-multipass/src/routes/authorize.ts:113-128`）。
- **客户端冒充与开放重定向**：客户端注册表是封闭的，`redirect_uri` 必须与声明完全一致，未知客户端连错误都不重定向（`services/svc-multipass/src/routes/authorize.ts:66-97`）。
- **提权**：历史角色 `EDITOR` / `VIEWER` 不再表示「全平台可写/可读」，而是被映射成平台根上的 editor/viewer 授权，并带弃用警告（`services/svc-multipass/src/store/user-store.ts:61`、`:113-139`）。

### 5.4 代码结构

| 路径 | 作用 |
|------|------|
| `services/svc-multipass/src/routes/oauth.ts` | `/multipass/api/oauth2/token`、`/revoke`、`/userinfo`、OIDC discovery |
| `services/svc-multipass/src/routes/authorize.ts` | 登录页 GET/POST、请求校验、code 签发 |
| `services/svc-multipass/src/routes/auth.ts` | 开发用密码登录、开放注册、`/auth/me`、`getCurrent` |
| `services/svc-multipass/src/store/` | 用户目录（内存/Pg）、客户端注册表、refresh token 与 code 的内存存储 |
| `services/svc-multipass/src/{keys,pkce,passwords,login-page}.ts` | ES256 密钥、PKCE、scrypt、登录页 HTML 与 CSP |
| `services/svc-multipass/src/cli/users.ts` | 建账号、改密码、设平台角色/组/Marking（唯一写账号的路径） |
| `services/svc-gateway/src/server.ts` | 中间件与路由的注册顺序（本章的核心证据之一） |
| `services/svc-gateway/src/middleware/auth.ts` | Bearer 校验、免鉴权前缀、`orgRid` 提取 |
| `services/svc-gateway/src/proxy.ts` | 逐跳头剥离、`x-user-*` 重写、`Content-Length` 丢弃、502 |
| `services/svc-gateway/src/rate-limiter.ts` | 令牌桶（内存/Redis）、键的选取 |
| `packages/permissions/src/{identity,roles,authorizer,authorization-store,middleware}.ts` | 身份、角色、判定、存储、路由守卫 |
| `packages/auth-tokens/src/*.ts` | claims 类型、签发、校验、scope、BearerToken 封装 |

### 5.5 认证：两条 OAuth2 流程

`svc-multipass` 同时实现授权码（带 PKCE，给浏览器）和 client_credentials（给脚本与服务）两条流程。两者共用一个 token 端点，靠 `grant_type` 分派；不支持的取值直接 400（`services/svc-multipass/src/routes/oauth.ts:72-90`）。

#### 5.5.1 授权码 + PKCE：浏览器怎么拿到令牌

```mermaid
sequenceDiagram
  autonumber
  participant B as 浏览器
  participant M as svc-multipass 8084
  participant T as TokenStore 内存
  B->>M: GET /multipass/api/oauth2/authorize client_id redirect_uri scope code_challenge S256
  M->>M: validateAuthorizeRequest 闭注册表 精确 redirect_uri 仅 S256
  M-->>B: 200 登录页 CSP 禁脚本 no-store
  B->>M: POST /authorize 隐藏字段加 username password
  M->>M: authenticate 校验 scrypt 口令 未知用户也做一次派生
  M->>T: storeAuthCode 十分钟 单次
  M-->>B: 303 redirect_uri code 与 state
  B->>M: POST /multipass/api/oauth2/token authorization_code code code_verifier
  M->>T: consumeAuthCode 单次消费
  M->>M: 校验 code 的 client 绑定 redirect_uri PKCE
  M->>M: userClaims 重新读用户 禁用或降权立即生效
  M-->>B: access_token ES256 可选 refresh_token
```

流程的每一步都有明确的拒绝分支，按 `validateAuthorizeRequest` 的顺序（`services/svc-multipass/src/routes/authorize.ts:62-140`）：

1. `client_id` 缺失或不在注册表 → 页面错误，**不重定向**（避免把错误送到未验证的地址）。
2. `redirect_uri` 必须与客户端声明的某一项完全相等；客户端只有一个 URI 时可以省略（`services/svc-multipass/src/routes/authorize.ts:80-97`）。
3. `response_type` 必须是 `code`，客户端必须被允许 `authorization_code`（`services/svc-multipass/src/routes/authorize.ts:106-111`）。
4. 有 `code_challenge` 时方法必须是 S256 且 challenge 符合 RFC 7636 的 43–128 字符；公开客户端没有 `code_challenge` 直接拒绝（`services/svc-multipass/src/routes/authorize.ts:113-128`）。
5. `scope` 必须是该客户端声明 scope 的子集（`services/svc-multipass/src/routes/authorize.ts:130-137`）。
6. 只有密码校验通过才签发 code（`services/svc-multipass/src/routes/authorize.ts:219-247`），POST 用 303 而不是 302，避免浏览器把凭据重新 POST 给客户端的回调地址（`services/svc-multipass/src/routes/authorize.ts:205-207`）。
7. code 存在内存里，10 分钟过期、只能消费一次（`services/svc-multipass/src/store/token-store.ts:29`、`:86-94`）。

`/authorize` 不设 consent 步骤，也不需要 cookie 或服务端会话：授权请求由隐藏表单字段在 GET 与 POST 之间携带，POST 重新完整校验一遍（`services/svc-multipass/src/routes/authorize.ts:8-20`）。登录页是纯 HTML，响应头禁止脚本、禁止被 iframe 嵌入（`services/svc-multipass/src/login-page.ts:20-31`）。

#### 5.5.2 client_credentials：脚本与服务怎么拿到令牌

```mermaid
sequenceDiagram
  autonumber
  participant S as 脚本或服务
  participant G as svc-gateway 8080
  participant M as svc-multipass 8084
  S->>G: POST /multipass/api/oauth2/token client_credentials client_id client_secret
  G->>M: 代理 该前缀免鉴权
  M->>M: authenticatedClient 机密客户端 摘要比较密钥
  M->>M: 校验 grantTypes 含 client_credentials 公开客户端拒绝
  M->>M: sub 为 service 前缀 clientId org 为 default-org roles 取客户端声明
  M-->>S: access_token 无 refresh_token
  S->>G: GET /api/v2/... Authorization Bearer
  G->>G: validateToken 校验 ES256 iss aud exp
  G->>G: 丢弃客户端 x-user- 按 claims 写入 x-user-
  G-->>S: 代理响应 下游据此鉴权
```

要点：

- 只有机密客户端（声明了 `clientSecret`）能做 client_credentials；`OAUTH_CLIENTS` 解析时就把公开客户端配这个 grant 判为配置错误（`services/svc-multipass/src/store/client-store.ts:207-209`）。
- 密钥比较用固定长度摘要加 `timingSafeEqual`，长度与首个不同字节都不泄露（`services/svc-multipass/src/store/client-store.ts:108-122`）。
- 主体是 `service:<clientId>`，不是某个用户；`org` 固定为 `default-org`，`roles` 只来自客户端声明（`services/svc-multipass/src/routes/oauth.ts:250-255`）。
- client_credentials 永远不发 refresh token（`services/svc-multipass/src/routes/oauth.ts:317-332` 的条件是 `forUser && client.grantTypes.includes("refresh_token")`）。

#### 5.5.3 JWT 的声明、签名与生命周期

令牌是 ES256 签名的 JWT，默认有效期 3600 秒（`services/svc-multipass/src/config.ts:70`），issuer 固定 `openfoundry-multipass`、audience `openfoundry-api`（`services/svc-multipass/src/routes/oauth.ts:294-306`）。claims 的结构定义在 `packages/auth-tokens/src/claims.ts:8-55`：

| claim | 含义 | 谁写 | 缺省/备注 |
|-------|------|------|-----------|
| `sub` | 主体 | 用户登录为 `user.username`；客户端为 `service:<clientId>` | 是用户名，不是 RID |
| `org` | 组织 | 用户的 `orgRid`；无组织或客户端为 `default-org` | 下游用于 Organization 检查 |
| `roles` | 平台角色 | 用户的 `platform_roles`；客户端自己的 `roles` | 空格分隔；只有 `ADMIN` / `SERVICE_ACCOUNT` 有意义 |
| `groups` | 组 | 用户的 `group_ids` | 空格分隔；客户端不携带 |
| `markings` | Marking | 用户的 `marking_ids` | 空格分隔；客户端不携带 |
| `scope` | OAuth scope | 默认 `api:read api:write` | 签发后没有任何地方校验（见「已知限制」） |
| `sid` / `jti` | 会话 ID / 令牌 ID | 每次签发随机生成 | `sid` 不是服务端会话，没有会话存储 |
| `exp` / `iat` | 过期/签发时间 | 签发时计算 | 秒级 Unix 时间 |

一次开发账号登录解码后的 payload 形如（值取自 `DEV_USERS` 与默认配置）：

```json
{
  "sub": "admin",
  "sid": "0b0f0a4e-6a1f-4c1d-9a1e-2f1b6c7d8e90",
  "jti": "ri.multipass.main.token.xxxxx",
  "org": "default-org",
  "svc": "multipass",
  "iss": "openfoundry-multipass",
  "aud": "openfoundry-api",
  "scope": "api:read api:write",
  "roles": "ADMIN",
  "iat": 1760000000,
  "exp": 1760003600
}
```

生命周期上有三个设计点：**claims 在每次签发时从记录重新读取**（登录、code 交换、refresh 都会重读用户，禁用或降权在下一个令牌生效，`services/svc-multipass/src/routes/oauth.ts:180-191`）；**refresh token 轮换**（用旧换新时旧 token 立即撤销，`services/svc-multipass/src/routes/oauth.ts:276-280`）；**密钥对可缺失**（没有配置时生成临时 ES256 对，重启即失效，`services/svc-multipass/src/keys.ts:23-40`）。公钥导入强制 PEM SPKI 格式，避免把 PEM 字节当成 HMAC 密钥（`packages/auth-tokens/src/token-validator.ts:84-108`）。

账号来源只有两条：`DEV_USERS`（`admin` / `developer` / `analyst`，密码公开在仓库里，只在 `NODE_ENV !== "production"` 时加载，`services/svc-multipass/src/store/user-store.ts:252-261`）与 `users` 表。生产账号的密码与平台角色只能由 `services/svc-multipass/src/cli/users.ts` 写入：它要求 `DATABASE_URL`，密码从环境变量或标准输入读取、从不作为命令行参数（`services/svc-multipass/src/cli/users.ts:14-39`、`:160-163`）。svc-admin 的 Users 页面能建 profile，但新账号没有密码，必须再跑 `set-password` 才能登录（`GUIDE.md:159`）；HTTP 上没有创建**可登录**账号的路径（开放注册只签发令牌、不落库）。生产环境没有数据库时目录为空，任何人都登录不了（`services/svc-multipass/src/store/user-store.ts:279-296`）。

### 5.6 网关中间件顺序

`services/svc-gateway/src/server.ts` 的注释列出了注册顺序（`:23-36`），实际代码与之基本一致，但有两个细节必须看代码才能确认：`metricsPlugin` 与 multipass 路由不在注释里，而 `authPlugin` 是**直接调用**而不是 `app.register`。

| 顺序 | 插件/钩子 | 挂载方式 | 作用 | 顺序后果 |
|------|-----------|----------|------|----------|
| 0 | 生产环境检查 | 构造 Fastify 前 | `NODE_ENV=production` 且无 `DATABASE_URL` 直接抛错 | 生产环境不允许无数据库启动 |
| 1 | CORS | `app.register(cors)` | 反射 origin、允许 credentials、暴露 `X-Request-Id` | CORS 头作用于全部路由的响应 |
| 2 | Helmet | `app.register(helmet)` | HSTS 两年、其余安全头；CSP 关闭 | 安全头作用于全部响应；登录页自带更严格的 CSP，网关不重复 |
| 3 | 限流 | 直接调用 `rateLimitPlugin(app, …)` | 令牌桶，Redis 可选，失败回退内存 | 它的 `onRequest` 排在认证之前，`orgRid` 尚未写入 |
| 4 | X-Request-Id | 直接 `app.addHook("onSend")` | 响应缺头时补 `request.id` | 只影响响应头 |
| 5 | 错误处理 | `app.setErrorHandler` | 领域错误、Fastify 4xx 原样透出、其余 500 | 4xx 不再被掩盖成 500（`services/svc-gateway/src/server.ts:141-152`） |
| 6 | 认证 | 直接调用 `authPlugin(app, {config})` | 校验 Bearer，写 `request.claims` / `request.orgRid` | 直接挂根，兄弟路由全部被覆盖 |
| 7 | 指标 | `app.register(metricsPlugin)` | 注册 `GET /metrics` 与耗时钩子 | 根上的认证钩子仍覆盖子上下文里的路由，所以 `/metrics` 也 401 |
| 8 | 路由 | `app.register(healthRoutes / v2Routes / v1Routes / multipassRoutes)` | 健康检查、v2、v1、multipass 代理 | 都在认证之后注册，顺序便于阅读 |

为什么顺序决定行为，有三层原因：

1. **Fastify 的封装**：`app.register` 创建子上下文，子上下文里的 `onRequest` 钩子只覆盖该子上下文注册的路由。`authPlugin` 与 `rateLimitPlugin` 因此被**直接调用在根实例上**；若写成 `app.register(authPlugin, …)`，钩子只覆盖插件自己的子上下文，而所有路由都是根上的兄弟插件，结果是全部不设防。这一点在 `services/svc-gateway/src/middleware/auth.ts:63-80` 与 `services/svc-gateway/tests/auth.test.ts:1-8` 都有明确记录（后者是为这个回归写的测试）。
2. **同一上下文内 `onRequest` 按注册顺序执行**：限流钩子先于认证钩子，所以 `extractKey` 读到的 `request.orgRid` 仍是 `undefined`（见「已知限制」）。
3. **根钩子向下继承**：`metricsPlugin` 虽然后注册且自带子上下文，根实例上的认证钩子仍然作用于它注册的 `/metrics`，所以「注册在认证之后」并不等于「豁免认证」；测试直接断言 `/metrics` 返回 401（`services/svc-gateway/tests/auth.test.ts:195-199`）。

### 5.7 身份传播：`x-user-*` 是可信跳输入

网关验证令牌后，把 claims 转成五个请求头再转发（`services/svc-gateway/src/proxy.ts:89-93`），下游通过 `identityFromHeaders` 还原成 `Identity`（`packages/permissions/src/identity.ts:44-56`）。

| 头 | 含义 | 谁写 | 谁读 | 缺省 |
|----|------|------|------|------|
| `x-user-id` | 主体用户名（`sub`） | 网关，从已验证 claims | `identityFromHeaders` | 无 → 匿名请求 |
| `x-user-org` | 组织（`org`） | 同上 | 同上 | 无 → 列了组织的 policy 会拒绝 |
| `x-user-groups` | 组，逗号分隔 | 同上 | 同上（按空白或逗号切分） | 无 → 空列表 |
| `x-user-markings` | Marking，逗号分隔 | 同上 | 同上 | 无 → 空列表 |
| `x-user-platform-admin` | 是否平台管理员 | 同上，claims 的 `roles` 含 ADMIN/SERVICE_ACCOUNT 时为 `true` | 同上 | 无 → false |

规则只有一条：**这五个头是「可信跳」输入，不是客户端输入**。`services/svc-gateway/src/proxy.ts` 对 `x-user-` 前缀一律先丢弃再重写，且**不区分网关是否有 claims**——没有验证到令牌时一个都不写，下游看到匿名请求（`services/svc-gateway/src/proxy.ts:42`、`:83`；测试 `services/svc-gateway/tests/trusted-identity.test.ts:94-127`）。`identityFromClaims` 只认 claims 里的 `roles` 判断平台管理员，不认任何 `x-user-roles` 头（`packages/permissions/src/identity.ts:80-90`），测试也断言客户端伪造的 `x-user-roles` 不会保留。

信任之所以成立，前提是**只有网关能到达服务端口**。种子与同步脚本直接访问 :8081–:8092，不经过网关，因此网关的剥离对它们无效（`GUIDE.md:197` 之后一段也写明这一点）；这就是「服务端口不得公网可达」不是建议而是边界条件的原因。另外，网关仍然把 `Authorization` 原样转发给下游（`services/svc-gateway/tests/trusted-identity.test.ts:194-206`），但除 multipass 的 `/userinfo` 外没有服务再验证它——下游只信头。

#### 5.7.1 一次受保护请求的完整路径

把上面几节串起来，一次「浏览器改本体」的请求走的是：

1. 浏览器带 `Authorization: Bearer <jwt>` 请求 `PATCH /api/v2/ontologies/:rid`；若请求里带了 `x-user-*`，它们在网关被丢弃（`services/svc-gateway/src/proxy.ts:72-87`）。
2. 根实例上的 `onRequest` 依次执行限流（此时 `orgRid` 未定义）与认证；认证通过后 `request.claims` 与 `request.orgRid` 就位（`services/svc-gateway/src/server.ts:95-103`、`:181`；`services/svc-gateway/src/middleware/auth.ts:128-135`）。
3. 网关把请求代理给 `svc-ontology`，用 claims 重写五个 `x-user-*` 头，并补 `x-forwarded-for` / `x-request-id`（`services/svc-gateway/src/proxy.ts:89-101`）。
4. `svc-ontology` 的 `registerAuthorization` 在 `onRequest` 里把五个头还原成 `request.identity`（`packages/permissions/src/middleware.ts:45-55`、`packages/permissions/src/identity.ts:44-56`）。
5. 路由的 `preHandler` 是 `requirePermission("ontology:write")`（`services/svc-ontology/src/routes/v2/ontologies.ts:164`），它按路径参数构造链并调用 `evaluate`。
6. `evaluate` 先查链上 policy 的 Organization 与 Marking，再查 grants（`packages/permissions/src/authorizer.ts:36-48`）；通过则进 handler，失败抛 `permissionDenied`。
7. handler 读写存储；若这是创建路由，`grantCreatorOwner` 顺手把创建者加成 owner（`packages/permissions/src/middleware.ts:144-155`）。
8. 响应沿网关原路返回，状态码与头逐跳转发；上游不可达时网关返回 502（`services/svc-gateway/src/proxy.ts:114-133`）。

任何一步缺失都会让保护失效：没有第 2 步的令牌校验，`x-user-*` 就是客户端说了算；没有第 4 步的 `registerAuthorization`，`request.identity` 永远是 undefined，演示模式下全部放行；没有第 5 步，服务就只剩认证没有鉴权。

### 5.8 授权模型

权限模型是 Foundry 的：**Organization 与 Marking 是强制控制，先于角色且对管理员同样生效；角色（owner/editor/viewer/discoverer）持有在资源或容器上，向下继承**。设计依据与来源记在 `docs/foundry-fidelity.md`，实现只有一份，没有第二套全局角色。

#### 5.8.1 三层检查：Organization → Marking → Role

```mermaid
flowchart TD
  A[请求进入受保护路由] --> B{request.identity 存在}
  B -- 否 --> C{ENFORCE_PERMISSIONS}
  C -- false --> Z[放行 演示模式]
  C -- true --> D[403 PermissionDenied]
  B -- 是 --> E{platformAdminOverridesControls 且 platformAdmin}
  E -- 是 --> Z
  E -- 否 --> F{Organization 检查}
  F -- 策略列了组织且调用者不在其中 --> G[403 organization]
  F -- 通过 --> H{Marking 检查 链上每个都持有}
  H -- 否 --> I[403 marking]
  H -- 是 --> J{platformAdmin}
  J -- 是 --> Z
  J -- 否 --> K{链上或平台根存在足够角色}
  K -- 是 --> Z
  K -- 否 --> L[403 role]
```

判定入口是 `evaluate`（`packages/permissions/src/authorizer.ts:36-48`），顺序被写死为：

1. 读出链上所有资源的 policy，先跑 `mandatoryDenial`：只要某个 policy 列了组织而调用者的 `organization` 不在其中就拒绝；再把链上所有 Marking 取并集，调用者必须**全部持有**（`packages/permissions/src/authorizer.ts:74-90`）。注意这是合取，不是「任选其一」。
2. 平台管理员在通过强制层后直接放行，注释写明它等价于「平台根上的 Owner，向下继承」（`packages/permissions/src/authorizer.ts:45`）。
3. 其余调用者取链上（含平台根）的 grants，任一 grant 的角色等级 ≥ 该操作要求即通过（`packages/permissions/src/authorizer.ts:92-96`）。角色等级与操作要求定义在 `packages/permissions/src/roles.ts:18-23`、`:80-95`。

链的构造在 `defaultResourceChain`：路径里的 `ontologyRid` / `datasetRid` / `pipelineRid` 打头，平台根 `ri.platform.main.root.root` 收尾；如果路径里给的是别名（本体 API name），解析出的 RID 也加入链，别名与 RID 两者都算（`packages/permissions/src/middleware.ts:91-97`）。这正是「alias 未解析则 Marking 不生效」的修复点：本体服务通过 `registerAuthorization` 的 `alias` 选项把 API name 解析成 RID（`services/svc-ontology/src/server.ts:123-132`）。

#### 5.8.2 四个入口

| 入口 | 适用场景 | 做什么 | 关键行为 |
|------|----------|--------|----------|
| `requirePermission(operation, options?)` | 单资源读写/删除/管理 | 前置钩子做三层判定 | 资源默认取路径参数；`options.resource` 可自定义链；`platformAdminOverridesControls` 只给 policy 路由用 |
| `requireCaller` | 列表类路由 | 只要求「有身份」 | 强制模式下匿名 403；不要求任何角色，否则只有某资源角色的人永远看不到列表 |
| `filterDiscoverable` | 列表结果的裁剪 | 用 `evaluateEach` 按 `resource:discover` 批量判定 | 匿名直接返回全量；一次读 policy、一次读 grants（`packages/permissions/src/authorizer.ts:57-72`） |
| `grantCreatorOwner` | 创建资源之后 | 给创建者加 owner grant | 匿名调用是 no-op；没有它新建资源只有管理员能用 |

四者的实现都在 `packages/permissions/src/middleware.ts`（`:103-125`、`:132-134`、`:161-175`、`:144-155`），导出见 `packages/permissions/src/index.ts`。它们共享同一个前置条件：服务必须先**直接**调用 `registerAuthorization(app, {store, alias?})` 注册存储与身份钩子（`packages/permissions/src/middleware.ts:45-55`），理由与 `authPlugin` 相同——放进 `app.register` 只覆盖子上下文。目前接入的是 ontology、objects、actions、datasets、admin 五个服务；aip、compass、functions、media、sentinel、webhooks、multipass 没有资源级检查，只依赖网关认证。

`ENFORCE_PERMISSIONS` 在模块加载时读一次环境变量（`packages/permissions/src/middleware.ts:25-31`），所以改它要重启服务；测试通过 `setEnforcePermissions` 覆盖。默认 false 意味着**没有身份就不检查**，这正是演示模式能无令牌运行的原因。

#### 5.8.3 grants 与 policy 的数据结构

```ts
/** 一条「谁在哪个资源上持有什么角色」，向下继承。 */
interface RoleGrant {
  resourceRid: string;
  principalType: "user" | "group";
  principalId: string;      // 用户名或组 id，不是 RID
  roleId: "owner" | "editor" | "viewer" | "discoverer";
  grantedBy: string;        // 用户名，或 legacy-role-mapping
  grantedAt: string;        // ISO 时间
}
/** 资源上的强制控制，两者都是合取且继承。 */
interface ResourcePolicy {
  resourceRid: string;
  organizations: readonly string[];
  markings: readonly string[];
}
```

存储有两种，由 `createAuthorizationStore(pool?)` 选择（`packages/permissions/src/authorization-store.ts:250-252`）：

| 形态 | 位置 | 特点 |
|------|------|------|
| 文件（默认） | `/tmp/openfoundry-data/authorization.json` | 全部服务共享；读取时按 mtime 重载；写入走临时文件 + rename（`packages/permissions/src/authorization-store.ts:77-163`） |
| PostgreSQL | `resource_role_grants`、`resource_policies` | 由 `db/migrations/014_resource_authorization.sql:18-35` 建表；`ON CONFLICT DO NOTHING` 保证授权幂等 |

文件存储的 `formatVersion: 1` 形状里同时放 grants 与 policies；测试用 `new FileAuthorizationStore(null)` 完全绕开磁盘（`packages/permissions/src/authorization-store.ts:82`）。管理面在 `services/svc-admin/src/routes/v2/permissions.ts`：`GET /admin/roles` 只读，`POST /admin/resources/:resourceRid/roles/add|remove`、`GET|PUT .../policy` 需要 Owner，其中角色清单与 policy 的 wire 形状按 Foundry 的 filesystem API 对齐。

#### 5.8.4 平台管理员：唯一豁免

平台管理员由 claims 里的 `roles` 决定：`ADMIN` 或 `SERVICE_ACCOUNT` 任意一个出现即 `platformAdmin = true`（`packages/permissions/src/roles.ts:109-118`、`packages/permissions/src/identity.ts:89`）。它在权限模型里只有两处特殊：

- `evaluate` 中跳过角色层（强制层照旧），等价于平台根 Owner（`packages/permissions/src/authorizer.ts:45`）。
- 读取和替换**资源 policy** 的路由可以绕过 Organization 与 Marking 检查（`packages/permissions/src/middleware.ts:114`、`services/svc-admin/src/routes/v2/permissions.ts:71-77`）。这是刻意留的恢复路径：一个没人持有的 Marking 会让所有人（包括管理员）读不到资源内容，而 policy 路由是唯一还能改写它的入口。

legacy 角色映射是这一节的边角但必须提：`canonicalRoles` 把 `EDITOR` / `VIEWER` 从平台角色里分出来，转成平台根上的 editor/viewer 授权，`grantedBy` 记为 `legacy-role-mapping`；`set-roles` 或 svc-multipass 重启时 `syncLegacyRootGrants` 会收回不再声明的映射授权，而显式通过 API 授予的授权不受影响（`services/svc-multipass/src/store/user-store.ts:80-139`）。`OAUTH_CLIENTS` 里声明的 legacy 角色走同一条路，principal 是 `service:<clientId>`（`services/svc-multipass/src/server.ts:179-188`）。

除上述两点外，管理员没有第三个豁免面：搜索里的用户与组目录只对通过 `admin:manage` 的调用者可见，而 `admin:manage` 同样要先过平台根的强制层（`services/svc-gateway/src/routes/v2/search.ts:94-97`），普通资源的可见性仍按 `resource:discover` 逐条判定（`:84-93`）。

### 5.9 多租户：`org_rid` 与 RLS

租户隔离是**存储层**的机制，与权限模型是分层关系而不是替代关系：权限模型回答「这个调用者能不能碰这个资源」，RLS 回答「这个数据库连接能不能看见这一行」。

实现方式（`db/migrations/009_multi_tenancy.sql`）：

- 给约 35 张表加 `org_rid TEXT NOT NULL DEFAULT 'org:default' REFERENCES organizations(rid)`，并建索引（`:37-101`）。
- 每张表 `ENABLE ROW LEVEL SECURITY`，策略统一为 `USING (org_rid = current_setting('app.org_rid', true)) WITH CHECK (…)`，再 `FORCE ROW LEVEL SECURITY` 让表属主也受约束（`:129-205`）。
- 租户上下文由服务在事务里设置：`SET LOCAL app.org_rid = '<org_rid>'`（文件头注释 `:7`），实际代码用 `set_config('app.org_rid', $1, true)` 包在显式事务里，保证连接归还池时不泄漏（`services/svc-objects/src/store/pg-object-store.ts:161-176`）。
- 较新的表把列默认值也接到会话变量上：`DEFAULT COALESCE(NULLIF(current_setting('app.org_rid', true), ''), 'org:default')`，由数据库而不是 store 决定行归属（`scripts/migrate.sql:797` 及之后的 models/connectivity 段）。

关键事实是：**RLS 只在「连接角色受策略约束 + 服务真的设置了会话变量 + 服务真的知道当前租户」三者同时成立时才生效**。今天的代码只满足一部分：

- `PgObjectStore` 与 `PgConnectivityStore` 支持传入 `orgRid` 并设置会话变量（`services/svc-objects/src/store/pg-object-store.ts:161-176`、`services/svc-datasets/src/connectivity/pg-store.ts:55-63`），但 `svc-objects` 与 `svc-datasets` 构造时**都没有传**组织：`new PgObjectStore(pgPool)`（`services/svc-objects/src/server.ts:57`）、`new PgConnectivityStore(pool, SecretBox.fromEnv(...))`（`services/svc-datasets/src/server.ts:69-70`）。
- `deploy/` 用数据库超级用户连接（`deploy/.env.production:9` 的 `POSTGRES_USER=openfoundry`），超级用户绕过 RLS；`PgUserStore` 的注释也明说它需要超级用户或 `BYPASSRLS` 角色才能读登录前的用户行（`services/svc-multipass/src/store/pg-user-store.ts:15-20`）。
- `svc-ontology` 从不读 `org` claim、不写 `org_rid`、不设置会话变量（`tests/integration/tenant-isolation.test.ts:4-9`）。

> 文档与实现不一致：`AGENTS.md` 写「Tenant isolation is still `org_rid` + RLS in the stores」，指的是机制仍在、且它是与权限模型并列的一层；但仓库自己的集成测试把隔离用例标成 `it.todo`，并在文件头明确写「tenant isolation is NOT implemented … todo 与跳过都不能被当作一个租户看不见另一个租户数据的证据」（`tests/integration/tenant-isolation.test.ts:4-9`、`:161-188`）。两句话都对，但读者不能把前者理解为端到端已生效。

与权限模型的分层关系可以这样记：RLS 是**粗粒度、按行**的，靠 `org_rid` 一个维度；权限模型是**细粒度、按资源**的，靠 policy 的 `organizations` 与 `markings` 加上角色继承。一个资源可以被同一个组织内的不同角色区分读写，也可以被 policy 限制成只有列出的组织可见；但无论 policy 怎么写，RLS 都（在生效时）先把别的租户的行挡在查询之外。反过来，RLS 生效也不能替代权限模型——同租户内的越权读写仍要角色判定。

还有两处容易忽略的细节：

- `users` 与 `groups` 的策略额外允许 `org_rid IS NULL` 的行对任何租户可见（`db/migrations/009_multi_tenancy.sql:208-218`），而账号 CLI 的 `--org-rid` 默认为空（`services/svc-multipass/src/cli/users.ts:184-194`）。也就是说未显式指定组织的账号在 RLS 生效时是跨租户可见的。
- 权限存储本身不是租户隔离的：`resource_role_grants` 与 `resource_policies` 没有 `org_rid`，也没有 RLS（`db/migrations/014_resource_authorization.sql` 全文只有两张表和索引）。资源 RID 全局唯一，所以这在单租户部署里没问题，但它是「权限存储是全局的」这一事实，不应被误读为按租户分区。

### 5.10 设计决策与理由

**1. 认证在网关，鉴权在服务。** 网关只做令牌校验与身份转发，资源级判定留在拥有资源的服务里。理由是资源链、别名解析、继承关系都是领域知识，网关要判权就得复制每个服务的资源图；而且 `/api/v2/search` 这种网关自有路由已经证明「在网关判权」可行但需要直接读授权存储（`services/svc-gateway/src/routes/v2/search.ts:71-98`）。取舍是：每个服务都必须记得先 `registerAuthorization` 再注册路由，漏了就没有任何资源级保护；13 个服务里有 8 个没有接入（其中网关只在自有的 `/api/v2/search` 里直接调用 `evaluate`，不走 `registerAuthorization`）。

**2. 用 `x-user-*` 头而不是每次回查 multipass。** 令牌里已经带了 `org`、`groups`、`markings`，网关解码后写成头，下游零网络跳转即可判定。取舍是：身份有最长一个令牌生命周期的滞后（默认 1 小时），撤销与降权要等下一次签发；同时这套方案的安全性完全押在「网关剥离 + 服务端口不可达」两个前提上，任一被破坏就等于没有鉴权。

**3. 自签 JWT 而不是服务端会话。** 验证只需公钥，网关可以无状态横向扩展，不需要会话存储或粘性会话。取舍是：access token 无法即时撤销——`/revoke` 只撤销内存里的 refresh token（`services/svc-multipass/src/routes/oauth.ts:93-103`），JWT 本身到期前一直有效；密钥分发也必须成对正确（`AUTH_PUBLIC_KEY` 与 `JWT_PUBLIC_KEY` 不匹配会让所有令牌 401）。

**4. 限流保持在认证之前。** 先限流再验签，可以在做 ECDSA 验证之前挡掉洪泛，代价最小。取舍是：按租户分桶失效——限流钩子先跑，`request.orgRid` 还没写入（详见下节）。把认证提到限流之前能修好分桶，但会让未认证流量也消耗验签成本，所以顺序被刻意保留（`AGENTS.md` 对此有明确说明）。

**5. `/metrics` 不豁免。** 指标暴露路由名、状态码分布和调用量，对攻击者是侦察材料；健康检查才是编排系统真正需要的，所以只有 `/status/*` 与取令牌的前缀免鉴权。取舍是 Prometheus 需要带令牌抓取，或改用旁路；`deploy/` 里没有抓取配置，容器健康检查走 `/status/health`。

**6. 平台管理员只开一个豁免口。** 只允许绕过 policy 的 Organization/Marking 检查，不允许绕过普通资源的强制层。取舍是管理员能移除任何 Marking（敏感操作），换来的是「Marking 无人持有导致资源永久锁死」有确定的恢复路径；如果连普通读写也豁免，强制控制就名存实亡。

**7. 客户端注册表封闭、redirect URI 精确匹配、只收 S256。** 没有动态注册就没有「用自己的客户端收别人的 code」这条路；S256 是公开客户端唯一能把 code 绑到发起方的机制。取舍是运维必须在 `OAUTH_CLIENTS` 里显式声明每个客户端，改 redirect URI 要重启；开发客户端只在非 production 下种子化（`services/svc-multipass/src/store/client-store.ts:129-136`）。

**8. grants 默认放共享文件而不是数据库。** 演示模式不依赖 Postgres 也能跑通权限模型，`svc-admin` 写、其余服务按 mtime 读。取舍是跨主机部署不可用，且并发写只有「临时文件 + rename」级别的原子性，不是多写者协议；生产应改用 `PgAuthorizationStore`。

### 5.11 已知限制与坑

以下每一条都在代码里核实过，并与 `AGENTS.md` 的「Authentication and caller identity」「Identity and permissions」两节逐条对齐；证据给到代码位置，而不是转述 `AGENTS.md` 的结论。

| 现象 | 触发条件 | 影响 | 证据 |
|------|----------|------|------|
| 限流按租户分桶失效 | 开启限流且带令牌 | 每个令牌一个桶，刷新令牌即换桶 | `services/svc-gateway/src/rate-limiter.ts:273-290`、`services/svc-gateway/src/server.ts:95` 与 `:181` |
| svc-admin 用户/组路由 500 | 设置 `DATABASE_URL` | 用户全部路由与组的大部分路由不可用 | `services/svc-admin/src/routes/v2/users.ts:115`、`services/svc-admin/src/routes/v2/groups.ts:47` |
| 全部请求放行 | `AUTH_PUBLIC_KEY` 为空 | 无认证，任何客户端直连 | `services/svc-gateway/src/middleware/auth.ts:91-100`、`:113-115` |
| 全部真实令牌 401 | `AUTH_ISSUER` 与 multipass 的 `iss` 不一致 | 部署看起来配置了却全挂 | `services/svc-gateway/src/config.ts:102`、`services/svc-multipass/src/routes/oauth.ts:300` |
| 管理员可改任意 policy | 平台管理员 | 可移除任何 Marking | `packages/permissions/src/middleware.ts:114`、`services/svc-admin/src/routes/v2/permissions.ts:74-77` |
| 别名上的 Marking 不生效 | 路由接受 API name 而服务未配 alias | 用名字访问绕开 RID 上的策略 | `packages/permissions/src/middleware.ts:91-97`、`services/svc-ontology/src/server.ts:126` |
| access token 不可撤销 | 令牌未过期 | 只能等过期或换密钥 | `services/svc-multipass/src/routes/oauth.ts:93-103`、`services/svc-multipass/src/store/token-store.ts:1-5` |
| refresh token / code 重启即丢 | 服务重启或多副本 | 用户需重新登录；多副本不共享 | `services/svc-multipass/src/store/token-store.ts:31-33` |

**限流按租户分桶失效。** `extractKey` 的第一优先级是 `request.orgRid`（`services/svc-gateway/src/rate-limiter.ts:273-278`），但 `rateLimitPlugin` 在 `authPlugin` 之前被调用（`services/svc-gateway/src/server.ts:95-103` 对比 `:181`），两者都是根实例上的 `onRequest`，按注册顺序执行，所以限流跑的时候 `orgRid` 还没被赋值（`services/svc-gateway/src/middleware/auth.ts:131-135` 才写它）。于是每个带令牌的请求落到 `auth:<hash>` 分支，哈希取自 Authorization 头，令牌一换就是新桶。测试只覆盖了内存限流器与按 IP 的键，没有覆盖租户键（`services/svc-gateway/tests/rate-limiter.test.ts:142-261`）。

**`DATABASE_URL` 下 svc-admin 的用户与组路由 500。** 设置 `DATABASE_URL` 后 svc-admin 切换到 `PgUserStore` / `PgGroupStore`，方法全是 async；但用户路由的 `listUsers().filter(...)`、`getUser(...)` 等调用没有 `await`（`services/svc-admin/src/routes/v2/users.ts:101-190`），组列表直接 `groupStore.listGroups().map(...)`（`services/svc-admin/src/routes/v2/groups.ts:47`）。对 Promise 调 `.map`/`.filter` 会同步抛错，表现为 500。组列表、成员列表和三个 `groupMembers` 路由用了 `await`，两种模式都正常（`services/svc-admin/src/routes/v2/groups.ts:114-207`）。

**`AUTH_PUBLIC_KEY` 为空时全部放行。** `authPlugin` 在启动时尝试导入公钥，失败或为空就把 `publicKey` 置 null，并在钩子里直接 `return`（`services/svc-gateway/src/middleware/auth.ts:91-115`），同时打一条 warning。这是演示默认（`.env.example:55`、`GUIDE.md:165`），不是安全默认；`deploy/.env.production:29-39` 因此把三项密钥作为一对交给运维填写。

**`iss` 不匹配导致全部 401。** multipass 把 `iss` 硬编码为 `openfoundry-multipass`（`services/svc-multipass/src/routes/oauth.ts:300`），网关默认值也是它（`services/svc-gateway/src/config.ts:99-102`）。如果运维把 `AUTH_ISSUER` 设成别的值，签名能过、issuer 校验必失败，所有真实令牌 401。`deploy/` 因此**故意不设置**该变量（`deploy/docker-compose.yml:192-194` 的注释）。同理，`AUTH_PUBLIC_KEY` 与 `JWT_PUBLIC_KEY` 不成对也会全 401，且 multipass 在只配一半密钥时会退回临时密钥对并告警（`services/svc-multipass/src/keys.ts:23-40`、`services/svc-multipass/src/config.ts:78-95`）。

**平台管理员无 Marking 豁免是恢复路径。** 只有 `platformAdminOverridesControls: true` 的 policy 路由会跳过强制层（`packages/permissions/src/middleware.ts:114`），其余路由即使 `platformAdmin` 也要先过 Organization 与 Marking（`packages/permissions/src/authorizer.ts:43-45`）。这既是恢复手段，也意味着管理员账号本身是高价值目标——它可以在不持有 Marking 的情况下改写任何资源的 policy。

**alias 未解析则 Marking 不生效。** `defaultResourceChain` 只有在服务提供了 `alias` 时才会把「别名 + RID」都放进链（`packages/permissions/src/middleware.ts:91-97`）。本体服务提供了（`services/svc-ontology/src/server.ts:123-132`），所以 API name 与 RID 都受同一 policy 约束；objects/actions 把数据存在路径里的 `ontologyRid` 字符串下，授权必须贴在调用者实际使用的标识符上（`docs/foundry-fidelity.md` 末尾也记了这一点）。

其余在代码里确认、值得写进风险清单的：

- **`scope` 只签发不校验。** `client_credentials` 甚至接受请求方自报的 `scope`（`services/svc-multipass/src/routes/oauth.ts:250`），而网关与所有服务都不读 scope——`parseScopes` / `hasScope` / `satisfiesScope` 只出现在 `@openfoundry/auth-tokens` 自身及其测试里。scope 目前是装饰性的，授权完全靠角色与 Marking。
- **开放注册不持久化账号。** `POST /api/v2/auth/signup` 在 `OPENFOUNDRY_ALLOW_OPEN_SIGNUP=1` 时才开放，但用户与组织不写库，密码哈希算完即弃（`services/svc-multipass/src/routes/auth.ts:99-102`），签发的令牌里 `org` 也没有任何下游消费者。
- **两个端点不验签就解 JWT。** `/multipass/api/auth/me` 与 `/api/v2/admin/users/getCurrent` 只做 base64url 解码，不验证签名（`services/svc-multipass/src/routes/auth.ts:188-198`、`:219-226`）。在网关后面它们仍受令牌校验保护，但直连服务端口时等于没有校验。
- **`packages/markings` 与权限模型不是一套语义。** `MarkingEvaluator` 支持 MANDATORY（全持有）与 DISCRETIONARY（至少一个）（`packages/markings/src/marking-evaluator.ts:56-69`），但 permissions 的 `mandatoryDenial` 只做合取，且没有任何服务引用 `MarkingEvaluator`（只有它自己的测试）。如果将来要支持「可选 Marking」，需要把它接进 authorizer，而不是并排再放一套。
- **8 个服务没有资源级检查**：aip、compass、functions、media、sentinel、webhooks、multipass（以及网关自身的代理路由）。它们的边界只有「网关认证」这一层。
- **`scripts/migrate-rls.sql` 是漂移的死脚本。** 它设置的会话变量是 `app.current_org_rid`（`:5`、`:33`），与 `db/migrations` 的 `app.org_rid` 不同；默认组织 RID 是 `ri.multipass.main.organization.default`（`:49`），与 `org:default` 不同；还引用了两套 schema 里都不存在的 `actions`、`monitors` 表（`:14-15`）。仓库里没有任何脚本或文档引用它。
- **服务端口没有任何网络层保护。** 13 个服务（含网关）的 `HOST` 默认值都是 `0.0.0.0`（例如 `services/svc-gateway/src/config.ts:97`），网关之外的 12 个后端端口在本地运行时直接可达；Compose 只映射了网关与控制台，但「不要把这些端口暴露出去」是部署责任，不是代码保证。

> 文档与实现不一致：`deploy/.env.production:75-81` 与 `deploy/docker-compose.yml:33-34` 都写「网关故意不向下游断言 `X-User-*`，所以 `ENFORCE_PERMISSIONS=true` 会拒绝每个请求」。现在的 `services/svc-gateway/src/proxy.ts:89-93` 已经会从已验证 claims 写入这五个头，`trusted-identity.test.ts` 也覆盖了这条路径；注释停留在旧实现。注意这只影响注释：`deploy/` 仍把 `ENFORCE_PERMISSIONS` 设为 false，而 `AUTH_PUBLIC_KEY` 默认也为空，所以部署行为没有变化。

> 文档与实现不一致：`README.md:275-276` 把 `POST /api/v2/auth/token` 与 `GET /api/v2/auth/me` 列为认证端点，但 multipass 没有这两条路由（`services/svc-multipass/src/routes/auth.ts` 只有 `/multipass/api/auth/login`、`/multipass/api/auth/me`、`/api/v2/auth/signup`、`/api/v2/admin/users/getCurrent`），网关的 `/api/v2/auth/*` 只是把请求转发给 multipass，所以这两个地址实际 404。README 同段的 `/multipass/api/oauth2/*` 是准确的。

> 文档与实现不一致：`services/svc-gateway/src/config.ts:23-26` 的字段注释写「PEM-encoded public key (or JWKS URL)」并提到 `AUTH_JWKS_URL`，但实现只接受 PEM：`authPlugin` 调用 `importVerificationKey`，非 `-----BEGIN PUBLIC KEY-----` 的值直接抛错（`services/svc-gateway/src/middleware/auth.ts:91-93`、`packages/auth-tokens/src/token-validator.ts:84-108`），全仓库也没有读取 `AUTH_JWKS_URL` 的代码。JWKS 尚不支持。

### 5.12 关键代码位置

| 位置 | 说明 |
|------|------|
| `services/svc-gateway/src/server.ts:168-193` | 认证、指标、路由的注册顺序；直接调用而非 `register` 的原因写在注释里 |
| `services/svc-gateway/src/middleware/auth.ts:29-40` | 免鉴权前缀与开放注册开关 |
| `services/svc-gateway/src/middleware/auth.ts:91-148` | 公钥为空放行、Bearer 提取、claims 与 `orgRid` 写入、401 |
| `services/svc-gateway/src/proxy.ts:42-93` | `x-user-*` 剥离与按 claims 重写；`Content-Length` 丢弃 |
| `services/svc-gateway/src/rate-limiter.ts:273-329` | 限流键的选取与 429 响应 |
| `services/svc-gateway/src/routes/v2/search.ts:53-98` | 网关自有路由用 `evaluateEach` 过滤可见性 |
| `services/svc-multipass/src/routes/authorize.ts:62-140` | 授权请求校验：闭注册表、精确 redirect URI、PKCE、scope |
| `services/svc-multipass/src/routes/oauth.ts:193-335` | 两种 grant 的处理、refresh 轮换、claims 组装 |
| `services/svc-multipass/src/store/user-store.ts:61-139` | legacy EDITOR/VIEWER 映射与 `syncLegacyRootGrants` |
| `packages/permissions/src/authorizer.ts:36-96` | Organization → Marking → Role 的判定与平台管理员跳过角色层 |
| `packages/permissions/src/middleware.ts:25-175` | `ENFORCE_PERMISSIONS`、`registerAuthorization`、四个入口 |
| `db/migrations/009_multi_tenancy.sql:129-218` | RLS 策略与 `FORCE ROW LEVEL SECURITY`；users/groups 的 NULL 组织例外 |

---

## 第 6 章 本体与对象数据面

### 6.1 本章导读

OpenFoundry 的「本体」层把 Palantir Foundry 的类型系统压缩成一个可运行的本地实现：一组 TypeScript 类型负责**声明**对象类型、属性、链接、动作与查询，两个 Fastify 服务负责把这些声明**存下来**并变成可读写的对象数据——`svc-ontology`（:8081）管元数据，`svc-objects`（:8082）管对象与链接，`packages/object-set` 提供查询求值，`packages/search` 提供平台级全文检索。本章沿着「声明 → 存储 → 投影到线上格式 → 查询与聚合」这条链路走一遍，重点在数据面而不是动作执行语义（后者见〈平台业务服务〉章）。

边界先讲清楚：这一层没有本体事务、没有本体分支、没有场景，也没有跨对象类型的引用完整性约束。每次写入都是单条记录级的读写，一致性只到存储后端为止；所谓「查询」是在服务进程内存里对数组做过滤、排序、切片，而不是把谓词下推给数据库。

### 6.2 职责与边界

四个元数据包与两个服务各管一段，边界比名字看起来更松：

- `packages/ontology-schema` 是唯一的**类型权威**，既定义结构（`OntologyDefinition` 及其五类成员），也提供逐类型的校验函数。运行时服务只从它导入**类型**，不导入校验器。
- `packages/ontology-ir` 把 schema 定义编译成中间表示 `OntologyIr`，并负责 JSON 序列化与结构校验。
- `packages/ontology-generator` 从 IR 生成 TypeScript 对象接口、actions 与 client 骨架。
- `packages/ontology-maker` 是一层链式 builder DSL，产出 `OntologyDefinition`。
- `svc-ontology` 维护本体元数据的 CRUD 与 v1/v2 投影，存储是 `Map` + JSON 文件或 Postgres。
- `svc-objects` 维护对象与链接数据，按 ontology 作用域存储，并实现 ObjectSet 求值、搜索、聚合、CSV 导入、语义搜索代理。

这一层**不做**的事：不校验写入的元数据是否自洽（`validateOntology` 没有接进任何 HTTP 路由，见下文）；不强制链接两端的对象存在；不提供事务、分支或回滚；不在存储层做谓词下推（`PgObjectStore.queryObjects` 里有一套 `filterToSql` 翻译，但该方法是死代码，见〈已知限制〉）。

### 6.3 代码结构

| 路径 | 作用 |
|------|------|
| `packages/ontology-schema/src/ontology.ts` | `OntologyDefinition` 与跨引用校验 `validateOntology` |
| `packages/ontology-schema/src/object-type.ts` | `ObjectTypeDefinition`、主键/标题属性约束 |
| `packages/ontology-schema/src/property-types.ts` | 21 种 `PropertyType`、`PropertyDef`、构造辅助函数 |
| `packages/ontology-schema/src/link-type.ts` | 外键式 `LinkTypeDefinition`，拒绝自引用链接 |
| `packages/ontology-ir/src/ir-converter.ts` | `ontologyToIr` / `irToOntology` 双向转换 |
| `packages/ontology-ir/src/ir-serializer.ts` | IR 的 JSON 序列化与 `validateOntologyIr` |
| `packages/ontology-generator/src/generator.ts` | `generateFromOntologyIr`，产出多文件 TS SDK 骨架 |
| `packages/ontology-maker/src/define-*.ts` | `defineOntology` / `defineObject` / `defineAction` / `defineLink` / `defineInterface` |
| `services/svc-ontology/src/store/ontology-store.ts` | 内存 + JSON 文件元数据存储 |
| `services/svc-ontology/src/store/pg-ontology-store.ts` | Postgres 元数据存储（全异步） |
| `services/svc-ontology/src/routes/v1/serializers.ts` | v1 元数据投影，与 v2 分开 |
| `services/svc-objects/src/store/object-store.ts` | `ObjectReadWriteStore` 接口、内存实现、过滤器与聚合求值 |
| `services/svc-objects/src/store/link-store.ts` | 内存链接存储，按 ontology + 复合键索引 |
| `services/svc-objects/src/store/pg-object-store.ts` | Postgres 对象存储，全异步，`org_rid` 可选 |
| `services/svc-objects/src/store/ontology-scoped-file.ts` | `{ formatVersion: 2, ontologies }` 落盘格式与旧文件迁移 |
| `services/svc-objects/src/routes/v2/object-sets.ts` | `loadObjects` / `aggregate` / `search` 与 ObjectSet 递归求值 |
| `services/svc-objects/src/routes/v1/search-query.ts` | v1 `SearchJsonQuery` 的独立求值器 |
| `packages/object-set/src/query-engine.ts` | 共享查询引擎（where 求值、聚合、全文搜索） |
| `packages/search/src/pg-search.ts` | 平台级 `search_index` 全文检索，与对象过滤无关 |
| `packages/pagination/src/page-token.ts` | base64url 游标编解码，含 `snapshotSize` |

### 6.4 领域模型

本体是一棵两层树：一个 Ontology 拥有五类具名成员，成员之间用 **apiName 字符串**互相引用，而不是用对象引用或 RID。下面的关系图把声明层的引用关系画出来，属性（Property）是唯一以「字典键」形式挂在宿主上的成员。

```mermaid
erDiagram
  ONTOLOGY ||--o{ OBJECT_TYPE : declares
  ONTOLOGY ||--o{ LINK_TYPE : declares
  ONTOLOGY ||--o{ ACTION_TYPE : declares
  ONTOLOGY ||--o{ INTERFACE_TYPE : declares
  ONTOLOGY ||--o{ QUERY_TYPE : declares
  OBJECT_TYPE ||--o{ PROPERTY : owns
  INTERFACE_TYPE ||--o{ PROPERTY : requires
  OBJECT_TYPE }o--o{ INTERFACE_TYPE : implements
  LINK_TYPE }o--|| OBJECT_TYPE : source
  LINK_TYPE }o--|| OBJECT_TYPE : target
  ACTION_TYPE ||--o{ PARAMETER : takes
  ACTION_TYPE }o--o{ OBJECT_TYPE : modifies
```

图里所有关系都是「声明上的关系」，不是存储上的外键：链接类型只记录源类型、目标类型和一个源侧外键属性名，链接数据本身是独立记录；`implements` 与 `extendsInterfaces` 只是字符串数组，没有任何东西保证被引用的接口存在——除了 `validateOntology`，而它只被测试和 `packages/test-utils` 调用。

顶层类型定义很直白：五个 `Record<apiName, Definition>`，加三个身份字段（`rid`、`apiName`、`displayName`）和描述。真正的约束集中在对象类型上：

```ts
// packages/ontology-schema/src/object-type.ts:27（节选，保留字段与注释）
export interface ObjectTypeDefinition {
  readonly apiName: string;                 // 仅在一个 Ontology 内唯一，跨本体可重名
  readonly description: string;
  readonly primaryKeyApiName: string;       // 必须指向 properties 中的一个键
  readonly primaryKeyType: PropertyType;    // 21 种 PropertyType 之一
  readonly titlePropertyApiName: string;    // 展示用标题属性
  readonly properties: Record<string, PropertyDef>;
  readonly implements: readonly string[];   // 实现的 interface apiName
  readonly status: ObjectTypeStatus;        // ACTIVE | EXPERIMENTAL | DEPRECATED
}
```

`PropertyDef` 用三个泛型参数表达「类型 / 可空 / 重数」，再为 `VECTOR`、`STRUCT`、`TIMESERIES` 挂上维度、嵌套字段与值类型（`packages/ontology-schema/src/property-types.ts:109`）。21 种属性类型从 `STRING`、`INTEGER` 一直到 `GEOSHAPE`、`MARKING`、`VECTOR`、`MEDIA_REFERENCE`、`RID`（`packages/ontology-schema/src/property-types.ts:13`），其中 `LONG` 映射到 `bigint`、`DECIMAL` 映射到 `string`（`packages/ontology-schema/src/property-types.ts:47-51`）。这些映射只在类型层有意义：存储里属性一律是 `Record<string, unknown>`，没有运行时类型检查。

三条值得记住的声明约束：

1. 主键属性不允许可空，`validateObjectType` 会报错（`packages/ontology-schema/src/object-type.ts:126-138`）。
2. 链接类型拒绝自引用，即 `objectTypeApiName === linkedObjectTypeApiName` 直接报错（`packages/ontology-schema/src/link-type.ts:93-99`）。这与「员工汇报给员工」这类真实需求冲突，是一个被刻意收紧的限制。
3. 动作类型的每个 `modifiedEntities` 条目必须至少 `created` 或 `modified` 之一为真（`packages/ontology-schema/src/action-type.ts:109-116`）。

### 6.5 元数据生成链：四个包，只有一半在跑

从作者视角看，元数据有两条互不相交的路径：**离线生成链**（maker → schema → IR → generator）和**运行时服务链**（schema 类型 → svc-ontology 存储 → v1/v2 路由）。它们共享 `ontology-schema` 的类型，但没有任何一个服务会去读生成链的产物。

```mermaid
flowchart LR
  subgraph AUTHOR[作者侧]
    M[ontology-maker builder] --> DEF[OntologyDefinition]
    DEF --> VAL[validateOntology]
    DEF --> CONV[ontologyToIr]
    CONV --> IR[OntologyIr JSON]
    IR --> GEN[ontology-generator]
    GEN --> OUT[对象接口 actions client index]
  end
  subgraph RUNTIME[运行时]
    TYPES[ontology-schema 类型] --> SVC[svc-ontology]
    SVC --> ST[OntologyStore 或 PgOntologyStore]
    ST --> WIRE[v1 与 v2 投影]
  end
```

图分成两半是有意的：上半段（生成链）没有任何下游消费者，下半段（运行时）只消费类型定义，不消费生成物。逐包核实的结果如下。

| 包 | 职责 | 仓库内真实消费者 | 状态 |
|----|------|------------------|------|
| `@openfoundry/ontology-schema` | 类型权威 + 校验函数 | `svc-ontology` 的 store 与路由、`@openfoundry/sdk` 类型、`test-utils` 的 mock | 活跃，但只用类型 |
| `@openfoundry/ontology-ir` | schema ↔ IR 转换、IR 校验 | 只有 `@openfoundry/ontology-generator` | 生成链内部 |
| `@openfoundry/ontology-generator` | 从 IR 生成 TS 代码 | **没有**（无任何 package.json 依赖它） | 未接线 |
| `@openfoundry/ontology-maker` | builder DSL | 只有 `apps/app-docs` 的代码示例页 | 文档道具 |

生成物的用途从 `packages/ontology-generator/src/generator.ts` 的文件名就能看出来：每个对象类型一个接口文件、一个 `actions.ts`、一个 `client.ts`（`OntologyClient` 接口，含 `ObjectApi<T>` 的 `get`/`list`/`filter`）和一个汇总 `index.ts`（`packages/ontology-generator/src/generator.ts:28-118`）。它生成的是**类型骨架**，不是种子数据，也不是控制台运行时类型——控制台读的是服务返回的 JSON，从不 import 这些产物。

> 文档与实现不一致：`apps/app-docs` 的 OntologyGuide 页面把 maker DSL 写成链式 API（`defineObject("Employee").description(...).primaryKey("employeeId", "STRING")...build()`），并称它「产生中间表示 OntologyIr」（`apps/app-docs/src/pages/OntologyGuide.tsx:136-155`）。真实 API 是 `defineObject(apiName, builder)` 回调式，`primaryKey` 只收一个参数，builder 上没有公开的 `build()`，且产物是 `OntologyDefinition` 而非 IR（`packages/ontology-maker/src/define-object.ts:112`、`packages/ontology-maker/src/define-ontology.ts:75`）。该页的示例无法编译。同一页还声称平台会「Validate all writes against the declared schema」「Generate strongly-typed TypeScript SDKs」并「Build query plans for the ObjectSet query engine」（`apps/app-docs/src/pages/OntologyGuide.tsx:23-25`）：写入校验并不存在，SDK 生成器存在但没有任何调用方，ObjectSet 是在内存里递归求值而不是生成查询计划。

IR 这一层还有个容易忽略的性质：**往返转换是有损的**。`LinkTypeIr` 只有 `objectTypeA`/`objectTypeB`/`cardinality`/`description`，没有外键属性名（`packages/ontology-ir/src/ontology-ir.ts:110-116`），所以 `irToOntology` 只能按约定猜一个 `${objectTypeB}Id`（`packages/ontology-ir/src/ir-converter.ts:276-284`）；`ObjectTypeIr` 没有 `description`，回填时被写成 `Object type: ${apiName}`（`packages/ontology-ir/src/ir-converter.ts:221-230`）；Ontology 的 `rid` 则由 apiName 现编（`packages/ontology-ir/src/ir-converter.ts:366`）。IR 适合「编译一次、单向消费」，不适合当持久化格式。

### 6.6 svc-ontology：元数据 CRUD、别名与 v1/v2 投影

服务启动时按 `DATABASE_URL` 在两种存储间二选一，然后把同一个 store 注册给 v2 与 v1 两套路由（`services/svc-ontology/src/server.ts:39-50`、`services/svc-ontology/src/server.ts:135-136`）。注意最后那两行的 `store as OntologyStore`：`PgOntologyStore` 的方法全是 `Promise`，而路由的 `opts` 类型声明为同步的 `OntologyStore`，这里用类型断言把差异压掉了——与 `svc-objects` 用联合接口的做法不同（对比见下一节）。

#### 6.6.1 路由表

v2 是主要面，v1 只覆盖元数据读操作。两套分别挂在 `/api/v2` 与 `/api/v1` 前缀下。

| 方法 | v2 路径（前缀 `/api/v2`） | 说明 |
|------|---------------------------|------|
| GET/POST | `/ontologies` | 列表带 `pageSize`/`pageToken`；创建后授予创建者 Owner |
| GET/DELETE | `/ontologies/:ontologyRid` | 路径参数可以是 RID 或 apiName |
| GET | `/ontologies/:ontologyRid/fullMetadata` | 一次性返回本体与全部类型集合 |
| GET/POST | `/ontologies/:rid/objectTypes` | 列表支持 `includeDatasources` 开关 |
| GET/PUT/DELETE | `/ontologies/:rid/objectTypes/:apiName` | 按 apiName 读写 |
| GET/POST | `/ontologies/:rid/linkTypes` | 全量链接类型 |
| GET | `/ontologies/:rid/objectTypes/:apiName/linkTypes` | 双向 |
| GET | `/ontologies/:rid/objectTypes/:apiName/outgoingLinkTypes` | 只取源侧，返回 `LinkTypeSideV2` |
| GET/DELETE | `/ontologies/:rid/linkTypes/:apiName` | — |
| GET/POST/PUT/DELETE | `/ontologies/:rid/actionTypes[/:apiName]` | — |
| GET/POST/DELETE | `/ontologies/:rid/interfaceTypes[/:apiName]` | — |
| GET/POST/DELETE | `/ontologies/:rid/sharedPropertyTypes[/:apiName]` | v1 没有这一族 |
| GET | `/ontologies/:rid/queryTypes` | 恒返回空列表（`services/svc-ontology/src/routes/v2/query-types.ts:14-20`） |

v1 只有八个只读端点：本体列表与详情、对象类型列表与详情、出站链接类型、动作类型列表与详情、查询类型列表（`services/svc-ontology/src/routes/v1/ontologies.ts:63-184`）。这个数量差异本身就是设计选择：v1 只提供读操作，本仓库没有为这些资源的写操作实现 v1 模型。

#### 6.6.2 存储

内存存储是 `Map<rid, StoredOntology>`，每个 `StoredOntology` 内部再各持五个 `Map<apiName, Definition>`，另有一个 schema 里没有的 `sharedPropertyTypes`（`services/svc-ontology/src/store/ontology-store.ts:30-42`）。所有写方法结尾都调 `saveToDisk()`，后者把 Map 展开成普通对象、写临时文件再 `rename`，异常被吞掉（`services/svc-ontology/src/store/ontology-store.ts:73-102`）。与 `svc-objects` 不同，这个文件**没有** `formatVersion`，也没有按 ontology 分区的概念——它本来就是一个「本体表」。

Postgres 实现有 925 行，但结构是机械的：每个实体的行类型、`rowTo*` 转换、CRUD。两处值得记住：

- `list*` 方法默认 `pageSize = 100` 且返回 `{ items, total }`，排序按 `api_name`；`listOntologies` 却按 `created_at`（`services/svc-ontology/src/store/pg-ontology-store.ts:207-226`）。这正是 AGENTS.md 警告的「不要按 createdAt 排序」：`listOntologies` 只有 `created_at`，没有主键兜底。
- `getOntology` 走 `hydrateOntology`，后者对六张子表各发一条查询再拼回 `StoredOntology`（`services/svc-ontology/src/store/pg-ontology-store.ts:843-899`）。单本体读取因此是 1 + 6 次查询，且 `version` 被硬编码为 1。

#### 6.6.3 别名解析

「本体可以用 apiName 代替 RID 出现在路径里」这件事在三个地方各实现了一遍：

1. **鉴权层**：`registerAuthorization` 的 `alias` 回调把非 `ri.` 开头的标识符按 apiName 查成 RID，返回 `undefined` 表示不是别名（`services/svc-ontology/src/server.ts:123-132`）。`requirePermission` 会把 `[别名, RID, ROOT]` 一起放进资源链，这样挂在 RID 上的 Marking 也能约束用别名访问的调用方（`packages/permissions/src/middleware.ts:87-96`）。
2. **v2 路由层**：`GET /ontologies/:ontologyRid` 在参数不以 `ri.` 开头时先按 apiName 找一遍（`services/svc-ontology/src/routes/v2/ontologies.ts:180-186`）。
3. **v1 路由层**：v1 的每个路由都先过 `resolveOntology`，逻辑与 v2 相同（`services/svc-ontology/src/routes/v1/ontologies.ts:40-48`）。

`svc-objects` 完全没有这一层：它注册授权时不传 `alias`（`services/svc-objects/src/server.ts:128-130`），路由直接把路径里的字符串当存储键用。因此用 apiName 代替 RID 读取对象列表不会报错，只会返回空列表（单个对象查询则 404）——见〈已知限制〉。

#### 6.6.4 v1/v2 投影差异

v2 的元数据路由基本是「把 store 里的对象原样返回」（对象类型、链接类型、动作类型、接口类型都是），v1 则有一整套 `serializers.ts` 做显式投影。差异不是前缀，是模型：

| 模型 | v1 | v2（OpenFoundry 实际输出） | 证据 |
|------|----|---------------------------|------|
| Ontology | 四字段，无 `version` | 多一个 `version`，恒为 1 | `services/svc-ontology/src/routes/v1/serializers.ts:100-107`、`services/svc-ontology/src/routes/v2/ontologies.ts:9-17` |
| ObjectType.primaryKey | `string[]`（复合键语义，长度为 1） | 原样回显请求体；HTTP 创建的类型带的是单数 `primaryKey` | `services/svc-ontology/src/routes/v1/serializers.ts:139-171` |
| Property | `{ baseType }` 扁平字符串 | 原样返回请求体里的 `type` | `services/svc-ontology/src/routes/v1/serializers.ts:109-114` |
| ActionType | `rid` + `operations[]` 必需 | 无 `rid`、无 `operations` | `services/svc-ontology/src/routes/v1/serializers.ts:181-229` |
| LinkTypeSide | 与 v2 同形但可选字段省略 | 字段恒在 | `services/svc-ontology/src/routes/v1/serializers.ts:236-247`、`services/svc-ontology/src/routes/v2/link-types.ts:16-24` |

v1 需要的 `rid` 是现算的：两个 store 都不记录对象类型/动作类型的 RID，序列化器用 `generateDeterministicRid("ontology", type, ontologyRid + "/" + apiName)` 做 SHA-256 派生（`services/svc-ontology/src/routes/v1/serializers.ts:88-94`）。这样重启和换后端都不变，代价是 RID 不可反解、也无法与 Postgres 里的行 RID 对应。

### 6.7 svc-objects：对象与链接的数据面

`svc-objects` 的启动逻辑与 `svc-ontology` 对称：`DATABASE_URL` 决定内存还是 Pg，`LinkStore` 永远是内存实现（没有 Pg 链接存储），然后分别注册 v2 的 objects/objectSets/links/import/semanticSearch 与 v1 路由（`services/svc-objects/src/server.ts:50-63`、`services/svc-objects/src/server.ts:128-161`）。

#### 6.7.1 路由表

| 方法 | 路径（前缀 `/api/v2`） | 说明 |
|------|------------------------|------|
| GET | `/ontologies/:rid/objects/:type` | 列表；`select`/`orderBy`/`excludeRid`/`snapshot` |
| GET | `/ontologies/:rid/objects/:type/:pk` | 单对象 |
| POST | `/ontologies/:rid/objects/:type` | 创建；body 里 `upsert: true` 时改走 upsert |
| PUT/DELETE | `/ontologies/:rid/objects/:type/:pk` | 更新 / 删除 |
| POST | `/ontologies/:rid/objects/:type/search` | `SearchJsonQueryV2` |
| POST | `/ontologies/:rid/objects/:type/aggregate` | 单类型聚合 |
| GET/POST | `/ontologies/:rid/objects/:type/:pk/links/:linkType` | 链接目标对象列表 / 建链接 |
| DELETE | `.../links/:linkType/:targetPrimaryKey` | 删链接 |
| POST | `/ontologies/:rid/objectSets/loadObjects` | ObjectSet 查询主入口 |
| POST | `/ontologies/:rid/objectSets/aggregate` | ObjectSet 聚合 |
| POST | `/ontologies/:rid/objectSets/search` | ObjectSet 内全文搜索 |
| POST | `/ontologies/:rid/import/csv` | CSV 批量导入 |
| POST | `/ontologies/:rid/objects/semantic-search` | 代理到 svc-aip |

v1 只有五个：对象列表、单对象、链接对象列表、`search`、`aggregate`（`services/svc-objects/src/routes/v1/objects.ts:203-498`）。v1 的列表用 `properties=` 而不是 `select=`，`search` 的过滤字段叫 `query` 而不是 `where`，`orderBy` 嵌在 `{ fields: [...] }` 里。

#### 6.7.2 存储抽象：一个接口，两种实现

对象数据的核心是一个「可能同步、可能异步」的接口。它的注释把设计意图写得很清楚：

```ts
// services/svc-objects/src/store/object-store.ts:81（节选，省略参数与方法）
export interface ObjectReadWriteStore {
  allObjects(ontologyRid: string, objectType: string): StoredObject[] | Promise<StoredObject[]>;
  listObjects(ontologyRid: string, objectType: string, options?: ListOptions): ListResult | Promise<ListResult>;
  getObject(ontologyRid: string, objectType: string, primaryKey: string): StoredObject | Promise<StoredObject>;
  createObject(ontologyRid: string, objectType: string, primaryKey: string, properties: Record<string, unknown>): StoredObject | Promise<StoredObject>;
  upsertObject?: (/* … */) => StoredObject | Promise<StoredObject>;      // 可选能力
  propertyNames?: (ontologyRid: string, objectType: string) => ReadonlySet<string> | undefined | Promise<ReadonlySet<string> | undefined>;
  // 其余：getObjectsByKeys / updateObject / deleteObject / linkTargetObjectType
}
```

三个可选成员（`upsertObject`、`propertyNames`、`linkTargetObjectType`）是两种实现能力差异的出口：内存 `ObjectStore` 不持有类型声明，所以没有后两个；`PgObjectStore` 全部实现。接口把每个数据方法的**第一个参数固定为 `ontologyRid`**，理由写在接口注释里：对象类型的 apiName 只在一个本体里唯一（`services/svc-objects/src/store/object-store.ts:75-80`）。

| 维度 | 内存 `ObjectStore` / `LinkStore` | `PgObjectStore` |
|------|----------------------------------|-----------------|
| 同步性 | 同步方法，直接返回数据 | 全异步，返回 `Promise` |
| 存储结构 | `Map<ontologyRid, Map<objectType, Map<pk, obj>>>`；链接是 `Map<ontologyRid, Map<复合键, StoredLink[]>>` | `objects` 表，按 `object_type_rid` 外键归属类型行；链接无 Pg 实现 |
| 类型归属 | 无类型行，apiName 直接当键 | `resolveObjectTypeRid(ontologyRid, apiName)` 先查 `object_types`，查不到 404（`services/svc-objects/src/store/pg-object-store.ts:545-557`） |
| 持久化 | `{ formatVersion: 2, ontologies: {...} }`，原子写 | 数据库事务，可选 `org_rid` 租户列 |
| 顺序 | Map/数组插入序，精确可复现 | `ORDER BY created_at ASC, primary_key ASC`（`services/svc-objects/src/store/pg-object-store.ts:288`、`services/svc-objects/src/store/pg-object-store.ts:399`） |
| 分页 | 全量数组 `slice`，`totalCount` 是数组长度 | `COUNT(*)` + `LIMIT/OFFSET`（`services/svc-objects/src/store/pg-object-store.ts:269-302`） |
| 属性名校验 | 无 `propertyNames`，因此不校验 | 读 `object_types.properties` 的键集合（`services/svc-objects/src/store/pg-object-store.ts:319-336`） |
| 冲突 | `Map.has` → 409 | 唯一约束 23505 → 409（`services/svc-objects/src/store/pg-object-store.ts:564-570`） |
| 组织隔离 | 无 | 构造时传 `orgRid` 才写 `org_rid` 并 `set_config('app.org_rid')`（`services/svc-objects/src/store/pg-object-store.ts:167-183`、`services/svc-objects/src/store/pg-object-store.ts:517-533`） |

最后一行有个部署细节：`createServer` 用 `new PgObjectStore(pgPool)` 构造，**没有传 orgRid**（`services/svc-objects/src/server.ts:57`），所以生产路径上 `withTenant` 不会设置 `app.org_rid`，插入依赖列默认值 `org:default`。测试里显式构造 `new PgObjectStore(pool, "org:acme")` 才验证租户写入（`services/svc-objects/tests/ontology-scoping.test.ts:536-556`）。

#### 6.7.3 落盘格式与 `*.unscoped-<ms>` 迁移

内存存储的持久化格式在 `services/svc-objects/src/store/ontology-scoped-file.ts` 里统一定义，两个 store 共用：

```json
{
  "formatVersion": 2,
  "ontologies": {
    "ri.ontology.main.ontology.<uuid>": {
      "Employee": {
        "emp-1": {
          "rid": "ri.phonograph2-objects.main.object.<uuid>", "objectType": "Employee",
          "primaryKey": "emp-1", "properties": { "name": "Carol" },
          "createdAt": "2026-01-01T00:00:00.000Z", "updatedAt": "2026-01-01T00:00:00.000Z"
        }
      }
    }
  }
}
```

加载时如果 `formatVersion !== 2` 且文件非空，就把整个文件 `rename` 成 `<file>.unscoped-<epoch ms>`，在 stderr 打一条告警，然后以空存储启动（`services/svc-objects/src/store/ontology-scoped-file.ts:45-77`）。理由在文件头注释里：旧格式的记录不携带 ontology，既不能安全地分给某个本体，也不能复制给所有本体，只能保留原文件、不提供服务。`start.sh` 会在启动服务前主动检查 `object-store.json` 的 `formatVersion`，不是 2 就删掉三个 store 文件再重新播种（`start.sh:104-114`）。

> 文档与实现不一致：`services/svc-objects/src/store/ontology-scoped-file.ts:16` 的注释说旧文件会被移走、由「demo seeds 重新填充（`start.sh --reseed`）」，但 `start.sh` 实际做的是**删除** `ontology-store.json`、`object-store.json`、`link-store.json` 三个文件，而不只是重新播种；并且它只检查 `object-store.json`，`link-store.json` 单独存在旧格式时不会被预清理（会被服务在加载时移走）。

#### 6.7.4 一次普通读取怎么走完

以 `GET /api/v2/ontologies/:rid/objects/Employee/emp-1?select=name` 为例：

1. `requirePermission("objects:read")` 从路径取 `ontologyRid` 组成资源链并评估权限（`packages/permissions/src/middleware.ts:98-124`）。
2. 处理器调 `rejectUnsupportedOntologyScoping`，`branch`/`scenarioRid`/`transactionId` 只要有一个非空就 404（`services/svc-objects/src/routes/v2/objects.ts:355`）。
3. `parseListParam`/`parseBooleanParam` 解析查询参数；非法布尔值直接 400 而不是当 false（`services/svc-objects/src/routes/v2/query-params.ts:33-43`）。
4. 取类型声明 `declaredProperties`：有注入的 `objectTypeSchema` 就用它，否则用 `store.propertyNames?.()`；都没有时为 `undefined`，`assertPropertiesExist` 直接放行（`services/svc-objects/src/routes/v2/query-params.ts:147-152`）。
5. `await store.getObject(...)`，内存实现同步返回，Pg 实现返回 promise——两条路径都因为 `await` 而正确。
6. `applySelectedProperties` 裁剪属性、`applyExcludeRid` 决定是否去掉 `rid`，然后原样返回 `StoredObject`（`services/svc-objects/src/routes/v2/objects.ts:366-371`）。

列表接口有个分支值得单独记：只有当请求带 `orderBy`、`snapshot=true` 或游标里已有 `snapshotSize` 时，才走「读全量 → 排序 → 切片」；否则直接交给 `store.listObjects`，在 Pg 下是一条 `LIMIT/OFFSET`，不会每页全表读（`services/svc-objects/src/routes/v2/objects.ts:292-341`）。这个分支是 v2 读路径里唯一真正的下推。

#### 6.7.5 链接

链接存储用复合键 `sourceObjectType:sourcePrimaryKey:linkType` 索引一个数组（`services/svc-objects/src/store/link-store.ts:73-85`）。`createLink` 对相同目标去重并直接返回已有记录（幂等），`deleteLink` 按目标主键 `splice`，数组空了就删键（`services/svc-objects/src/store/link-store.ts:87-126`、`services/svc-objects/src/store/link-store.ts:172-196`）。

链接对象列表接口把「链接记录」解析成「目标对象」再分页：先按目标类型分组批量 `getObjectsByKeys`，再按链接顺序重组，目标已删除的链接自然消失（`services/svc-objects/src/routes/v2/links.ts:137-200`）。批量而非逐条读是刻意的，测试里用 40 条链接断言只发生两次批量读（`services/svc-objects/tests/async-object-store.test.ts:243`）。

#### 6.7.6 嵌入与语义搜索

`services/svc-objects/src/hooks/embedding-hook.ts` 定义了一个 fire-and-forget 的 `createEmbeddingHook`：对象 upsert 时把属性拼成 `"ObjectType: k=v, ..."` 发给 svc-aip 的 `/api/v2/aip/embeddings`，失败只打 stderr（`services/svc-objects/src/hooks/embedding-hook.ts:61-98`）。但它**没有被任何地方调用**——全仓库只有定义本身和一行注释引用它。因此默认栈里对象不会自动产生向量；`POST /objects/semantic-search` 仍然可用，它只是把请求代理给 svc-aip，由 AIP 侧决定有没有可检索的向量（`services/svc-objects/src/routes/v2/semantic-search.ts:56-90`）。

### 6.8 ObjectSet 求值

#### 6.8.1 类型模型：声明 17 种，路由实现 7 种

`packages/object-set/src/object-set-types.ts` 的判别联合声明了 17 种 ObjectSet（`packages/object-set/src/object-set-types.ts:7-24`），但 `svc-objects` 的路由求值器只处理其中 7 种：`BASE`、`FILTER`、`UNION`、`INTERSECT`、`SUBTRACT`、`STATIC`、`SEARCH_AROUND`（`services/svc-objects/src/routes/v2/object-sets.ts:44-126`）。其余 10 种（`REFERENCE`、`INTERFACE_BASE`、`NEAR`、`WITHIN`、`INTERSECTS_BBOX`、`INTERSECTS_POLYGON`、`ALL_GEO`、`NATURAL_LANGUAGE`、`DATAFRAME`、`VERSIONED`）落到 `default: return []`——返回空集而不是报错。README 说的「7 ObjectSet types」指的就是这 7 种（`README.md:93`）。

| 类型 | 语义 | 路由求值器 | 包内 `evaluate` | 复杂度（n = 集合规模） |
|------|------|-----------|----------------|----------------------|
| `BASE` | 某对象类型的全部对象 | ✅ `allObjects` | ✅ | O(n) |
| `FILTER` | 对子集再过滤 | ✅ `evaluateFilter` | ✅ | O(n) |
| `UNION` | 多集合并集，按 `rid` 去重 | ✅ | ✅ | O(Σn) |
| `INTERSECT` | 交集，以第一个集合为基准 | ✅ | ✅ | O(Σn) |
| `SUBTRACT` | 左集减右集 | ✅ | ✅ | O(n+m) |
| `STATIC` | 显式主键集合 | ✅ `getObjectsByKeys` | ✅（扫全量后过滤） | O(k) / O(n) |
| `SEARCH_AROUND` | 沿链接类型遍历到目标对象 | ✅ 逐源对象查链接 + 逐个 `getObject` | ❌（落 default） | O(源对象数 × 链接数) 次读 |
| 其余 10 种 | — | ❌ 返回 `[]` | ❌ 返回入参 | — |

`SEARCH_AROUND` 是唯一有 N+1 风险的形态：它没有像链接列表接口那样按目标类型批量解析，而是对每个源对象的每条链接调用一次 `store.getObject`，并把 404 静默跳过（`services/svc-objects/src/routes/v2/object-sets.ts:98-122`）。

#### 6.8.2 `loadObjects` 的完整流程

```mermaid
flowchart TD
  A[POST objectSets/loadObjects] --> B[rejectUnsupportedOntologyScoping]
  B --> C[assertStringListBody 校验 select 与 orderBy]
  C --> D[declaredForSet 解析叶子对象类型]
  D --> E[assertPropertiesExist 校验属性名]
  E --> F[resolveApiObjectSet 递归求值]
  F --> G[orderBy 排序]
  G --> H[totalCount 取排序后全量长度]
  H --> I[按 offset 与 pageSize 切片]
  I --> J[select 属性投影]
  J --> K[返回 data totalCount nextPageToken]
```

对应代码：`services/svc-objects/src/routes/v2/object-sets.ts:383-463`。几个必须知道的细节：

- **整个求值只在一个 ontology 作用域内**：每个叶子的 `allObjects` / `getObjectsByKeys` / `getObject` 都以路径里的 `ontologyRid` 为首参，两个本体声明同名类型也不会互相取数（`services/svc-objects/src/store/object-store.ts:75-80`）。
- **求值发生在分页之前**，且 `totalCount` 是求值后的全量长度，不是页内长度（`services/svc-objects/src/routes/v2/object-sets.ts:426`）。
- **排序也在分页之前**，`orderBy` 逐字段比较，`null`/`undefined` 在两个方向上都排最后（`services/svc-objects/src/routes/v2/object-sets.ts:409-424`）。
- **投影在切片之后**，所以 `select` 不会影响 `totalCount`，也不会影响排序字段的可用性。
- **`executeInMemoryOnly` 只被校验、不被使用**：路由本来就完全在内存里求值，这个参数只能因为不是 `true`/`false` 而报 400（`services/svc-objects/src/routes/v2/object-sets.ts:388-392`）。
- **返回体是 `StoredObject`**，包含 `objectType`、`primaryKey`、`createdAt`、`updatedAt`，而不是 Foundry v2 的裸属性 map。

#### 6.8.3 两个求值器

仓库里实际存在两套 ObjectSet 求值逻辑：`packages/object-set` 的 `ObjectSetQueryEngine.evaluate`（`packages/object-set/src/query-engine.ts:76-161`）和 `svc-objects` 路由里的 `resolveApiObjectSet` / `resolveObjectSet`（`services/svc-objects/src/routes/v2/object-sets.ts:38-253`）。服务的路由**从不调用** `evaluate`；它们只用引擎的 `evaluateWhere`、`aggregate`、`search`。两套实现在未知类型上的行为正好相反：包内 `evaluate` 的 `default` 返回入参（等于「全部对象」），路由的 `default` 返回空数组。`evaluate` 只有单元测试在跑（`packages/object-set/tests/query-engine.test.ts:57`）。

### 6.9 查询与聚合

#### 6.9.1 `packages/search`：平台检索，不是对象过滤

`packages/search` 解决的不是对象集过滤，而是控制台顶部的全局搜索。`SearchEngine` 接口只有 `search` / `indexEntity` / `removeEntity` / `reindexAll` 四个方法（`packages/search/src/types.ts:57-63`），结果是 `{ rid, entityType, title, description, highlight, score }`（`packages/search/src/types.ts:17-28`），实体类型覆盖 `ONTOLOGY`、`OBJECT_TYPE`、`OBJECT`、`DATASET`、`RESOURCE`、`FUNCTION`、`ACTION`、`USER`、`GROUP` 九种（`packages/search/src/types.ts:5-15`）。`PgSearchEngine` 用 Postgres 的 `tsvector` / `plainto_tsquery` 查一张 `search_index` 表，并从 8 张源表重建索引（`packages/search/src/pg-search.ts:29-94`、`packages/search/src/pg-search.ts:111-179`）；`InMemorySearchEngine` 是测试用的子串打分实现，标题 3 分、描述 2 分、内容 1 分（`packages/search/src/in-memory-search.ts:97-111`）。它和 `SearchJsonQueryV2`、`ObjectSetQueryEngine` 没有调用关系，仓库里唯一的消费者是网关的 `/api/v2/search`（`services/svc-gateway/src/routes/v2/search.ts:3-4`）。

#### 6.9.2 `SearchJsonQueryV2` 算子

v2 的对象搜索接收 `where` 递归结构，路由里有一个显式转换函数把它折成查询引擎的 `WhereClause`（`services/svc-objects/src/routes/v2/objects.ts:138-209`）。**实际接受**的算子如下（大小写不敏感）：

| 算子 | 语义 | 作用的值类型 | v1 对应 |
|------|------|--------------|---------|
| `eq` / `neq` | 严格相等 / 不等 | 任意 JSON 值 | `eq`（无 `neq`） |
| `gt` `gte` `lt` `lte` | 比较；数字按数值、其余按字符串 | number / string | 同名 |
| `isNull` | 值为 `null` 或缺失 | 任意 | `isNull`（必须带布尔 value） |
| `contains` | 字符串子串包含 | string | `contains` 是**数组成员**，`phrase` 才是子串 |
| `startsWith` | 字符串前缀 | string | `prefix` |
| `in` | 值在给定数组中 | string / number / boolean | 无 |
| `and` / `or` / `not` | 逻辑组合，子句放 `value` 数组或单值 | 任意 | 同名 |
| `containsAnyTerm` | 按空白切词，命中任一即真 | string | `anyTerm` |
| `containsAllTerms` | 按空白切词，全部命中才真 | string | `allTerms` |

合计 15 个名字，其中路由注释只列了 13 个（漏了 `neq` 与 `in`，`services/svc-objects/src/routes/v2/objects.ts:445-447`）。**未知算子不会报错**：`searchFilterToWhere` 原样透传类型名，`evaluateWhere` 的 `default` 返回 `true`（`packages/object-set/src/query-engine.ts:236-238`），于是「拼错一个算子名」等价于「匹配全部对象」。

> 文档与实现不一致：`README.md:92` 把 `SearchJsonQueryV2` 的 13 个算子写成 `eq, gt, lt, gte, lte, isNull, contains, startsWith, and, or, not, prefix, anyTerm`。其中 `prefix` 与 `anyTerm` 是 v1 语法，v2 不接受（会走上面的静默全匹配）；v2 真正实现的 `neq`、`in`、`containsAnyTerm`、`containsAllTerms` 反而没列。

#### 6.9.3 v1 的独立实现与 `contains` 语义

v1 没有复用这套转换，而是自己实现了一个 14 种变体的求值器：`and`/`or`/`not` + `eq`/`isNull`/`contains`/`prefix`/`phrase`/`allTerms`/`anyTerm`/`lt`/`lte`/`gt`/`gte`（`services/svc-objects/src/routes/v1/search-query.ts:95-172`）。文件头把原因写得很直接：两版的算子名和语义都不同，v1 `contains` 测数组包含，共享引擎的 `contains` 测子串；若把 v1 翻译到引擎词汇表，`contains` 会答错（`services/svc-objects/src/routes/v1/search-query.ts:4-11`）。三个实现点的对照：

| 实现 | `contains` 语义 | 证据 |
|------|-----------------|------|
| v1 `services/svc-objects/src/routes/v1/search-query.ts` | `Array.isArray(actual) && actual.includes(value)` | `services/svc-objects/src/routes/v1/search-query.ts:136-138` |
| 共享引擎 `evaluateWhere` / `evaluateCanonicalFilter` | `typeof val === "string" && val.includes(...)` | `packages/object-set/src/query-engine.ts:191-194`、`packages/object-set/src/query-engine.ts:260-263` |
| Pg 过滤器 SQL 翻译 | `properties->>... ILIKE '%v%'` | `services/svc-objects/src/store/pg-object-store.ts:83-84` |

另外，v1 拒绝 `fuzzy: true` 而不是忽略它（`services/svc-objects/src/routes/v1/search-query.ts:113-118`）；未知过滤器类型抛 400 而不是静默匹配（`services/svc-objects/src/routes/v1/search-query.ts:171`）。这两点比 v2 严格。

#### 6.9.4 聚合：声明 8 种，v1 暴露 6 种

`packages/object-set/src/aggregations.ts` 声明了 8 种聚合（`packages/object-set/src/aggregations.ts:5-13`），查询引擎实际计算 7 种（`exact_distinct` 与 `approximate_distinct` 共用一个基于 `Set` 的分支，`packages/object-set/src/query-engine.ts:378-388`），v1 白名单只放 6 种（`services/svc-objects/src/routes/v1/objects.ts:45-52`）。

| metric | 语义 | 边界行为 |
|--------|------|----------|
| `count` | 对象条数 | 不需要 field |
| `sum` | 数值字段求和 | 非 number 值被过滤掉，空集得 0 |
| `avg` | 数值字段均值 | 空集得 0（不是 null） |
| `min` / `max` | 数值极值 | 空集得 0；`Math.min(...nums)` 对超大数组有参数上限风险 |
| `approximateDistinct` | 去重计数 | 实为精确 `Set`，没有 HyperLogLog |
| `exactDistinct` | 去重计数 | 引擎支持，v1 白名单没有 |
| `approximatePercentile` | 分位数 | 声明了但**没有实现**，落 `default` 返回 0 |

v2 的聚合路由接受任意 `type` 字符串，未识别的类型静默返回 0（`services/svc-objects/src/routes/v2/objects.ts:607-627`）；v1 的聚合路由则先白名单校验、再对非 `count` 的 metric 强制要求 `field`，`groupBy` 只允许 `exact`（`services/svc-objects/src/routes/v1/objects.ts:409-444`）。v1 还处理了一个引擎缺陷：引擎用 `name` 做 metric 键，省略 `name` 且只有 `field` 不同的两个聚合会互相覆盖，所以 v1 用请求下标当键再映射回名字（`services/svc-objects/src/routes/v1/objects.ts:468-494`）。

#### 6.9.5 分页与排序

分页游标是 base64url 编码的 JSON `{ offset, lastId?, sortValues?, snapshotSize? }`（`packages/pagination/src/page-token.ts:10-48`），默认页大小 100、上限 10000（`packages/pagination/src/page-request.ts:4-5`）。三个现实约束：

1. **只有 offset 在用**。`lastId` 与 `sortValues` 声明了、也参与编解码，但没有任何服务写它们，所以分页是纯偏移量语义——翻页期间插入新记录会让后续页错位。
2. **`snapshot=true` 用「冻结长度」而不是时间戳**。第一页把当前长度写进游标，后续页只读前 N 条（`services/svc-objects/src/routes/v2/query-params.ts:250-273`）。注释明说用计数而非时间戳，因为同一毫秒可能写入多条。
3. **各端点的 token 形态不统一**：对象与链接列表用 base64 游标，链接存储在内部用裸数字偏移（`services/svc-objects/src/store/link-store.ts:140-153`），`objectSets/search` 直接返回 `String(offset + len)` 并用 `parseInt` 读回（`services/svc-objects/src/routes/v2/object-sets.ts:553-558`、`packages/object-set/src/query-engine.ts:442`）。v2 链接路由两种都接受（`services/svc-objects/src/routes/v2/links.ts:56-60`）。

### 6.10 设计决策与理由

**1. 对象与链接按 ontology 作用域存储。** 对象类型的 apiName 只在一个本体里唯一，两个本体都可以声明 `Employee`；早期实现按 apiName 全局解析，导致一个本体的路径能读写另一个本体的数据。修复的代价是每个 store 方法多一个首参、内存文件格式升到 v2、旧文件必须弃用、Pg 每次读写多一次 `resolveObjectTypeRid` 查询（`services/svc-objects/src/store/pg-object-store.ts:545-557`）。取舍是**用一次额外的键查找换正确性**，并且把作用域放在接口签名里而不是靠调用方自觉。

**2. 内存 store 同步、Pg store 异步，用 `T | Promise<T>` 统一。** 这样内存实现简单、测试不需要 async 样板，Pg 实现不被同步接口限制。代价是类型系统无法强制调用方 `await`：漏掉 `await` 在内存模式下完全正确，在 Pg 模式下会把 pending promise 当 200 返回。团队用 `AsyncObjectStore` 测试替身专门复现这个不对称（`services/svc-objects/tests/async-object-store.test.ts:1-10`、`services/svc-objects/tests/async-object-store.test.ts:170`）。

**3. 一个 `ObjectReadWriteStore` 接口 + 一套路由，而不是两套路由。** 路由注册在接口类型上，两种后端零改动切换，`services/svc-objects/src/server.ts` 里没有任何类型断言。代价是接口必须为能力差异留可选成员（`upsertObject?`、`propertyNames?`），并且把同步/异步的分辨责任推给每个 handler。`svc-ontology` 走了另一条路——把 Pg store 断言成同步类型（`services/svc-ontology/src/server.ts:135-136`）——于是它必须逐处记得 `await`，而五条 DELETE 路由没有（见〈已知限制〉）。

**4. ObjectSet 递归求值，而不是编译成查询计划。** 递归实现直接对应 wire 格式的嵌套结构，代码短、易读、易测（`services/svc-objects/src/routes/v2/object-sets.ts:38-127`）。代价是没有下推：每次求值从叶子 `allObjects` 全量拉取，`UNION`/`INTERSECT` 在内存里做集合运算，`totalCount` 需要全量长度，所以**内存占用与集合基数成正比**。在 Pg 模式下这些全量读意味着每个 ObjectSet 查询至少把整个对象类型读进 Node 进程。

**5. 聚合在服务端做。** 客户端只发 metric 列表，服务端把对象全量取出后计算。好处是客户端不必分页拉全量、聚合结果与过滤条件天然一致。代价是聚合的内存成本与对象数成正比，且因为聚合在内存做，Pg 的 `GROUP BY` 能力完全没用上；`approximateDistinct` 也退化成精确去重。

**6. 用 RID 而不是 UUID。** RID 是 `ri.<service>.<instance>.<type>.<locator>` 的五段式字符串（`packages/rid/src/rid.ts:8-12`），可读、可分段校验、能表达「哪个服务的哪种资源」，并且 `generateDeterministicRid` 让 v1 那些「store 里没有 RID」的实体也能有稳定标识（`services/svc-ontology/src/routes/v1/serializers.ts:88-94`）。代价是 RID 是纯字符串、没有中心注册表、可以伪造；对象 RID 用随机 UUID locator（`packages/rid/src/rid-generator.ts:13-21`），确定性 RID 用 SHA-256，两者混用意味着「RID 相同 ⇒ 同一资源」并不总成立。

### 6.11 已知限制与坑

| 限制 | 触发条件 | 影响 | 证据 |
|------|----------|------|------|
| `createdAt` 毫秒碰撞 | 同一毫秒内创建多条记录 | 不能用作排序键或分页边界；内存 store 用插入序规避，Pg 用 `created_at` + 主键兜底，但 `queryObjects` 只有 `created_at` | `AGENTS.md:70-72`、`services/svc-objects/src/store/pg-object-store.ts:288`、`services/svc-objects/src/store/pg-object-store.ts:399`、`services/svc-objects/src/store/pg-object-store.ts:474` |
| 测试非幂等 + 固定数据目录 | 对运行中的网关重复跑集成测试 | 第二次运行 409、计数膨胀；`/tmp/openfoundry-data` 无环境变量覆盖，两个 checkout 互相覆盖 | `AGENTS.md:61-66`、`AGENTS.md:136`；默认路径见 `services/svc-objects/src/store/object-store.ts:308`、`services/svc-objects/src/store/link-store.ts:44`、`services/svc-ontology/src/store/ontology-store.ts:64` |
| v1/v2 `contains` 语义不同 | 同一个词在两版搜索里 | v1 找数组成员，v2 找子串；跨版本迁移查询会得到不同结果 | `services/svc-objects/src/routes/v1/search-query.ts:136-138` vs `packages/object-set/src/query-engine.ts:260-263` |
| 无本体分支/场景/事务 | 请求带 `branch`/`scenarioRid`/`transactionId` | 一律 404；仓库里所有 `branch` 都是数据集分支 | `packages/errors/src/ontology-scoping.ts:1-27`、`packages/errors/src/ontology-scoping.ts:108-120` |
| 分页的内存成本 | 大对象集 + `loadObjects`/`aggregate`/`orderBy` | 全量对象进内存，`totalCount` 需要全量；偏移分页在并发写入下会错位 | `services/svc-objects/src/routes/v2/object-sets.ts:406-437`、`services/svc-objects/src/routes/v2/objects.ts:310` |
| 未知算子/类型静默放行 | 拼错 filter 类型或 ObjectSet 类型 | v2 搜索匹配全部对象；ObjectSet 未知类型返回空集；包引擎未知类型返回全部 | `packages/object-set/src/query-engine.ts:236-238`、`packages/object-set/src/query-engine.ts:158-159`、`services/svc-objects/src/routes/v2/object-sets.ts:124-125` |
| 属性名校验只在 Pg 生效 | 内存模式且未注入 `objectTypeSchema` | `select=nmae`/`orderBy=p.nmae` 返回 200，客户端以为请求被接受 | `services/svc-objects/src/routes/v2/query-params.ts:147-152`、`services/svc-objects/src/server.ts:39`、`services/svc-objects/src/server.ts:136` |
| 元数据写入零校验 | `POST/PUT` 对象类型、链接类型、动作类型 | 请求体原样存进 store；v2 返回的字段取决于客户端发了什么，v1 序列化器需要 `primaryKey`/`primaryKeyApiName` 双拼写兜底 | `services/svc-ontology/src/routes/v2/object-types.ts:60-72`、`services/svc-ontology/src/routes/v1/serializers.ts:116-137` |
| v2 对象返回形状与 SDK 类型不符 | 任何对象读 | 服务返回 `StoredObject`（含 `properties`），SDK 的 `OntologyObject` 声明的是 `__apiName`/`__primaryKey`/`__rid`；控制台按 `properties` 读，所以实际能用的是前者 | `services/svc-objects/src/routes/v2/objects.ts:366-371`、`packages/sdk/sdk/src/types.ts:82-87`、`apps/app-console/src/pages/ObjectExplorer.tsx:56` |
| `svc-objects` 不解析本体别名 | 用 apiName 代替 RID 读取对象列表 | 返回空列表而不是解析或 404（字符串被当存储键）；单对象查询仍会 404 | `services/svc-objects/src/server.ts:128-130`、`services/svc-objects/src/store/object-store.ts:346-351` |
| Pg 元数据列表上限 100 叠加路由分页 | Postgres 模式下本体/类型超过 100 个 | 路由在 store 已截断的 100 条上再分页，第 101 条起永远看不到；v1 列表没有分页参数，同样只能看到 100 条 | `services/svc-ontology/src/store/pg-ontology-store.ts:207-226`、`services/svc-ontology/src/store/pg-ontology-store.ts:296-318`、`services/svc-ontology/src/routes/v1/ontologies.ts:56-62` |
| 嵌入钩子未接线 | 期望「对象写入即向量化」 | 不会发生；语义搜索依赖 AIP 侧已有向量 | `services/svc-objects/src/hooks/embedding-hook.ts:61` |
| `queryObjects` 是死代码 | 期望过滤被下推到数据库 | 两个 store 都实现了它（含 `filterToSql` 的 `ILIKE` 翻译），但没有任何路由调用，过滤始终发生在 Node 内存里 | `services/svc-objects/src/store/pg-object-store.ts:446`、`services/svc-objects/src/store/object-store.ts:476` |

还有一条只在 Pg 模式出现的陷阱，值得从「坑」升级成警告：**漏 `await` 不会在任何内存测试里暴露**。`svc-ontology` 的六条 DELETE 路由都是 `store.deleteX(...)` 后直接 `reply.status(204)`（`services/svc-ontology/src/routes/v2/ontologies.ts:252`、`services/svc-ontology/src/routes/v2/object-types.ts:124`、`services/svc-ontology/src/routes/v2/link-types.ts:136`、`services/svc-ontology/src/routes/v2/action-types.ts:87`、`services/svc-ontology/src/routes/v2/interface-types.ts:69`、`services/svc-ontology/src/routes/v2/shared-property-types.ts:64`）。内存 store 同步完成，一切正常；Pg store 返回 promise，于是 204 先发出、删除在后台继续，失败也不会进入 Fastify 的错误处理器。这与 `svc-objects` 里读路由一律 `await`（或直接返回 promise，效果等价）的做法形成对照（`services/svc-objects/src/store/object-store.ts:66-74` 的注释就是为此写的）。

### 6.12 关键代码位置

1. `packages/ontology-schema/src/ontology.ts:19-46` —— `OntologyDefinition`，五类成员的 apiName 字典。
2. `packages/ontology-ir/src/ir-converter.ts:152-194` —— schema → IR 的编译入口。
3. `packages/ontology-ir/src/ir-converter.ts:276-284` —— 有损回填：外键属性名靠约定猜。
4. `packages/ontology-generator/src/generator.ts:28-118` —— 生成物清单（对象接口 / actions / client / index）。
5. `services/svc-ontology/src/server.ts:123-136` —— 别名回调与「Pg store 断言成同步类型」的注册点。
6. `services/svc-ontology/src/routes/v1/serializers.ts:88-171` —— v1 投影与确定性 RID 派生。
7. `services/svc-objects/src/store/object-store.ts:66-126` —— `ObjectReadWriteStore` 接口与「首个参数是 ontologyRid」的理由。
8. `services/svc-objects/src/store/ontology-scoped-file.ts:19-96` —— v2 落盘格式与 `*.unscoped-<ms>` 迁移。
9. `services/svc-objects/src/store/pg-object-store.ts:167-183` —— `withTenant`：事务内设置 `app.org_rid`，随提交/回滚复位。
10. `services/svc-objects/src/routes/v2/object-sets.ts:38-127` —— ObjectSet 递归求值器（7 种类型）。
11. `services/svc-objects/src/routes/v2/object-sets.ts:383-463` —— `loadObjects`：校验 → 求值 → 排序 → 分页 → 投影。
12. `packages/object-set/src/query-engine.ts:302-396` —— 聚合与分组实现（含 `approximate_distinct` 的精确退化）。

---

## 第 7 章 平台业务服务：动作、函数、自动化与协作

### 7.1 本章导读

本体（Ontology）定义了「世界上有哪些对象」，对象服务（svc-objects）保存了「对象现在长什么样」，本章要讲的是**让数据动起来的那一层**：动作（Actions）把一次用户点击翻译成多个对象的写入；函数（Functions）在不产生副作用的前提下做计算；Sentinel 在后台按时间或阈值触发副作用；Webhooks、Media、Compass 提供事件外发、文件托管与资源树；svc-admin 提供用户、组、审计与授权管理；`packages/pipeline` 提供 DAG 变换的计算内核。

这一层的共同特点是：**它们几乎不拥有数据本身**，而是通过 HTTP 调用 svc-objects 或共享同一个存储层来读写别人的数据。正因为如此，「这些服务之间到底怎么协作、有没有事务边界」是本章最需要回答的问题，也是设计上最脆弱的地方。

### 7.2 职责与边界

本章覆盖的七个服务与一个库，职责划分如下：

| 做什么 | 不做什么 |
|--------|----------|
| svc-actions：注册动作类型、校验参数、按 handler 执行多对象副作用、记录执行历史 | 不定义动作类型（定义在 svc-ontology），不实现对象存储与事务 |
| svc-functions：注册、更新、执行用户提交的函数代码 | 不提供沙箱进程、不限制资源、不访问对象数据 |
| svc-sentinel：monitor 的 CRUD、60 秒调度循环、五类副作用、通知收件箱 | 不订阅对象变更事件（`OBJECT_CHANGED` 触发器无驱动方） |
| svc-webhooks：webhook 注册、HMAC 签名投递、重试与投递日志 | 不产生事件，只消费外部 POST 进来的事件 |
| svc-media：媒体上传、下载、元数据、S3 或内存双后端 | 不与对象属性绑定（`MEDIA_REFERENCE` 只是字符串） |
| svc-compass：资源树（SPACE/PROJECT/FOLDER）与血缘边 | 不承载真实文件，不与 datasets 自动同步 |
| svc-admin：用户/组/审计/资源角色，双存储模式 | 不签发 token（那是 svc-multipass，见〈身份与权限〉章） |
| `packages/pipeline`：DAG 校验、拓扑排序、纯内存变换 | 不做 I/O、不写 dataset、不调度 |

与相邻章节的边界：动作类型的**定义**属于本体章（见〈本体与对象数据面〉章），管道的 **HTTP 面与 dataset 分支**属于数据章（见〈数据与 AI〉章），资源角色的**权限模型**属于身份章。本章只写实现细节与耦合方式。

### 7.3 服务与端口总表

端口逐个从各服务的 `config.ts` 核实，启动情况从 `start.sh` 核实：

| 服务 | 端口 | 职责 | 被 start.sh 启动 | 存储模式 |
|------|------|------|------------------|----------|
| svc-gateway | 8080 | 反向代理、鉴权、限流 | 是 | 无状态 |
| svc-ontology | 8081 | 本体定义 CRUD | 是 | 内存 / PG |
| svc-objects | 8082 | 对象与链接读写、ObjectSet | 是 | 内存 / PG |
| svc-actions | 8083 | 动作注册与执行 | 是 | 内存 / PG |
| svc-multipass | 8084 | OAuth2、JWT、用户目录 | 是 | 内存 / PG |
| svc-datasets | 8085 | dataset、文件、分支、管道 HTTP 面 | 是 | 内存 / PG |
| svc-compass | 8086 | 资源树与血缘 | **否** | 仅内存（PG 实现是死代码） |
| svc-admin | 8087 | 用户、组、审计、资源角色 | 是 | 内存 / PG（双模有缺陷） |
| svc-functions | 8088 | 函数注册与执行 | 是 | 内存 / PG |
| svc-webhooks | 8089 | webhook 注册与投递 | **否** | 内存 / PG |
| svc-media | 8090 | 媒体上传下载 | **否** | 内存 / S3 |
| svc-sentinel | 8091 | 监控、调度、副作用 | 是 | 内存 / PG |
| svc-aip | 8092 | LLM 与向量检索 | 是 | 内存 / PG |

启动脚本的 10 个服务名单在 `start.sh:118`，端口清理名单在 `start.sh:85`，启动后自检的 9 个端口在 `start.sh:131`。

> 文档与实现不一致：`README.md:90` 写「10 services running」，`README.md:305` 写「13 defined, 10 active」，而 `start.sh:118` 确实只启动 10 个。但 `README.md:96` 声称动作有「create, modify, **delete** objects」的真实副作用——`services/svc-actions/src/objects-client.ts` 只实现了 `getObject`/`createObject`/`updateObject` 三个方法，没有删除能力，四个 handler 也没有任何一处删除对象。

> 文档与实现不一致：`start.sh:85` 会杀掉 8086（svc-compass）上的进程，但它从不启动 svc-compass；而 `start.sh:95` 的强杀名单又漏掉了 8086。结果是「杀了一个不会启动的服务端口，却没有强杀它」。更要紧的是 8091（svc-sentinel）既不在清理名单也不在自检名单，见〈已知限制与坑〉。

### 7.4 代码结构

| 路径 | 作用 |
|------|------|
| `services/svc-actions/src/routes/v2/actions.ts` | v2 五个端点：apply / validate / applyBatch / executions |
| `services/svc-actions/src/routes/v1/actions.ts` | v1 三个端点，独立实现且正确 await 存储 |
| `services/svc-actions/src/validation.ts` | 参数类型校验，按 `PropertyType` 逐类型判断 |
| `services/svc-actions/src/handlers/` | 四个演示动作的副作用实现 |
| `services/svc-actions/src/objects-client.ts` | 调用 svc-objects 的 HTTP 客户端 |
| `services/svc-functions/src/executor.ts` | `new Function` 执行器与超时守卫 |
| `services/svc-sentinel/src/scheduler.ts` | 60 秒 tick 与 cron 匹配 |
| `services/svc-sentinel/src/effects/` | WEBHOOK / ACTION / NOTIFICATION / LOG / EMAIL 五类副作用 |
| `services/svc-webhooks/src/delivery.ts` | HMAC-SHA256 签名、三次指数退避重试 |
| `services/svc-media/src/store/s3-media-store.ts` | S3 后端实现 |
| `services/svc-compass/src/store/compass-store.ts` | 资源树（唯一被使用的 store） |
| `services/svc-admin/src/routes/v2/` | users / groups / audit / permissions / stats 五组路由 |
| `packages/pipeline/src/` | dag.ts、executor.ts、types.ts |

### 7.5 动作执行：svc-actions

#### 7.5.1 数据模型

动作类型是一个进程内注册表条目，包含参数定义、被修改实体清单和一个可选 handler：

```ts
// services/svc-actions/src/store/action-registry.ts:53
export interface RegisteredAction {
  readonly apiName: string;
  readonly displayName: string;
  readonly parameters: Map<string, ParameterDef>;   // 名称 → 类型 + 是否必填
  readonly modifiedEntities: Record<string, ModifiedEntity>; // 声明式元数据，不驱动执行
  readonly status: "ACTIVE" | "EXPERIMENTAL" | "DEPRECATED";
  handler?: ActionHandler;                          // 真正的副作用在这里
}
```

`modifiedEntities` 只用于对外展示动作会碰哪些对象类型，**执行路径完全不读它**；真正改什么由 handler 里的 HTTP 调用决定。这是一处声明与行为分离的设计。

注册表有两种后端：`ActionRegistry`（内存 Map）和 `PgActionRegistry`（表 `action_registry`）。后者把参数与实体清单写库，但 handler 仍留在进程内的 `handlers` Map 里，读取时再合并（`services/svc-actions/src/store/pg-action-registry.ts:117`）。因此 handler 是**代码，不是数据**，重启后必须靠 `registerPestControlActions` 重新挂载（`services/svc-actions/src/server.ts:135`）。

#### 7.5.2 端点表面

| 版本 | 方法 | 路径 | 权限 |
|------|------|------|------|
| v2 | POST | `/api/v2/ontologies/:ontologyRid/actions/:actionApiName/apply` | `actions:execute` |
| v2 | POST | `.../actions/:actionApiName/validate` | `actions:read` |
| v2 | POST | `.../actions/:actionApiName/applyBatch` | `actions:execute` |
| v2 | GET | `.../actions/:actionApiName/executions` | `actions:read` |
| v2 | GET | `.../executions/:executionRid` | `actions:read` |
| v1 | POST | `/api/v1/ontologies/:ontologyRid/actions/:actionType/apply` | `actions:execute` |
| v1 | POST | `.../actions/:actionType/applyBatch` | `actions:execute` |
| v1 | POST | `.../actions/:actionType/validate` | `actions:read` |

注意路径里**必须带 ontologyRid**：不存在 `/actions/:action/apply` 这种扁平形式。`services/svc-gateway/src/routes/v2/actions.ts` 虽然注册了 `/api/v2/actions/*` 前缀并把整条路径原样转发给 svc-actions，但 svc-actions 没有任何 `/api/v2/actions/...` 路由，所以该前缀下的一切请求都是 404。

#### 7.5.3 一次 apply 的完整时序

```mermaid
sequenceDiagram
    participant C as Console 客户端
    participant G as svc-gateway 8080
    participant A as svc-actions 8083
    participant L as ActionLog
    participant O as svc-objects 8082

    C->>G: POST /api/v2/ontologies/:rid/actions/:name/apply
    G->>G: 校验 JWT 并写入 x-user-* 身份头
    G->>A: 正则匹配 apply 后转发 8083
    A->>A: requirePermission actions:execute
    A->>A: 拒绝 branch 与 transactionId
    A->>A: registry.getAction 查注册表
    alt 动作未注册
        A-->>C: 404 Action not found
    end
    A->>A: validateActionParameters
    alt 参数不合法
        A-->>C: 400 INVALID_ARGUMENT
    end
    A->>L: logStart 生成 execution RID
    A->>O: GET objects ServiceJob jobId
    O-->>A: 对象记录
    A->>O: PUT objects ServiceJob jobId
    A->>O: GET 并 PUT objects Technician
    A->>O: POST objects Invoice
    alt 任一步失败
        A->>L: logFailure 记录错误
        A-->>C: 500 ActionExecutionFailed
    end
    A->>L: logComplete
    A-->>C: 200 rid status result
```

逐步骤说明：

1. **网关路由**。`services/svc-gateway/src/routes/v2/ontologies.ts:35` 用正则 `/\/ontologies\/[^/]+\/actions\/[^/]+\/(apply|validate)$/` 判断去向：匹配则转发 svc-actions，否则一律转发 svc-ontology。这个正则带 `$` 锚点，因此 **applyBatch 和 executions 不匹配**，会被转发到 svc-ontology，而 svc-ontology 没有这些路由，结果是 404。也就是说：applyBatch 与执行历史查询**无法经网关到达 svc-actions**。
2. **权限与作用域**。`requirePermission("actions:execute")` 在 `services/svc-actions/src/routes/v2/actions.ts:73`；未设置 `ENFORCE_PERMISSIONS=true` 时它是空操作（`packages/permissions/src/middleware.ts:111`）。随后 `rejectUnsupportedOntologyScoping` 拒绝 `branch`、`scenarioRid`、`transactionId`，因为本仓库没有本体分支或本体事务（`packages/errors/src/ontology-scoping.ts:108`）。
3. **查注册表**（`services/svc-actions/src/routes/v2/actions.ts:80`）。未注册即 404。**这一步在参数校验之前**，是「apply 返回 404 而不是 400」的直接原因。
4. **参数校验**（同文件 86 行）。失败返回 400，`parameters.validationErrors` 列出每条错误。
5. **记录开始**（98 行）。`ActionLog.logStart` 生成 `ri.actions.execution.*` 并置为 `RUNNING`（`services/svc-actions/src/store/action-log.ts:50`）。
6. **执行副作用**（103–109 行）。仅当 `action.handler` 存在时执行；**没有 handler 的动作会返回 200 SUCCEEDED 但什么也没做**。
7. **成功/失败收敛**。成功 `logComplete` 并返回 `{rid, status, result}`；异常被捕获后 `logFailure` 并返回 500 `ActionExecutionFailed`。

#### 7.5.4 副作用与存储耦合方式

svc-actions **不共享 svc-objects 的存储**，它通过 HTTP 调用对方（`services/svc-actions/src/objects-client.ts:43`）：

```
POST ${OBJECTS_SERVICE_URL}/api/v2/ontologies/${ontologyRid}/objects/${objectType}
```

`OBJECTS_SERVICE_URL` 默认 `http://localhost:8082`（`services/svc-actions/src/config.ts:45`）。每次调用独立超时 10 秒（`services/svc-actions/src/objects-client.ts:55`），非 2xx 抛 `ObjectsClientError` 并带上游状态码。

以 `complete-service-job` 为例，一次 apply 最多产生 **7 次串行 HTTP 调用**：读 ServiceJob、改 ServiceJob、读 TreatmentProduct、改库存、读 Technician、改计数、创建 Invoice（`services/svc-actions/src/handlers/complete-service-job.ts:36`、`44`、`55`、`68`、`85`）。

#### 7.5.5 apply 与 applyBatch 的差异与原子性结论

| 维度 | v2 apply | v2 applyBatch | v1 applyBatch |
|------|----------|---------------|---------------|
| 参数校验时机 | 执行前整体校验 | 逐条校验 | **先全量校验**，任一条失败整体拒绝 |
| 单条校验失败 | 400，不执行任何副作用 | 该条记 FAILED，其余继续 | 400 `ActionValidationFailed` |
| 单条执行失败 | 500，前序副作用已落库 | 该条记 FAILED，其余继续 | 抛出，前序副作用已落库 |
| 响应 | `{rid, status, result}` | `{results: [...]}`，恒 200 | `{}`（空对象） |
| 网关可达 | 是 | **否**（被路由到 svc-ontology） | **否**（同样被路由到 svc-ontology） |

**原子性结论：三个入口都不原子，且没有任何回滚。** 依据有三条：

- handler 里的多次写入是串行 `await`，没有事务包裹，也没有补偿逻辑（`services/svc-actions/src/handlers/complete-service-job.ts:36-99` 全程无 try/catch、无 undo）。
- svc-objects 侧也没有跨对象事务：对象写入是按对象类型分表/分 Map 的单次操作，动作层没有 `transactionId` 可传（`rejectUnsupportedOntologyScoping` 明确拒绝该参数）。
- 注释与实现不一致：`services/svc-actions/src/objects-client.ts:4` 写「read, create, and update objects **atomically** during action execution」，但该类只是逐条发 HTTP 请求，没有任何原子性保证。

因此一个真实的失败场景是：ServiceJob 已改为 completed、Technician 计数已减 1，但 Invoice 创建时主键冲突，接口返回 500，而前两步已经生效。`invoiceId` 由 jobId 派生（`services/svc-actions/src/handlers/complete-service-job.ts:78`），对同一 job 重复执行必然主键冲突——这正是 GUIDE.md 记录的 `complete-service-job` invoice-id 冲突。

另外，`schedule-new-job` 用模块级可变计数器生成 ID（`services/svc-actions/src/handlers/schedule-new-job.ts:11-12`），进程重启后回到 100，多实例部署会直接撞号。

### 7.6 函数服务：svc-functions

执行模型极其简单：把用户提交的代码字符串交给 `new Function`。

```ts
// services/svc-functions/src/executor.ts:58 —— 两个形参就是全部执行环境
const executor = new Function("args", "console", fn.code);
const output = executor(args, mockConsole);
```

端点只有一组，全部无权限守卫（`services/svc-functions/src/routes/v2/functions.ts:33`、`48`、`89`）：

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v2/functions` | 分页列表，pageSize 上限 1000 |
| POST | `/api/v2/functions` | 注册，重复 apiName 返回 409 |
| GET/PUT/DELETE | `/api/v2/functions/:functionRid` | 读取、更新、删除 |
| POST | `/api/v2/functions/:functionRid/execute` | 执行，body 为 `{args}` |

**与 actions 的区别是本质性的**：actions 有 handler、有注册表、会写对象、有执行日志；functions 没有 I/O 能力（执行环境里只有 `args` 和 mock `console`），是纯计算。`FunctionResult` 返回 `{output, duration, logs}`（`services/svc-functions/src/executor.ts:7`），`logs` 收集 mock console 的输出。

执行模型的三个事实：

- **不是沙箱**。`new Function` 的函数体在全局作用域求值，可以访问 `globalThis`、`process`、`fetch`，只是拿不到 `require`。文件头注释写「basic sandboxed function execution」，与实现不符。
- **超时只对异步代码有效**。定时器在 `new Function` 之外注册（`services/svc-functions/src/executor.ts:52`），若函数体是同步死循环，事件循环根本不会回到定时器，`setTimeout` 永不触发。
- **标称 TypeScript，实际只跑 JavaScript**。存储类型写死 `runtime: "TYPESCRIPT"`（`services/svc-functions/src/store/function-store.ts:21`），但服务依赖里没有任何转译器，`fn.code` 直接进 `new Function`。带类型标注的代码会抛 `SyntaxError`。

> 文档与实现不一致：`README.md:103` 写「3 seeded TypeScript functions with execute endpoint」，但演示脚本注册的三个函数（`scripts/demo/seed-pest-control.sh:375`、`385`、`395`）函数体引用了自由变量 `objects`，而执行器只注入 `args` 与 `console`。这些函数一旦执行就是 `ReferenceError: objects is not defined`。单元测试用的是 `return args.x + args.y;`（`services/svc-functions/tests/functions-service.test.ts:17`），因此测试通过而演示失败。

### 7.7 自动化：svc-sentinel

#### 7.7.1 三类模型

Sentinel 的核心数据是 monitor，由「触发器 + 副作用列表」组成（`services/svc-sentinel/src/store/monitor-store.ts:8`）：

| 维度 | OBJECT_CHANGED | SCHEDULE | PROPERTY_THRESHOLD |
|------|----------------|----------|--------------------|
| 语义 | 被监听属性发生变化 | 到点执行 | 数值越过阈值 |
| 配置 | `{properties: string[]}` | `{cron}` 或 `{intervalMinutes}` | `{threshold, direction}` |
| 有驱动方 | **否**（无对象变更事件源） | 是（60 秒 tick） | 部分（SCHEDULE 里附带阈值查询） |
| 有实现 | 是，但在未接线的 `trigger-evaluator.ts` 内 | 是（scheduler 自己实现一份） | 是 |

副作用有五种（`services/svc-sentinel/src/store/monitor-store.ts:13`）：WEBHOOK、ACTION、NOTIFICATION、LOG、EMAIL，由 `EffectExecutor` 按类型分发（`services/svc-sentinel/src/effects/effect-executor.ts:49`）。

#### 7.7.2 调度循环

调度器在服务启动时无条件启动（`services/svc-sentinel/src/server.ts:150`），间隔硬编码 60 秒，定时器 `unref()` 以免阻止进程退出（`services/svc-sentinel/src/scheduler.ts:37`）。伪代码：

```
每 60 秒 tick():
  monitors = listMonitors() 过滤 status==ACTIVE 且 trigger.type==SCHEDULE
  若为空则直接返回
  for m in monitors:
    若 shouldFireCron(m.trigger.config, now) 为假 → continue
    若配置了 objectType+property+threshold:
        调 svc-objects 求满足条件的对象数 → violated
        若未违反 → continue
    monitorStore.triggerMonitor(m.rid)      // 先记 lastTriggeredAt
    results = effectExecutor.executeAll(m.effects, context)
    executionLog.startExecution + completeExecution
  单个 monitor 抛错 → 记一条 FAILURE 执行记录，继续下一个
```

cron 匹配是手写的五行字段解析，支持 `*`、`*/n`、`a,b,c` 与精确值（`services/svc-sentinel/src/scheduler.ts:148`），不支持区间 `1-5`。`intervalMinutes` 的判定是 `now.getMinutes() % intervalMin === 0`，即每个满足条件的分钟都会触发一次。

#### 7.7.3 幂等性与失败处理

- **不幂等**。没有任何「本分钟已触发」的去重标记，`lastTriggeredAt` 只写不读；tick 重叠或进程重启后重入，同一分钟内会重复执行副作用。
- **无重试**。`executeAll` 逐条串行执行，每条 effect 自己吞掉异常并返回 `{status: "FAILURE"}`（例如 EMAIL 在缺 `RESEND_API_KEY` 时直接返回 FAILURE，`services/svc-sentinel/src/effects/email-effect.ts:24`）。失败不会让 monitor 进入 ERROR 状态，也不会重投。
- **无持久化游标**。内存模式下 monitor 与执行日志都在进程内，重启即清空；演示脚本也不种 monitor，所以全新环境下调度器每个 tick 都是空转。
- **时钟语义不统一**。调度器用本地时间（`services/svc-sentinel/src/scheduler.ts:140` 的 `now.getMinutes()`），而导出的 `evaluateTrigger` 用 UTC（`services/svc-sentinel/src/triggers/trigger-evaluator.ts:72` 的 `getUTCMinutes()`）。后者只被自己的单元测试引用，是**未被生产路径使用的死代码**——同一套 cron 语义存在两份实现，且时区不同。

### 7.8 协作资源：Webhooks、Media、Compass

#### 7.8.1 svc-webhooks

| 端点 | 说明 |
|------|------|
| GET/POST `/api/v2/webhooks` | 列表、注册 |
| GET/PUT/DELETE `/api/v2/webhooks/:webhookRid` | 读取、更新、删除 |
| GET `/api/v2/webhooks/:webhookRid/deliveries` | 投递历史 |
| POST `/api/v2/webhooks/test/:webhookRid` | 测试投递 |
| POST `/api/v2/events` | **事件入口**，按事件类型扇出到订阅方 |

数据模型：`{rid, name, url, secret?, events[], status, failureCount, lastDeliveryAt?}`，事件枚举固定五个：`object.created`、`object.updated`、`object.deleted`、`action.executed`、`ontology.modified`（`services/svc-webhooks/src/store/webhook-store.ts:8`）。

投递语义（`services/svc-webhooks/src/delivery.ts:10`）：HMAC-SHA256 签名放在 `X-OpenFoundry-Signature: sha256=...`，最多 3 次尝试，退避 500ms/1000ms，单次超时 10 秒，`/events` 用 `Promise.allSettled` 并行扇出后统一返回 202。

**关键事实：仓库内没有任何服务向 `/api/v2/events` 投递事件。** 全仓库检索 `WEBHOOKS_SERVICE_URL` 只命中网关配置与网关测试，svc-objects 与 svc-actions 在写入后都不发事件。因此 webhook 是一条只有接收端、没有发送端的管道——这也解释了为什么它能容忍自己不被 `start.sh` 启动。

#### 7.8.2 svc-media

| 端点 | 说明 |
|------|------|
| POST `/api/v2/media/upload` | 原始字节体上传，文件名走 `X-Filename` 头 |
| GET `/api/v2/media/:mediaRid` | 下载，回 `content-type`/`content-disposition` |
| GET/PUT `/api/v2/media/:mediaRid/metadata` | 读、合并元数据 |
| DELETE `/api/v2/media/:mediaRid` | 删除 |
| GET `/api/v2/media` | 列表 |

存储抽象是 `IMediaStore`，六个方法全部声明为 `T | Promise<T>`（`services/svc-media/src/store/media-store.ts:33`），因此内存与 S3 两个后端可互换。S3 是**显式 opt-in**：只有设置了 `S3_BUCKET` 才启用，仅有 `S3_ENDPOINT` 不够（`services/svc-media/src/server.ts:14`）。选择逻辑在 `services/svc-media/src/server.ts:56`：`s3Config ? new S3MediaStore(s3Config) : new MediaStore()`。

无 S3 时的降级是「退化为进程内 Map」，不是报错：`MediaStore` 把 `Uint8Array` 存在 `Map<string, StoredMedia>` 里（`services/svc-media/src/store/media-store.ts:57`），重启即丢，且**不受 `DATABASE_URL` 影响**——svc-media 没有 PG 后端。S3 模式下对象键由 RID 派生，文件名与创建时间写进对象自定义元数据（`services/svc-media/src/store/s3-media-store.ts:104`）。

#### 7.8.3 svc-compass

资源树只有三种节点类型：`SPACE | PROJECT | FOLDER`（`services/svc-compass/src/store/compass-store.ts:8`），节点形状为 `{rid, name, type, parentRid, description, createdAt, updatedAt, createdBy, markings[]}`。`markings` 只是一个字符串数组，不参与鉴权（svc-compass 没有 `registerAuthorization`）。

| 端点 | 说明 |
|------|------|
| GET/POST `/api/v2/compass/resources` | 列根节点、创建节点 |
| GET `/api/v2/compass/search` | 按名称搜索 |
| GET/PUT/DELETE `/api/v2/compass/resources/:rid` | 读取、更新、删除 |
| GET `/api/v2/compass/resources/:rid/children` | 子节点 |
| GET `/api/v2/compass/resources/:rid/path` | 路径 |
| POST `/api/v2/compass/resources/:rid/move` | 移动 |
| GET/POST `/api/v2/compass/lineage/*` | 血缘边与上下游遍历 |

血缘模型是独立的边集合（`services/svc-compass/src/store/lineage-store.ts:7`）：`{sourceRid, targetRid, edgeType, metadata?}`，六种边类型 `DERIVED_FROM | PRODUCES | CONSUMES | TRANSFORMS | COPIES | JOINS`。

> 文档与实现不一致：svc-compass 目录下有两套资源 store。`compass-store.ts`（SPACE/PROJECT/FOLDER）是 `services/svc-compass/src/server.ts:30` 实际使用的；`resource-store.ts`（FOLDER/PROJECT/DATASET/ONTOLOGY/FILE）只被 `pg-resource-store.ts` 引用，而 `pg-resource-store.ts` 谁都不引用。这两个文件是彻底的死代码。svc-compass 因此**没有可用的 PostgreSQL 后端**——旁证是它的配置里根本没有 `databaseUrl` 字段（`services/svc-compass/src/config.ts`）。

### 7.9 平台管理：svc-admin

#### 7.9.1 双存储模式

| 存储 | 触发条件 | 方法签名 | 可用性 |
|------|----------|----------|--------|
| `UserStore` / `GroupStore` / `MemoryAuditStore` | 无 `DATABASE_URL` | 同步返回 | 演示默认，全部路由可用 |
| `PgUserStore` / `PgGroupStore` / `PgAuditStore` | 有 `DATABASE_URL` | 全部 `async` | 部分路由失效 |

选择逻辑在 `services/svc-admin/src/server.ts:48-66`。审计 store 是第三种情况：`MemoryAuditStore` 永远报告 `available = false` 并返回空列表（`services/svc-admin/src/store/audit-store.ts:68`），因为审计条目由 `@openfoundry/db` 的 `writeAuditLog` 写入，内存模式根本没有表可写。`GET /admin/audit` 会把 `available` 字段带出去（`services/svc-admin/src/routes/v2/audit.ts:79`），前端据此区分「没有权限看」与「没有事件」。

#### 7.9.2 同步/异步错配

路由把 store 当作同步对象调用，而 PG 实现是异步的。逐条核实后的清单：

| 路由 | 调用的方法 | 无 DB | 有 DB 的实际表现 |
|------|-----------|-------|------------------|
| GET `/admin/users/search` | `searchUsers` | 正常 | 500（对 Promise 调 `.map`） |
| GET `/admin/users` | `listUsers` | 正常 | 500（对 Promise 调 `.filter`/`.map`） |
| POST `/admin/users` | `createUser` | 正常 | 201 但响应体为空对象 |
| GET `/admin/users/:userRid` | `getUser` | 正常 | 200 但响应体为空对象；带 `status` 断言时 400 |
| PUT `/admin/users/:userRid` | `updateUser` | 正常 | 200 但响应体为空对象 |
| DELETE `/admin/users/:userRid` | `deleteUser` | 正常 | 200 但响应体为空对象 |
| GET `/admin/users/:userRid/groups` | `getUser` + `getGroupsForUser` | 正常 | 500（对 Promise 调 `.map`） |
| GET `/admin/groups` | `listGroups` | 正常 | 500（对 Promise 调 `.map`） |
| POST/GET/DELETE `/admin/groups[/:groupRid]` | `createGroup`/`getGroup`/`deleteGroup` | 正常 | 201/200/204 但内容为空 |
| POST `/admin/groups/:groupRid/members` | `getUser` + `addMember` | 正常 | **静默不生效**，仍返回 201 |
| DELETE `.../members/:userRid` | `removeMember` | 正常 | **静默不生效**，仍返回 204 |
| GET `/admin/groups/:groupRid/members` | `getMembers` + `getUser` | 正常 | 正常（正确 `await`，`services/svc-admin/src/routes/v2/groups.ts:114`） |
| GET/POST `/admin/groups/:groupRid/groupMembers*` | `getMembers` 等 | 正常 | 正常（正确 `await`，`services/svc-admin/src/routes/v2/groups.ts:149`） |
| GET `/admin/audit` | `listEntries` | 正常 | 正常（接口本身是 async） |
| GET `/admin/stats` | `listUsers` + `listGroups` | 正常 | 500（`for...of` 一个 `{items,total}` 对象） |

这比 AGENTS.md 的结论更细一层：**并非全部 500，写操作多半是「静默成功但没写进去」**，比 500 更难排查。

还有一处更深的问题：即使补上 `await` 也修不好。内存 `UserStore.listUsers()` 返回 `StoredUser[]`（`services/svc-admin/src/store/user-store.ts:109`），而 `PgUserStore.listUsers()` 返回 `Promise<{items, total}>`（`services/svc-admin/src/store/pg-user-store.ts:116`）——**两者返回形状不同**。`listGroups` 同样如此。所以修复需要统一接口，不是加个 `await` 就完事。

> 文档与实现不一致：`AGENTS.md:167` 说这些路由「still calls them synchronously and so 500s in that mode」。代码显示只有列表类与 `/admin/stats` 会真的 500，单资源读写返回 200/201/204 但响应体是 `{}`，成员增删则是静默失效。

#### 7.9.3 审计与资源角色

审计是**只读视图**：`GET /admin/audit` 支持 `action`、`user`、`dateFrom`、`dateTo` 与游标分页，游标由 `@openfoundry/pagination` 编解码。仓库里没有任何 HTTP 写入审计的入口，写入方是库函数而非路由。

资源角色相关端点（`/admin/roles`、`/admin/resources/:resourceRid/roles*`、`.../policy`）全部以 `requirePermission("admin:manage")` 或 `resource:manage-roles` 守卫，其权限模型本身属于〈身份与权限〉章，此处只记一点：**svc-admin 是授权数据的写入方**，它把 grants 写进 `/tmp/openfoundry-data/authorization.json` 或 Postgres 的 `resource_role_grants`/`resource_policies`，其他服务只读。

### 7.10 管道：packages/pipeline

#### 7.10.1 DAG 数据结构

```ts
// packages/pipeline/src/types.ts:5 —— 管道定义
export interface PipelineDefinition {
  rid: string;
  name: string;
  steps: PipelineStep[];
  schedule?: PipelineSchedule;      // 只存不用，见下文
  inputDatasets: string[];          // 注释说 dataset RID，实际被当作对象类型 API 名
  ontologyRid?: string;
  outputDataset: string;            // 声明了目标 dataset，但执行时不写入
  status: PipelineStatus;
  createdAt: Date;
  updatedAt: Date;
}

// packages/pipeline/src/types.ts:21 —— 一个步骤
export interface PipelineStep {
  id: string;
  name: string;
  type: PipelineStepType;           // FILTER/MAP/AGGREGATE/JOIN/SORT/DEDUPLICATE/CUSTOM/DERIVE/LIMIT
  config: Record<string, unknown>;
  dependsOn?: string[];             // 依赖的步骤 id
}
```

#### 7.10.2 执行流程

`validateDag` 用 Kahn 算法做环检测，并检查重复 id、未知依赖与自引用（`packages/pipeline/src/dag.ts:11`）；`topologicalSort` 在排序前先校验，队列每次插入后重新排序以保证确定性输出（`packages/pipeline/src/dag.ts:125`、`packages/pipeline/src/dag.ts:141`）。

`PipelineExecutor.execute(pipeline, inputData, joinData)` 是**纯同步函数**：接收行数组，按拓扑序逐步骤变换，返回 `PipelineRun` 元数据（`packages/pipeline/src/executor.ts:288`）。它不 fetch、不写库、不碰 dataset。

两个语义细节值得记录：

- **多依赖只取最后一个**。步骤若有多个 `dependsOn`，只把**最后一个**依赖的输出当作输入（`packages/pipeline/src/executor.ts:335`），前面的依赖被忽略。真正的 DAG 语义需要合并，这里没有。
- **`rowsProcessed` 是各步输入行数之和**，不是去重后的行数（`packages/pipeline/src/executor.ts:345`），因此会大于源数据行数。

#### 7.10.3 与 svc-datasets 的关系

`packages/pipeline` 只被 svc-datasets 消费：`services/svc-datasets/src/store/pipeline-store.ts` 复用它导出的类型，`services/svc-datasets/src/routes/v2/pipelines.ts` 提供全部 HTTP 端点（`/api/v2/pipelines*`，含 `validate`、`runs`、`execute`）。库负责算法，服务负责 I/O——分工清晰。

但 `execute` 的实现暴露了一个结构性问题（`services/svc-datasets/src/routes/v2/pipelines.ts:513`）：

1. 取 `inputDatasets[0]` 当**对象类型名**（注释自认这是「convention」，`services/svc-datasets/src/routes/v2/pipelines.ts:523`），从 svc-objects 拉全量对象；
2. 调 `PipelineExecutor.execute` 得到一个 `PipelineRun`（只有元数据）；
3. 因为 executor 不返回数据，路由**用 `executeStepLocal` 把每一步再算一遍**以拿到结果行（`services/svc-datasets/src/routes/v2/pipelines.ts:578`）。

也就是说同一套变换逻辑有两份实现：库里的 `PipelineExecutor.executeStep` 与路由里的 `executeStepLocal`（`services/svc-datasets/src/routes/v2/pipelines.ts:196`），任何一侧改动都可能让「运行记录里的行数」与「返回的数据」对不上。

顺带一提，`services/svc-datasets/src/routes/v2/pipelines.ts` 里的 `pipelineStore` 调用同样没有 `await`（例如 `services/svc-datasets/src/routes/v2/pipelines.ts:519`），而 `PgPipelineStore.getPipeline` 是 `async`（`services/svc-datasets/src/store/pg-pipeline-store.ts:133`）——`DATABASE_URL` 模式下 `/api/v2/pipelines*` 与本章其他服务一样会失效。

**执行语义结论：pipeline 是「模拟运行」而非物化运行。** 输出行只出现在 HTTP 响应里，从不写入 `outputDataset`，不创建 dataset 事务，也不产生分支。全文检索 `pipelines.ts` 找不到任何 `putFile`/`transaction`/`commit`。`PipelineSchedule` 同样只存不用——svc-datasets 里没有任何 `setInterval` 或调度器，`schedule` 字段仅是元数据。数据集分支与事务的真实语义见〈数据与 AI〉章。

### 7.11 跨服务协作模式

这是本章最重要的架构结论：**这一层是「共享库 + 点对点 HTTP」，不是事件总线，也不是共享数据库。**

| 调用方 | 被调方 | 用途 | 证据 |
|--------|--------|------|------|
| svc-actions | svc-objects | 动作副作用读写对象 | `services/svc-actions/src/objects-client.ts:43` |
| svc-sentinel | svc-objects | 阈值查询取对象 | `services/svc-sentinel/src/server.ts:146` |
| svc-sentinel | svc-ontology | 解析对象类型所属本体 | `services/svc-sentinel/src/server.ts:147` |
| svc-sentinel | svc-actions | ACTION 类副作用 | `services/svc-sentinel/src/effects/action-effect.ts:28` |
| svc-sentinel | api.resend.com | EMAIL 类副作用 | `services/svc-sentinel/src/effects/email-effect.ts:63` |
| svc-admin | ontology/objects/datasets/functions/actions | `/admin/stats` 汇总计数 | `services/svc-admin/src/routes/v2/stats.ts:26` |
| svc-datasets | svc-objects + svc-ontology | 管道执行取源数据 | `services/svc-datasets/src/routes/v2/pipelines.ts:21` |
| svc-objects | svc-aip | 写入时生成向量 | `services/svc-objects/src/server.ts:154` |
| svc-aip | svc-ontology | 读本体做提示词上下文 | `services/svc-aip/src/llm/real-client.ts:26` |

四点结论：

1. **调用方向是单向的、无环的**（aip 与 objects 互为对端但用途不同，不构成控制环），没有服务编排层，也没有分布式事务。跨服务调用失败时，调用方各自决定降级：svc-admin 的 `fetchCount` 捕获异常返回 0，动作直接抛 500。
2. **多数服务不注册授权**。全仓库只有 svc-actions、svc-admin、svc-datasets 调用 `registerAuthorization`；svc-functions、svc-sentinel、svc-webhooks、svc-media、svc-compass 一条 `requirePermission` 都没有。也就是说这些服务的任何端点对任何调用方开放，安全边界完全依赖「网关是唯一入口」这一部署假设。
3. **webhooks 是孤岛**。它设计成被 `POST /api/v2/events` 驱动，但仓库里没有生产者。若要让 `object.created` 真的外发，必须在 svc-objects 里加一次 HTTP 调用——这是当前架构里最明显的一处「设计了但没有接线」。
4. **Sentinel 调 actions 的路径是坏的**：`services/svc-sentinel/src/effects/action-effect.ts:28` 拼出 `${actionsServiceUrl}/api/v2/actions/execute`，而 svc-actions 没有这个路由（它的动作路径是 `/api/v2/ontologies/:rid/actions/:name/apply`）。该 effect 必然 404，且被包装成 `status: "FAILURE"` 的普通结果，不会报错。同文件第 8 行的注释还把 actions 服务写成 `http://localhost:8085`（那是 datasets 的端口，`services/svc-sentinel/src/effects/action-effect.ts:8`）。`services/svc-sentinel/tests/sentinel-service.test.ts:578` 甚至断言了这个错误的 URL，所以测试是绿的。

### 7.12 设计决策与理由

**1. 动作注册表放在服务内，而不是从本体服务读取。**
svc-actions 启动时把四个演示动作硬编码注册进去（`services/svc-actions/src/handlers/index.ts:22`），svc-ontology 则独立保存动作类型定义。取舍：动作服务不依赖本体服务即可启动、可独立测试；代价是**两份真相源**，本体里列出但注册表里没有的动作，apply 必然 404。这正是集成测试那条长期失败用例的根因（`tests/integration/api-endpoints.test.ts:535` 期望 400，实际 404）。

**2. 副作用走 HTTP 而不是共享存储。**
`ObjectsClient` 是普通 `fetch` 客户端，而不是直接打开 svc-objects 的 store。取舍：进程边界清晰，内存/PG 两种存储对动作层透明，动作服务可单独部署；代价是没有跨调用事务，一次动作的多次写入无法回滚，且每次调用多一次序列化与网络开销。

**3. 函数执行用 `new Function` 而不是容器沙箱。**
取舍：零依赖、启动快、单文件即可实现，演示场景足够；代价是完全没有隔离（可读 `process.env`、可发起网络请求）、超时对同步代码无效、无法限制内存。这条决策把「执行用户代码」的风险整体推给了部署环境。

**4. Sentinel 用进程内 `setInterval` 而不是外部调度器。**
取舍：不引入 Redis/队列依赖，`unref()` 后不影响进程退出，演示可解释性强；代价是无持久化游标、不幂等、多实例部署会重复触发，且 cron 解析自己手写了一份（与 `trigger-evaluator.ts` 那份语义还不一致）。

**5. 存储接口返回 `T | Promise<T>`，由调用方决定是否 await。**
这是横切多服务的统一约定：`ObjectReadWriteStore`、`ActionRegistrySource`、`ActionExecutionLog`、`IMediaStore` 都是这个形状。取舍：内存模式代码最简（无需 async），同一份路由能同时服务两种后端；代价是**漏掉一个 `await` 不会编译报错，只在 PG 模式下静默失效**。svc-admin、svc-actions v2、svc-functions、svc-sentinel、svc-webhooks 与 svc-datasets 的管道路由都是这条约定的受害者（都同步调用异步 store），而 svc-actions v1 与 svc-objects 的 v2 路由是正确用法的范例（`services/svc-objects/src/routes/v2/object-sets.ts:111`）。

**6. v1 与 v2 各自独立实现，不复用。**
`routes/v1/actions.ts` 不是 `routes/v2/actions.ts` 的别名，v1 的 `apply` 返回空对象、`applyBatch` 先全量校验。取舍：wire 形状能各自贴合 Foundry 的 SDK 契约（v1 `ApplyActionResponse` 确实没有字段）；代价是**修复只做一半**——v1 用了异步容忍接口，v2 还在用同步签名，于是 PG 模式下 v2 全坏而 v1 正常。

**7. Pipeline 做成纯函数库，HTTP 面留给 svc-datasets。**
取舍：算法可单元测试、无 I/O、可被未来的调度器复用；代价是 `execute` 返回的是元数据而非数据，调用方为了拿结果不得不重放一遍变换，产生第二份实现。

**8. Media 的 S3 后端显式 opt-in，缺配置就退化为内存。**
取舍：`bash start.sh` 在没有对象存储的机器上也能跑通全部演示；代价是「上传成功」在重启后消失，且没有任何提示告诉用户数据没落盘。

### 7.13 已知限制与坑

**未启动服务的 502。** svc-compass、svc-webhooks、svc-media 不在 `start.sh:118` 的名单里，但网关为它们注册了代理路由（`services/svc-gateway/src/routes/v2/index.ts:47`、`51`、`52`）。服务不可达时 `proxyRequest` 的 `fetch` 抛错，统一回 502 `BAD_GATEWAY`（`services/svc-gateway/src/proxy.ts:125`）。因此 `/api/v2/compass/*`、`/api/v2/webhooks*`、`/api/v2/media*` 的 502 不是权限问题也不是路由问题，而是「进程根本没起」。

**8091 冲突检查缺失。** `start.sh:85` 杀 8080–8088、8092、3000，`start.sh:131` 自检 9 个端口，两处都**没有 8091**。svc-sentinel 却是被启动的（`start.sh:118`）。后果：上一次运行残留的 sentinel 进程不会被清理，新进程 `listen` 失败退出，而自检根本不看这个端口，脚本仍然打印「OpenFoundry ready」。你会以为调度器在跑新代码，实际在跑旧进程。

**apply 的 404 / 400 不一致。** 路由先查注册表再校验参数（`services/svc-actions/src/routes/v2/actions.ts:80` 与 `86`），所以「动作不存在」永远压过「参数不合法」。集成测试从 svc-ontology 的 `actionTypes` 列表里取第一个动作再 apply（`tests/integration/api-endpoints.test.ts:520`），而 svc-actions 的注册表只有硬编码的四个名字——两个列表一旦不重合，测试拿到 404 而不是期望的 400。这条失败用例在 `AGENTS.md:59` 里被记为「predates this note」，代码层面的根因就是这行顺序。

**applyBatch 与 executions 经网关不可达。** 网关正则带 `$` 锚点（`services/svc-gateway/src/routes/v2/ontologies.ts:35`），只放行 `apply` 与 `validate`，其余动作路径全部落到 svc-ontology 并 404。直接打 8083 端口是通的，但这违反「网关是唯一入口」的部署前提。

**svc-admin 在 `DATABASE_URL` 下的部分失效。** 见上文表格：列表类与 `/admin/stats` 直接 500，单资源读写返回空对象，成员增删静默失效。根因有两层：路由未 `await`，且内存/PG 两种 store 的**返回形状本就不同**。

**svc-actions v2 在 `DATABASE_URL` 下整组失效。** v2 路由用同步签名调用 `registry.getAction`/`log.logStart`（`services/svc-actions/src/routes/v2/actions.ts:80`、`98`、`111`），而 server 把异步实现强转成同步类型传入（`services/svc-actions/src/server.ts:146`）。PG 模式下 `action` 是一个 Promise，`validateActionParameters` 里遍历 `action.parameters` 会抛 `TypeError` → 500。v1 路由因为用了 `ActionRegistrySource` 接口并正确 `await`（`services/svc-actions/src/routes/v1/actions.ts:91`、`120`）反而不受影响。同类问题在 svc-functions、svc-sentinel、svc-webhooks 的路由里同样存在（都同步调用异步 store）。

**Sentinel 在无 DB 模式下的行为。** 没有 `DATABASE_URL` 时 monitor 存在进程内 Map，而演示脚本不种 monitor，所以调度器每个 tick 都在空转——它「在跑」，但没有监控对象。这不是 bug，但会让人误判「调度器没工作」。

**Sentinel 在 PG 模式下的崩溃风险（未验证）。** `Scheduler.tick` 第一行就 `monitorStore.listMonitors().filter(...)`（`services/svc-sentinel/src/scheduler.ts:52`），PG 实现返回 Promise，`.filter` 不是函数 → 同步抛 `TypeError`。该异常发生在 `try` 之外，而定时器回调是 `() => void this.tick()`（`services/svc-sentinel/src/scheduler.ts:36`），被丢弃的 Promise 拒绝在 Node 默认策略下会导致进程退出。也就是说设了 `DATABASE_URL` 之后，svc-sentinel 最多活 60 秒。标注「未验证」是因为没有实际以 PG 模式启动过；结论由代码路径推断。

**Sentinel 的 ACTION 副作用必然失败。** 目标路由 `/api/v2/actions/execute` 在 svc-actions 里不存在，见〈跨服务协作模式〉。

**函数服务的三个坑。** 无权限守卫；超时对同步代码无效（`services/svc-functions/src/executor.ts:52`）；标称 TypeScript 实际只跑 JS，而演示数据恰好引用了未定义的 `objects`（`scripts/demo/seed-pest-control.sh:375`），所以 README 宣称的三个可执行函数一个都跑不起来。

**Pipeline 不落盘。** `execute` 只回数据不写 `outputDataset`，`schedule` 只存不调度，多依赖只取最后一个。任何「管道产出会变成 dataset」的预期都是错的。

**重复的 ID 生成方式。** `schedule-new-job` 用模块级计数器（`services/svc-actions/src/handlers/schedule-new-job.ts:11`），`complete-service-job` 用 `INV-<jobId>` 派生（`services/svc-actions/src/handlers/complete-service-job.ts:78`）。前者重启回绕、后者天然不唯一，两者都没有唯一性兜底。

**Compass 的死代码。** `services/svc-compass/src/store/resource-store.ts` 与 `services/svc-compass/src/store/pg-resource-store.ts` 无人引用，`ResourceType` 有两份不一致的定义（SPACE/PROJECT/FOLDER 与 FOLDER/PROJECT/DATASET/ONTOLOGY/FILE）。读代码时容易误判 Compass 支持 dataset 与 ontology 节点。

### 7.14 关键代码位置

- `services/svc-actions/src/routes/v2/actions.ts:80` —— 查注册表在校验之前，404 压过 400 的根因
- `services/svc-actions/src/routes/v2/actions.ts:103` —— handler 分发点，无 handler 则静默成功
- `services/svc-actions/src/routes/v1/actions.ts:157` —— v1 applyBatch 先全量校验再执行
- `services/svc-actions/src/objects-client.ts:43` —— 动作副作用唯一的出站 URL 拼装点
- `services/svc-actions/src/handlers/complete-service-job.ts:78` —— invoiceId 由 jobId 派生导致冲突
- `services/svc-functions/src/executor.ts:58` —— `new Function` 执行入口
- `services/svc-sentinel/src/scheduler.ts:52` —— tick 第一行即同步调用异步 store
- `services/svc-sentinel/src/effects/action-effect.ts:28` —— 指向不存在的 actions 路由
- `services/svc-gateway/src/routes/v2/ontologies.ts:35` —— 只放行 apply/validate 的正则
- `services/svc-admin/src/routes/v2/groups.ts:114` —— 唯一正确 `await` 的组成员路由
- `services/svc-datasets/src/routes/v2/pipelines.ts:572` —— 为拿数据而重放变换的注释与实现
- `packages/pipeline/src/executor.ts:288` —— 纯函数式管道执行器签名

---

## 第 8 章 数据与 AI：数据集、存储引擎与 AIP

### 8.1 本章导读

这一层负责「数据落在哪里、怎么读回来」：`svc-datasets` 提供数据集（Dataset）、分支（Branch）、事务（Transaction）、文件（File）四个核心对象以及 connectivity 与 pipeline 两个附加资源面；`@openfoundry/db` 提供连接池、迁移执行器与查询辅助；`svc-aip` 在数据之上提供 Foundry `models` v2 的 8 个操作和一组 AIP 会话接口。边界要先说清楚：文件字节没有对象存储，事务不提供文件版本，模型没有运行时——这三处缺口不是遗漏，而是每个端点都选择「拒绝或如实返回」而非编造字段的结果。本章的多数结论都用两种方式核实过：读代码，以及把仓库里的 SQL 原样加载进 PGlite 后执行各 Pg store 的真实语句。

### 8.2 职责与边界

三个组件、一条分界线：

- `services/svc-datasets`（默认端口 8085，`services/svc-datasets/src/config.ts:39`）拥有以下几块资源面：`/api/v2/datasets*`、`/api/v1/datasets*`、`/api/v2/connectivity/*`、`/api/v2/pipelines*`，以及健康检查。v1 与 v2 共用 store、各自序列化。
- `@openfoundry/db` 是纯库：`createPool`/`closePool`/`withTransaction`（`packages/db/src/pool.ts:28`）、`runMigrations`（`packages/db/src/migrator.ts:18`）、时间戳与 JSONB helper、审计写入，以及一个没有服务使用的通用 `PgStore`（`packages/db/src/pg-store.ts:24`）。
- `services/svc-aip`（默认端口 8092，`services/svc-aip/src/config.ts:91`）同时承载两件不相干的事：Foundry `models` v2 的资源管理，和 AIP 的 chat / query / embeddings / agents。前者有完整的领域建模，后者更像演示用的薄封装。

明确不做的事：不把文件内容写入 S3/GCS（PG 模式下内容留在进程内存，`services/svc-datasets/src/store/pg-file-store.ts:34-44`）；不提供事务级文件版本（`services/svc-datasets/src/routes/v1/files.ts:20-42`）；不加载模型、不训练（`services/svc-aip/src/routes/v2/models.ts:38-66`）；不暴露数据集 schema 端点——DDL 里有 `datasets.schema JSONB` 列（`scripts/migrate.sql:349`），但没有任何路由或 `PgDatasetStore` 字段读它。与相邻模块的分工：对象数据面与本体在〈本体与对象数据面〉章，权限与 `org_rid`/RLS 的模型在〈身份、权限与多租户安全〉章，pipeline 的编排语义在〈平台业务服务〉章，本章只讲它与数据集的接缝。

> 注：任务简报里的 `createPgConfig` 与 `registerDb` 并不属于 `@openfoundry/db`。`createPgConfig` 来自 `@electric-sql/pglite/node-postgres`，只在测试里把 PGlite 包装成 `pg.Pool`（`services/svc-datasets/tests/connectivity-contract.test.ts:17`）；包内没有 `registerDb` 这类 Fastify 插件，唯一的插件是 `auditHook`（`packages/db/src/audit-hook.ts:92`）。

### 8.3 代码结构

| 路径 | 作用 |
|------|------|
| `services/svc-datasets/src/server.ts` | 存储选择（内存 vs Pg）与路由装配，`DATABASE_URL` 是唯一开关 |
| `services/svc-datasets/src/store/dataset-store.ts` | 内存 Dataset/Branch/Transaction store 与 `DatasetReadWriteStore` 接口 |
| `services/svc-datasets/src/store/pg-dataset-store.ts` | 同一接口的 Postgres 实现，全部 async |
| `services/svc-datasets/src/store/file-store.ts` | 内存文件 store，键为 `${datasetRid}::${path}` |
| `services/svc-datasets/src/store/pg-file-store.ts` | 文件元数据入 PG、内容留内存缓存 |
| `services/svc-datasets/src/routes/v1/` | v1 数据面：`serializers.ts` 负责投影，`files.ts` 记录未实现操作 |
| `services/svc-datasets/src/routes/v2/` | v2 数据面、connectivity、pipelines、分页 helper |
| `services/svc-datasets/src/connectivity/` | 20 个操作中的 18 个、secrets 加密、Pg/内存双 store |
| `services/svc-aip/src/models/` | models v2 类型、API 解析、内存与 Pg store |
| `services/svc-aip/src/routes/v2/models.ts` | 8 个已服务操作，以及 24 个未服务操作的理由 |
| `services/svc-aip/src/routes/v2/aip.ts` | chat / query / generate-function / embeddings / semantic-search / agents |
| `services/svc-aip/src/llm/` | 提供方选择、真实客户端、mock 客户端、本体感知包装 |
| `packages/db/src/migrator.ts` | `_migrations` 追踪表与逐文件事务执行 |
| `db/migrations/`、`scripts/migrate.sql` | 两套 schema 来源（见〈迁移体系〉） |

### 8.4 数据集领域模型

四个对象是层层包含关系，但只有前两层在存储里是「容器」，事务与文件都是平铺的表：事务挂在分支上，文件只按数据集 + 路径寻址，不挂在事务上。

```mermaid
erDiagram
    DATASET ||--o{ BRANCH : contains
    DATASET ||--o{ TRANSACTION : records
    BRANCH ||--o{ TRANSACTION : opens
    DATASET ||--o{ FILE : stores
    DATASET {
        string rid PK
        string name
        string parentFolderRid
        string createdAt
    }
    BRANCH {
        string rid PK
        string name
        boolean isDefault
        string createdAt
    }
    TRANSACTION {
        string rid PK
        string branchRid FK
        string status
        string type
        string committedAt
    }
    FILE {
        string datasetRid FK
        string path
        string transactionRid
        number size
        string updatedTime
    }
```

图中的基数要配一段说明：`Dataset` 创建时自动带一个名为 `main` 的默认分支（`services/svc-datasets/src/store/dataset-store.ts:119-126`），所以 `BRANCH` 的最小基数是 1 而不是 0；`TRANSACTION` 在 TypeScript 类型里只有 `branchRid`、没有 `datasetRid`，但 PG 表 `dataset_transactions` 有 `dataset_rid` 列（`db/migrations/003_phase1_tables.sql:74-84`），内存实现则把事务存在所属 `StoredDataset.transactions` 这个 Map 里——这是类型与表之间唯一的形状差；`FILE` 的主键是 `(datasetRid, path)` 复合键，`transactionRid` 只是记录，不参与寻址，也不构成外键语义上的「归属」。

核心类型（节选，注释说明它约束了什么）：

```ts
export type TransactionStatus = "OPEN" | "COMMITTED" | "ABORTED";
export type TransactionType = "UPDATE" | "APPEND" | "SNAPSHOT";

export interface Branch {
  rid: string;
  name: string;          // 分支以 name 寻址，rid 只出现在事务的 branchRid 上
  isDefault: boolean;    // 默认分支不可删除
  createdAt: string;
}

export interface Transaction {
  rid: string;
  branchRid: string;
  status: TransactionStatus;
  type: TransactionType;
  createdAt: string;
  committedAt?: string;  // 只有 commit 会写；abort 没有关闭时间
}
```

```ts
// 路由可以依赖的最小接口：每个方法都可能同步或异步返回。
// 处理器一律 await，才能在两种后端上同时正确。
export interface DatasetReadWriteStore {
  createDataset(input: CreateDatasetInput): StoredDataset | Promise<StoredDataset>;
  getDataset(rid: string): StoredDataset | Promise<StoredDataset>;
  createBranch(datasetRid: string, input: CreateBranchInput): Branch | Promise<Branch>;
  // …… list/get/delete branch、open/commit/abort transaction 同形
}
```

文件记录是唯一同时携带「内容」与「元数据」的类型：`StoredFile` 有 `content: Uint8Array` 与 `updatedTime: string`（`services/svc-datasets/src/store/file-store.ts:7-23`），后者是 Foundry 两个版本的 `File` 都要求的必填字段，注释明确写了它「每次 putFile 都会前移，包括覆盖同一路径」。

| 领域对象 | 内存键 | PG 表 | 关键约束 |
|----------|--------|-------|----------|
| Dataset | `Map<rid, StoredDataset>` | `datasets` | 名称唯一只由内存线性查重保证；PG 两套 schema 都没有 name 唯一约束 |
| Branch | 所属 dataset 的 `Map<name, Branch>` | `dataset_branches` | `UNIQUE (dataset_rid, name)`，`is_default` 标记 |
| Transaction | 所属 dataset 的 `Map<rid, Transaction>` | `dataset_transactions` | 状态机 `OPEN → COMMITTED/ABORTED`，`CHECK` 限定取值 |
| File | `Map<"${datasetRid}::${path}", StoredFile>` | `dataset_files` | 主键 `(dataset_rid, path)`，同一路径只有一份 |

语义要点（都能在 store 里逐行读到）：创建数据集即创建 `main` 分支；删除默认分支返回 `INVALID_ARGUMENT`（`services/svc-datasets/src/store/dataset-store.ts:197-199`）；重复创建分支返回 `CONFLICT`；对非 `OPEN` 事务再次 commit/abort 返回 `INVALID_ARGUMENT`；文件覆盖是「替换记录」，旧内容与旧写入时间一起消失。

### 8.5 端点地图与 v1/v2 序列化差异

两个版本不是同一套 API 加前缀。v1 的每条响应都经过 `services/svc-datasets/src/routes/v1/serializers.ts` 投影；v2 只对数据集做了投影，分支与事务直接把 store 的对象交给 Fastify（`services/svc-datasets/src/routes/v2/datasets.ts:7-16`、`:99`、`:163`）。文件面反过来——两个版本各自投影，字段恰好相同，但仍然分开写。

| 能力 | v2 | v1 | 差异要点 |
|------|----|----|----------|
| 数据集 | 列表、创建、读取、删除 | 仅创建、读取 | v1 没有列表；创建时 `parentFolderRid` 必填 |
| 分支 | 列表、创建、按 name 读取、删除 | 同四项，按 `branchId` 读取 | v1 的响应只有 `{ branchId }`，值是分支名 |
| 事务 | 打开、commit、abort | 同三项 | 字段名不同：`type`→`transactionType`，`createdAt`→`createdTime`，`committedAt`→`closedTime` |
| 文件 | 列表、上传、下载字节、删除 | 列表、读元数据、删除 | v1 没有上传，也没有下载字节 |
| 数据集 schema | 无 | 无 | DDL 有 `schema` 列，但没有路由暴露 |

细节都能在代码里对上：

- **状态码不同**：v2 的创建类端点返回 201（`services/svc-datasets/src/routes/v2/datasets.ts:47`、`:98`、`:162`），v1 全部返回默认 200（`services/svc-datasets/src/routes/v1/datasets.ts:67`、`:113`、`:178`）。
- **文件下载只有 v2 有**：`GET /api/v2/datasets/{rid}/files/*` 读 store 后把 `content` 以 `Buffer` 返回，并设置 `content-type` 响应头（`services/svc-datasets/src/routes/v2/files.ts:140-152`）；v1 的同名 GET 返回的是 `File` 元数据 JSON（`services/svc-datasets/src/routes/v1/files.ts:111-126`），v1 根本没有下载字节的端点。
- **`contentType` 两边都不发**：它是 store 的字段，不是 Foundry 的字段，只在下载响应头里出现（`services/svc-datasets/src/routes/v2/files.ts:29-39` 的注释把这条理由写全了）。
- **`V1Branch.transactionRid` 永远缺席**：v1 声明了这个可选字段，但两个 store 都无法回答「分支上最近的事务」——`PgDatasetStore` 查 `dataset_transactions` 没有 `ORDER BY`，而 `createdAt` 在毫秒级会碰撞，所以字段被省略而不是猜一个（`services/svc-datasets/src/routes/v1/serializers.ts:21-33`）。
- **v1 的事务类型是白名单**：`DELETE` 在 v1 的枚举里存在，但 store 没有对应表示，因此被拒绝而不是悄悄当成 `APPEND`；缺省类型取 `APPEND`（`services/svc-datasets/src/routes/v1/datasets.ts:13-29`）。
- **v1 上传未实现**：`POST /datasets/{rid}/files:upload` 需要把文件放进一个具名事务，而 store 一个路径只有一份记录，后写的会覆盖前面的，所以宁可不注册（`services/svc-datasets/src/routes/v1/files.ts:67-81`）。
- **v1 可能读不出数据集**：v2 允许创建没有父文件夹的数据集，v1 的 `Dataset.parentFolderRid` 必填，读到这类数据集会返回 500 `DatasetNotRepresentableInV1`（`services/svc-datasets/src/routes/v1/serializers.ts:55-81`）。

### 8.6 写路径：一次上传如何落库

下面的时序图把「创建数据集 → 建分支 → 开事务 → 上传文件 → commit/abort」串起来，并标出每一步真正碰到的存储。关键点是：文件上传与事务之间只有一条「记录一个字符串」的联系，没有校验、没有锁、没有回滚。

```mermaid
sequenceDiagram
    participant C as Client
    participant A as svc-datasets
    participant D as DatasetStore or PgDatasetStore
    participant F as FileStore or PgFileStore
    participant P as Postgres

    C->>A: POST /api/v2/datasets
    A->>D: createDataset
    D->>P: INSERT datasets
    D->>P: INSERT dataset_branches main is_default true
    A-->>C: 201 Dataset

    C->>A: POST /api/v2/datasets/{rid}/branches
    A->>D: createBranch name
    D->>P: INSERT dataset_branches
    A-->>C: 201 Branch

    C->>A: POST /api/v2/datasets/{rid}/transactions
    A->>D: openTransaction branchRid type
    D->>P: SELECT branch then INSERT transaction status OPEN
    A-->>C: 201 Transaction

    C->>A: PUT /api/v2/datasets/{rid}/files/raw/a.csv
    A->>D: getDataset
    A->>F: putFile datasetRid path bytes contentType emptyString
    F->>P: INSERT dataset_files ON CONFLICT DO UPDATE
    A-->>C: 201 File

    C->>A: POST /api/v2/datasets/{rid}/transactions/{tx}/commit
    A->>D: commitTransaction
    D->>P: UPDATE status COMMITTED committed_at NOW
    A-->>C: 200 Transaction
```

逐步说明与失败分支：

1. **创建数据集**：内存实现先线性查重名称，再生成 `ri.foundry.main.dataset.*` 与默认分支 RID（`services/svc-datasets/src/store/dataset-store.ts:97-130`）；PG 实现把两条 INSERT 放在同一方法里但没有包事务（`services/svc-datasets/src/store/pg-dataset-store.ts:93-119`），所以分支插入失败会留下一个没有分支的数据集。v1 还额外要求 `parentFolderRid`（`services/svc-datasets/src/routes/v1/datasets.ts:61-63`），v2 不要求。
2. **建分支**：名称在数据集内唯一；内存实现用 `Map.has`，PG 实现把 `23505` 唯一冲突翻译成 `CONFLICT`（`services/svc-datasets/src/store/pg-dataset-store.ts:70-77`）。
3. **开事务**：先验证分支存在，再写入 `status: "OPEN"`。v2 接受 `branchRid` 或 `branchName` 查询参数，并接受 `transactionType` 作为 `type` 的别名（`services/svc-datasets/src/routes/v2/datasets.ts:141-164`）；v1 用 `branchId`（名字），缺省时解析 `isDefault` 分支（`services/svc-datasets/src/routes/v1/datasets.ts:169-176`）。
4. **上传文件**：路由先把 body 读成 `Buffer`，再用 `fileStore.putFile(datasetRid, path, content, contentType, "")` 落库（`services/svc-datasets/src/routes/v2/files.ts:128-134`）。第五个参数是硬编码的空字符串，注释写着「direct upload 时 transactionRid 可选」——也就是说上传完全不校验事务是否 `OPEN`，也不把事务 RID 记进去。
5. **commit / abort**：只改事务行的状态。abort 不删除已写入的文件，commit 也不「发布」任何东西——因为文件从来就不属于某个事务。

| 失败点 | 触发条件 | 返回 | 已落库的副作用 |
|--------|----------|------|----------------|
| 建数据集重名 | 同名数据集已存在（仅内存模式） | `CONFLICT` | 无；PG 模式无唯一约束，重名会成功 |
| 删默认分支 | `isDefault` 为真 | `INVALID_ARGUMENT` | 无 |
| 开事务分支不存在 | `branchRid` 不匹配 | `NOT_FOUND` | 无 |
| 重复 commit/abort | 状态不是 `OPEN` | `INVALID_ARGUMENT` | 无 |
| 上传路径以 `/` 开头 | 仅 `pathPrefix` 列表校验，上传不校验 | 上传不报错 | 文件已写入 |

上表最后一行值得单独说：`GET .../files` 会拒绝以 `/` 开头的 `pathPrefix`（`services/svc-datasets/src/routes/v2/files.ts:94-100`），但 `PUT .../files/*` 的路径来自通配符，没有任何归一化或前导斜杠校验，同一个路径可以用 `/a.csv` 与 `a.csv` 两种形式各存一份。

> 实现与 Foundry 语义不一致：事务在本仓库里只是一个元数据状态机，不覆盖文件写入。Foundry 的写入必须落在一个打开的事务里，commit 才使数据可见；这里 `putFile` 不校验事务、不记录事务（直传写空串），`abort` 也不回滚。要支持 Foundry 的语义需要按事务保存文件版本目录，那是新存储而不是补参数——`services/svc-datasets/src/routes/v1/files.ts:20-42` 把这一点写成了显式拒绝：`branchId`/`startTransactionRid`/`endTransactionRid`/`transactionRid` 四个参数一律返回 `INVALID_ARGUMENT`，而不是静默忽略。

### 8.7 存储双模：内存是默认，Postgres 是开关

`DATABASE_URL` 是唯一的切换条件，两个服务各自在 `createServer` 里做同一个判断（`services/svc-datasets/src/server.ts:63-77`、`services/svc-aip/src/server.ts:51-60`）。没有 `DATABASE_URL` 时，数据集相关的一切都在进程内存里，**不落盘**。

| 维度 | 内存 / 进程内 | Postgres |
|------|---------------|----------|
| 启动条件 | `DATABASE_URL` 未设置 | `DATABASE_URL` 已设置 |
| 数据位置 | 各 store 的 `Map`/数组，进程退出即丢 | 表：`datasets`、`dataset_branches`、`dataset_transactions`、`dataset_files`、`pipelines`、`ml_*`、`aip_*` |
| 文件内容 | `StoredFile.content` 在 Map 里 | 元数据入 PG，内容仍在进程内 `contentCache`（`services/svc-datasets/src/store/pg-file-store.ts:44`） |
| 并发语义 | 单进程同步，无锁；同名检查与写入之间无原子性 | 依赖 `dataset_branches` 等表的唯一约束与 `CHECK`；`datasets.name` 没有唯一约束；多语句方法未包事务 |
| 测试隔离 | 每个用例 `new DatasetStore()` / `new FileStore()` | PGlite 内存库逐 schema 构建（`services/svc-datasets/tests/connectivity-contract.test.ts:60-72`） |
| 重启后 | 数据集、分支、事务、文件全部消失 | 数据集元数据保留；**文件下载返回 0 字节**（`services/svc-datasets/src/store/pg-file-store.ts:98`） |
| 租户上下文 | 不涉及 | `datasets` 在 009 里有 `org_rid` 与 RLS；分支/事务/文件三表只有 `org_rid` 列、没有策略；数据集 store 从不设置 `app.org_rid` |

关于 `/tmp/openfoundry-data`：它是**本体与对象**的默认文件存储路径（`services/svc-ontology/src/store/ontology-store.ts:64`、`services/svc-objects/src/store/object-store.ts:308`、`packages/permissions/src/authorization-store.ts:82`），不是数据集的。数据集在无 `DATABASE_URL` 时没有任何文件持久化，所以「demo 模式下数据集重启后还在」这个预期不成立；AGENTS.md 里「两个 checkout 共享 `/tmp/openfoundry-data`」的风险只影响本体、对象与授权。

**全异步带来的调用约定**：`DatasetReadWriteStore` 的每个方法都声明为 `T | Promise<T>`（`services/svc-datasets/src/store/dataset-store.ts:62-84`），文件面另有 `FileReadWriteStore`（`services/svc-datasets/src/routes/v2/files.ts:16-27`），注释直接写明「不 await 的读取在内存测试里全绿，在 Postgres 上会把 pending promise 当 200 发出去」。v1 路由与文件路由确实逐个 `await`（`services/svc-datasets/src/routes/v1/datasets.ts:65`、`services/svc-datasets/src/routes/v2/files.ts:102`）。

但 v2 的数据集/分支/事务路由是反例：它们把 store 声明成同步的 `DatasetStore`（`services/svc-datasets/src/routes/v2/datasets.ts:20`），而 `server.ts` 用一个 cast 把 `PgDatasetStore` 塞了进去（`services/svc-datasets/src/server.ts:162`），于是这些处理器里 `createDataset`/`getDataset`/`listBranches` 都没有 `await`（`services/svc-datasets/src/routes/v2/datasets.ts:45`、`:57`、`:83`）。按 Node/Fastify 的确定行为推断（两处机制已单独验证：展开 Promise 抛 `TypeError`、全 `undefined` 字段序列化为 `{}`、async 处理器直接 `return promise` 会被摊平）：

- `GET /api/v2/datasets`：`filterDiscoverable(request, listDatasets(), ...)` 对 Promise 做展开，抛 `TypeError`，500。
- `GET /api/v2/datasets/:rid` 与 `POST /api/v2/datasets`：`serializeDataset(promise)` 得到全 `undefined` 对象，响应体为 `{}`（POST 仍返回 201，且数据其实已经写进 PG，客户端拿不到 RID）。
- `GET .../branches`：`paginateArray` 对 Promise 调 `.slice`，500。
- 直接 `return datasetStore.createBranch(...)` / `openTransaction(...)` 的处理器反而正常，因为 async 函数会摊平返回的 thenable。
- `deleteDataset`/`deleteBranch` 没有 await，`NOT_FOUND` 会变成 unhandled rejection（已验证：处理器内的浮动 rejection 会触发 `unhandledRejection`，204 照常返回）。

（以上为静态可判定加机制验证；未在真实 Postgres 上端到端复现整条 HTTP 路径。）

### 8.8 connectivity 与 pipeline：数据集的两个附加面

**connectivity 服务 20 个操作里的 18 个**，按资源分成四组：连接 7 个（创建、读取、`getConfiguration`、`getConfigurationBatch`、`updateExportSettings`、`updateSecrets`、`uploadCustomJdbcDrivers`）、文件导入 5 个（创建、读取、列表、替换、删除）、表导入 5 个（同五项）、虚拟表 1 个（创建）。未服务的两个是 `fileImports/{rid}/execute` 与 `tableImports/{rid}/execute`：它们要返回一个构建 RID，而这里没有构建系统也没有连接器运行时（`services/svc-datasets/src/routes/v2/connectivity.ts:34-49`）。

三处实现细节值得记住：

- **preview 门是插件级的**：`onRequest` 钩子注册在 connectivity 插件内部，因此只保护这个插件注册的路由，别的路由不受影响；不传 `preview=true` 一律 `ApiFeaturePreviewUsageOnly`（`services/svc-datasets/src/routes/v2/connectivity.ts:79-83`）。models 段用的是同一个模式（`services/svc-aip/src/routes/v2/models.ts:83-86`）。
- **驱动 jar 走原始字节**：为 `application/octet-stream` 单独注册解析器，上限 64 MiB，并校验 ZIP 魔数（`services/svc-datasets/src/routes/v2/connectivity.ts:86-90`、`:225`、`:242`）。
- **连接密钥只写不读**：`SecretBox` 用 AES-256-GCM、每条一个随机 12 字节 IV，存成 `iv.tag.ciphertext` 的 base64；API 只回密钥名，不回明文，所以没有 `open()`。`CONNECTIVITY_SECRETS_KEY` 缺失时，内存 store 用一次性随机密钥（反正进程结束就没了），Pg store 则拒绝写入——因为重启后它再也解不开（`services/svc-datasets/src/connectivity/secrets.ts:1-44`）。

connectivity 也是双模的：`MemoryConnectivityStore` 与 `PgConnectivityStore`，后者构造时可以带 `orgRid` 并在事务里 `set_config('app.org_rid', …)`（`services/svc-datasets/src/connectivity/pg-store.ts:55-63`）；但 `server.ts` 只传了 pool 与 `SecretBox`，没传 `orgRid`（`services/svc-datasets/src/server.ts:70`），所以它同样依赖超级用户绕过 RLS。它的契约测试是全仓库少见的「两套 schema 都跑」的范例：用 PGlite 分别加载 `db/migrations/*.sql` 与 `scripts/migrate.sql`，再对同一个 `PgConnectivityStore` 跑完整契约（`services/svc-datasets/tests/connectivity-contract.test.ts:60-100`）。

**pipeline 的边界更窄**。`svc-datasets` 拥有 `/api/v2/pipelines*` 路由与 `PipelineStore`/`PgPipelineStore`，而 `@openfoundry/pipeline` 是一个纯函数式的行变换引擎：`dag.ts` 做拓扑排序与环检测，`executor.ts` 只 import `@openfoundry/rid`、自己的类型和 `dag.js`，不碰任何 store（`packages/pipeline/src/executor.ts:1-12`）。执行一次 pipeline 的路径是：从 `inputDatasets[0]` 取源对象类型名（`services/svc-datasets/src/routes/v2/pipelines.ts:523`），按 `ontologyRid` 解析出本体（缺省时要求「恰好一个本体声明该类型」，否则报错），用 HTTP 从 svc-objects 拉对象（`OBJECTS_SERVICE_URL` 默认 `http://localhost:8082`，`ONTOLOGY_SERVICE_URL` 默认 `http://localhost:8081`，见 `services/svc-datasets/src/routes/v2/pipelines.ts:20-23`），交给 executor 跑步骤，最后**为了拿到输出行再本地重放一遍步骤**，因为 executor 只返回 `PipelineRun` 元数据不返回数据（`services/svc-datasets/src/routes/v2/pipelines.ts:588`）。

一个容易误解的点：`PipelineDefinition.outputDataset` 只是字符串元数据，`POST /pipelines/{rid}/execute` 把结果行直接放进 HTTP 响应，**从不写入数据集或文件 store**。所以 pipeline 目前不能产出数据集，只是「读对象 → 变换 → 返回」。内存模式还会在启动时 seed 3 条默认 pipeline（`services/svc-datasets/src/server.ts:80-82`、`services/svc-datasets/src/store/pipeline-store.ts:58`），Pg 模式不 seed。

### 8.9 迁移体系：两条路径、一次漂移、一类无法被文本测试抓住的错

仓库有两套 schema 来源，它们的加载方式完全不同：

```mermaid
flowchart TD
    A["docker compose up"] --> B["Postgres 初始化空库"]
    B --> C["加载 scripts/migrate.sql"]
    C --> D["建 42 张表 + 种子 _migrations 001-008"]
    E["pnpm db:migrate"] --> F["scripts/migrate.js"]
    F --> G["runMigrations 读 db/migrations 字典序"]
    G --> H{"_migrations 里有记录"}
    H -->|有| I["跳过"]
    H -->|无| J["单文件事务内执行并记录"]
    D --> K["数据库状态 A"]
    I --> K
    J --> K
    K --> L{"来源组合"}
    L -->|仅 Compose| M["核心表无 org_rid 与 RLS"]
    L -->|仅 db:migrate| N["50 张表 + RLS"]
    L -->|Compose 后再 db:migrate| O["009 失败 42P01"]
```

`scripts/migrate.sql` 是 Compose 的 init 脚本（`docker-compose.yml:14` 把它挂到 `/docker-entrypoint-initdb.d/01-schema.sql`），它自己建表、自己开 RLS，最后**把 001–008 号迁移标记为已应用**（`scripts/migrate.sql:961-971`），理由是「schema 已经完整」。`pnpm db:migrate` 走 `scripts/migrate.js`，默认连接串写死为 `postgresql://openfoundry:openfoundry@localhost:5432/openfoundry`（`scripts/migrate.js:10-14`），执行器建 `_migrations` 表、按文件名排序、每个文件一个事务（`packages/db/src/migrator.ts:18-54`）。`scripts/migrate-rls.sql` 是第三份、也是没人引用的历史文件：仓库里没有任何脚本或文档引用它（grep 为空），且它用的会话变量是 `app.current_org_rid`，与 009 的 `app.org_rid` 不同。

漂移点（按 `CREATE TABLE` 统计：Compose 42 张、`db/migrations` 50 张不同表——共 51 条建表语句，`interface_types` 在 001 与 004 各建一次）：

| 类别 | 仅 `scripts/migrate.sql` | 仅 `db/migrations` | 后果 |
|------|--------------------------|---------------------|------|
| 表 | `links`（另有 `_migrations`，由 `runMigrations` 运行期自建） | `aip_conversations`、`aip_messages`、`aip_embeddings`、`aip_generated_functions`、`pipelines`、`pipeline_runs`、`search_index`、`roles`、`role_assignments`、`permission_grants` | Compose 库上 `PgPipelineStore` 与 `PgEmbeddingStore` 的查询会 `42P01` |
| 租户列 | 仅 `users`/`groups`/`ml_*`/`connectivity_*` 有 `org_rid` | 另外给 35 张表补 `org_rid`（009） | Compose 库上核心表无租户列 |
| 列（示例） | `datasets.data`；`objects.object_type`、`ontology_rid`；`sentinel_notifications.acknowledged`、`metadata` | `objects.embedding`、`tsv`；`object_types.embedding`、`tsv`、`implements_interfaces`；`users.platform_roles`、`group_ids`、`marking_ids` | 同一份 store 代码在两套库上命中不同列 |
| 列约束 | `datasets.path` 可空 | `datasets.path` 为 `TEXT NOT NULL`（`db/migrations/001_initial.sql:290`） | `PgDatasetStore` 在迁移库上建数据集失败 |

> 文档与实现不一致：AGENTS.md:173 与 GUIDE.md:915 都说「只有 `db/migrations` 加 `org_rid` 和 RLS」。实际 Compose 脚本已经给 `users`、`groups`（`scripts/migrate.sql:56`、`:85`）以及 `ml_resources`、`ml_model_versions`、五张 `connectivity_*` 表写了 `org_rid` 与 RLS 策略（`scripts/migrate.sql:793-951`）。准确的表述是：核心本体与数据集表（`ontologies`、`object_types`、`objects`、`datasets` 等）的 `org_rid` 与 RLS 只在 `db/migrations/009_multi_tenancy.sql` 里。

**两条路径叠加会直接失败**。因为 Compose 把 001–008 标记为已应用，`pnpm db:migrate` 对 Compose 初始化过的库只会执行 009 及以后；而 009 第一条核心表的 `ALTER TABLE roles ADD COLUMN …` 就会命中 Compose 根本没有的 `roles` 表。把仓库里的 SQL 原样加载进 PGlite 复现的结果是：

```
001..008 skipped (marked applied by compose)
009_multi_tenancy.sql FAILED: 42P01 relation "roles" does not exist
```

单文件事务意味着 009 整体回滚，迁移停止，库停留在「Compose schema + 无 009」的状态。这不是理论风险：只要有人先 `docker compose up`、再 `pnpm db:migrate` 补迁移，就会遇到。

**RLS 怎么随迁移进来**：009 先建默认组织 `org:default`，再逐表 `ADD COLUMN IF NOT EXISTS org_rid TEXT NOT NULL DEFAULT 'org:default' REFERENCES organizations(rid)`（`db/migrations/009_multi_tenancy.sql:37-86`），然后对每张表 `ENABLE ROW LEVEL SECURITY` + `CREATE POLICY tenant_isolation … USING (org_rid = current_setting('app.org_rid', true))` + `FORCE ROW LEVEL SECURITY`（`db/migrations/009_multi_tenancy.sql:129-133`、`:165-170`）。会话变量的名字就是契约：`app.org_rid`。`PgModelStore` 与 `PgConnectivityStore` 都会在事务里 `set_config('app.org_rid', $1, true)`（`services/svc-aip/src/models/pg-model-store.ts:76-94`、`services/svc-datasets/src/connectivity/pg-store.ts:55-63`）；`PgDatasetStore` 与 `PgPipelineStore` **不设置**，而 `datasets` 与 `pipelines` 两张表都有 RLS，因此它们只在连接角色是超级用户（绕过 RLS）时才能工作；`PgFileStore` 的 `dataset_files` 没有策略，但它的写入依赖一个已存在的 dataset 行。`PgObjectStore` 也设 `app.org_rid`（`services/svc-objects/src/store/pg-object-store.ts:172`）。

> 文档与实现不一致：GUIDE.md:916 说 RLS「尚未端到端验证」，理由是 `pg-object-store.ts` 设置的是 `app.current_org_rid`。该文件现在设置的是 `app.org_rid`（`services/svc-objects/src/store/pg-object-store.ts:172`），这条理由已过时；真正未闭环的是数据集 store 完全没有租户上下文。

**「SQL 文本对不上 schema」这一类错**，仓库已经付过一次代价：`PgFileStore.putFile` 的 `ON CONFLICT` 一直写 `updated_at = NOW()`，而 `dataset_files` 从没有这一列，于是每次覆盖上传都 `42703`，内存测试全绿。修复方式不是改文本，而是加迁移（`db/migrations/010_dataset_file_updated_at.sql:1-24`，回填用 `created_at` 而不是 `NOW()`，理由写在注释里）并加三类回归测试：store 往返、两套 schema 都有该列、路由确实 await 异步 store（`services/svc-datasets/tests/pg-file-write-time.test.ts:1-16`）。

同一类错今天仍然存在于 `PgDatasetStore`，而且**两套 schema 各错一处**。把 `db/migrations` 全量、`scripts/migrate.sql` 全量分别灌进 PGlite，再执行 store 的真实语句：

| store 语句 | Compose schema | `db/migrations` schema |
|------------|----------------|------------------------|
| `createDataset` 的 `INSERT INTO datasets (rid, name, description, parent_folder_rid)` | 成功 | 失败 `23502`：`path` 为 `NOT NULL` 且 store 从不写它 |
| `putFile` 的 upsert（带 `updated_at`） | 成功 | 成功 |
| `putFile` 传空字符串 `transactionRid` | 失败 `23503`：`dataset_files_transaction_rid_fkey` | 失败 `23503`（同一外键） |
| `commitTransaction` 的 `UPDATE … committed_at = NOW()` | 失败 `42703`：列不存在 | 失败 `42703`：列不存在 |

`committed_at` 在两套 schema 里都不存在（`db/migrations/003_phase1_tables.sql:74-84` 只有 `created_at`/`updated_at`），而 `PgDatasetStore` 的 `TransactionRow` 却声明了它并在 commit 时写入（`services/svc-datasets/src/store/pg-dataset-store.ts:36-44`、`:273-279`）。没有任何测试在真实数据库上执行 `PgDatasetStore`——这是它能活到现在的原因。相比之下 `PgFileStore` 至少有一个用 `FakePool` 断言 SQL 文本的测试，`PgConnectivityStore` 与 `PgModelStore` 则各有跑真库的契约测试。

**models 表为什么能两边一致**：`db/migrations/012_models.sql` 被逐字复制进 `scripts/migrate.sql`（`scripts/migrate.sql:793-850` 附近），`services/svc-aip/tests/models-pg-schema.test.ts:19-21` 用 `readFileSync` 断言 `migrate.sql` 的文本包含整个迁移文件——这是唯一一处用「文本包含」而不是「执行」来守的约束，因为它的目的就是防止复制时漏改。真正证明 store 在两套 schema 上都能工作的是 `models-contract.test.ts`：它为两套来源各建一个 schema（`db/migrations` 只取 `001_initial.sql` + `012_models.sql`，见 `services/svc-aip/tests/models-contract.test.ts:76-83`），用 `MODELS_TEST_DATABASE_URL` 决定是否启用，CI 里由 postgres service 提供（`.github/workflows/ci.yml:44-59`）。

### 8.10 AIP：models v2 与一层演示型 AI 接口

`svc-aip` 里有两套完全不同的东西，先分清：`/api/v2/models*` 是对 Foundry `models` 段的对象建模，`/api/v2/aip/*` 是平台自造的对话接口。

**models：32 个操作里的 8 个。** 已服务的是 Model create/get、`promoteVersion`、Model Version create/get/list、Model Studio create/get；其余 24 个（live deployments 7 个、model functions 3 个、experiments 6 个、model studio trainers 2 个、config versions 4 个、launch 与 runs 2 个）一律不注册，理由逐条写在模块注释里：它们都需要一个「能加载 dill 序列化函数并执行推理」的运行时，或一个训练/构建系统，而本仓库两个都没有（`services/svc-aip/src/routes/v2/models.ts:26-75`）。所有已服务操作都是 Foundry 的 preview 端点，因此插件级 `onRequest` 钩子要求 `preview=true`，否则 `ApiFeaturePreviewUsageOnly`（`services/svc-aip/src/routes/v2/models.ts:83-86`）。契约测试还把 24 个未服务路径逐条断言为 404（`services/svc-aip/tests/models-contract.test.ts:188-213`、`:857-863`）。

| 操作 | 方法与路径 | 备注 |
|------|------------|------|
| 创建 Model | `POST /v2/models` | 只返回 `{ rid }`，display name 每文件夹唯一 |
| 读取 Model | `GET /v2/models/{rid}` | 不存在时 `ModelNotFound` |
| 提升版本 | `POST /v2/models/{rid}/promoteVersion` | 复制 `modelApi`/依赖/`modelFiles`，`source` 记为 `promoted` |
| 创建版本 | `POST /v2/models/{rid}/versions` | body 上限 64 MiB（`services/svc-aip/src/routes/v2/models.ts:282`），落在 `master` 分支 |
| 读取版本 | `GET /v2/models/{rid}/versions/{versionRid}` | 版本必须属于该 model |
| 列版本 | `GET /v2/models/{rid}/versions` | 按 `seq` 排序，token 分页 |
| 创建 Studio | `POST /v2/models/modelStudios` | `parentFolderRid` 必须是 folder RID |
| 读取 Studio | `GET /v2/models/modelStudios/{rid}` | 返回 `{ rid, folderRid, createdTime }` |

**模型注册表**：Models 与 Model Studios 共用一张 `ml_resources`，因为「一个显示名在每个文件夹里只能被 Model 或 Studio 用一次」这条规则需要在一个地方用唯一约束表达：`UNIQUE (org_rid, parent_folder_rid, display_name)`（`db/migrations/012_models.sql:23-34`）。版本在 `ml_model_versions`，`model_api`/`conda_requirements`/`backing_repositories`/`model_files`/`source` 都是 JSONB，另有 `seq BIGSERIAL` 专门承载插入顺序——列表用它排序，绝不用 `created_time`，因为同一毫秒内创建的两个版本时间戳相同（`services/svc-aip/src/models/pg-model-store.ts:159-173`）。两张表都带 `org_rid`、RLS 与 `FORCE ROW LEVEL SECURITY`；`PgModelStore` 在构造时可选 `orgRid`，有值就把每条语句包进「BEGIN + set_config + 语句 + COMMIT」的事务，没值就以无租户上下文运行——契约测试证明这种 store 读不到任何行、写入被 `42501` 拒绝（`services/svc-aip/tests/models-contract.test.ts:982-997`）。

**AIP 侧的九个端点**（都在 `/api/v2` 前缀下）：

| 端点 | 作用 | 依赖 |
|------|------|------|
| `POST /aip/query` | 自然语言问答，返回 `{ answer, sources, confidence }` | LLM `chat`，提示词要求 JSON |
| `POST /aip/generate-function` | 由描述生成 TypeScript 函数体，返回 `{ code, apiName, explanation }` | LLM `chat` |
| `POST /aip/embeddings` | 为文本生成并存储向量，返回 `{ embeddings, model }` | LLM `embed` + embedding store |
| `POST /aip/semantic-search` | 向量检索，返回 `{ results: [{ rid, objectType, score, properties }] }` | LLM `embed` + embedding store |
| `GET /aip/models` | 硬编码的 6 个 LLM 模型目录 | 无 |
| `POST /aip/chat` | 多轮对话，可带 `sessionId` | LLM `chat` + 进程内会话 |
| `GET /aip/agents` | 硬编码的 5 个 agent | 无 |
| `GET /aip/sessions/{id}` | 读会话历史 | 进程内会话 |
| `DELETE /aip/sessions/{id}` | 清空会话 | 进程内会话 |

`/aip/query` 的系统提示要求模型「尽量」输出 `{answer, sources, confidence}` 形状的 JSON，服务端用 `tryParseStructuredResponse` 从回复里抓第一个 `{…}` 尝试 `JSON.parse`，失败就退回原始文本并把 `confidence` 填成 0.8（`services/svc-aip/src/routes/v2/aip.ts:111-121`、`:518-533`）。`/aip/generate-function` 只把模型返回的文本 `trim()` 后当作函数体，`apiName` 由描述的前四个词拼成，没有编译或校验（`services/svc-aip/src/routes/v2/aip.ts:157-164`、`:535-546`）。

**chat 的形状**。请求是 `{ messages: [{role, content}], model?, ontologyRid?, sessionId?, tools? }`，响应是 `{ message: {role: "assistant", content}, model, sessionId, usage: {promptTokens, completionTokens} }`（`services/svc-aip/src/routes/v2/aip.ts:281-338`）。多轮靠进程内的 `sessions` Map：首次请求生成 `ses-<ms>-<rand>`，之后把客户端送来的消息追加进历史，再连同一条 system 提示一起交给 LLM，最后把助手回复也追加进去（`services/svc-aip/src/routes/v2/aip.ts:289-324`）。会话 1 小时过期，后台每 10 分钟清理一次（`services/svc-aip/src/routes/v2/aip.ts:70-79`）。`GET /aip/sessions/:id` 与 `DELETE /aip/sessions/:id` 读写的是同一个 Map。

这里有一个必须点明的事实：`conversationStore` 被注入、被解构成 `_conversationStore` 之后**从未使用**（`services/svc-aip/src/routes/v2/aip.ts:93`）。`PgConversationStore` 与 `aip_conversations`/`aip_messages` 两张表因此是死代码：`DATABASE_URL` 下会话依然只存在进程内存，重启即失。请求体里的 `model` 字段同样被忽略（`_requestedModel`），客户端无法按请求切换模型——提供方在 `createServer` 时就固定了。

**响应是非流式的**。`LlmClient` 只有 `chat()` 与 `embed()` 两个返回完整结果的方法（`services/svc-aip/src/llm/llm-client.ts:29-35`），Ollama 客户端显式传 `stream: false`（`services/svc-aip/src/llm/ollama-client.ts:52`），服务里没有任何 SSE 或 `text/event-stream`。控制台是一次性 fetch 后整体渲染。

**提供方选择与 mock 降级**。两个地方各写了一份检测逻辑，顺序不同：

| 提供方 | `config.ts` 的检测顺序 | `create-client.ts` 的检测顺序 | embed 支持 |
|--------|------------------------|-------------------------------|------------|
| GLM（智谱） | 不检测，只能 `LLM_PROVIDER=glm` 显式指定 | 第 1 位：`GLM_API_KEY`（`services/svc-aip/src/llm/create-client.ts:36`） | OpenAI 兼容 |
| OpenAI | 第 1 位：`OPENAI_API_KEY` | 第 2 位 | 有 |
| Anthropic | 第 2 位：`ANTHROPIC_API_KEY` | 第 3 位 | **无**，`embed()` 直接抛错（`services/svc-aip/src/llm/anthropic-client.ts:103-111`） |
| Ollama | 第 3 位：`OLLAMA_URL` | 第 4 位 | 有 |
| mock | 兜底 | 兜底 | 返回 4 维确定性向量 |

`server.ts` 总是把 `config.llmProvider` 显式传给工厂（`services/svc-aip/src/server.ts:63-69`），所以实际生效的是 `config.ts` 的顺序：`LLM_PROVIDER` > OpenAI > Anthropic > Ollama > mock。`create-client.ts` 里 GLM 优先的那段自动检测只在「调用方不传 provider」时才会走到，而本服务从不这样调用。

> 文档与实现不一致：`services/svc-aip/src/llm/create-client.ts:53-56` 的注释只列了 OpenAI / Anthropic / Ollama 三条自动检测，代码第 36 行却先判 `GLM_API_KEY`。

mock 不是「返回固定字符串」那么简单：`MockLlmClient` 会按关键词生成表格化回答（客户数、待办工单、低库存、营收汇总等），并且会**真的去拉 svc-objects 的数据**——依次尝试 `localhost:8080` 与 `localhost:8085` 的 `/api/v2/ontologies`（后者实际是 svc-datasets 的端口，多半 404；未验证），再调 `objectSets/loadObjects`，2 秒超时，失败则回落到内置的虫害防治样例数据（`services/svc-aip/src/llm/mock-client.ts:25-72`）。这让 demo 在没有任何 API key 时也能回答「有多少客户」这类问题，代价是答案取决于本机是否恰好跑着 svc-objects，且回落的样例数据是印尼虫害防治行业的硬编码内容。

**agents 与 embeddings**。`GET /aip/agents` 返回 5 个硬编码 agent（ontology-explorer、scheduling-assistant、inventory-manager、revenue-analyst、code-generator），没有存储、没有执行入口（`services/svc-aip/src/routes/v2/aip.ts:343-413`）。`POST /aip/embeddings` 调 `llmClient.embed(texts)`，给每条文本生成 `ri.aip.main.text.*` 作为 `sourceRid`，再 `storeBatch` 落库（`services/svc-aip/src/routes/v2/aip.ts:181-199`）；`POST /aip/semantic-search` 把查询向量化后交给 store，内存实现做余弦相似度并按分数排序，Pg 实现用 pgvector 的 `<=>` 算子（`services/svc-aip/src/store/pg-embedding-store.ts:91-129`）。两者都按 `ontologyRid` 过滤，因为对象类型的 API name 只在一个本体里唯一。

**与本体如何关联**：分两条路，不要混为一谈。

- mock 路径：`buildChatSystemPrompt` 把 7 个对象类型（Customer、Technician、ServiceJob、TreatmentProduct、Invoice、Vehicle、Schedule）及其字段、关系、IDR 货币硬编码进 system prompt（`services/svc-aip/src/routes/v2/aip.ts:487-516`）。它**不读本体服务**，`ontologyRid` 只是被拼进提示词的一句话。`/aip/query` 的系统提示同样硬编码（`services/svc-aip/src/routes/v2/aip.ts:460-485`）。
- 真实 LLM 路径：`createLlmClient` 给 OpenAI/Anthropic/Ollama/GLM 都套一层 `OntologyAwareLlmClient`，它在首次调用时 `GET /api/v2/ontologies` 与每个本体的 `/fullMetadata`，把对象类型、属性、可空性、主键与 link 关系渲染成 Markdown 表格注入 system prompt，缓存 5 分钟（`services/svc-aip/src/llm/real-client.ts:20-83`、`:118-144`）。也就是说，只有配置了真实 key 时，「AI 知道你的本体长什么样」才成立。

> 实现与安全模型的落差：`/api/v2/aip/*` 没有注册任何权限钩子——`svc-aip` 全服务没有 `registerAuthorization`、没有 `requirePermission`（grep 为空），而 `svc-datasets` 有 42 处 `requirePermission`。AIP 端点只依赖网关这一层，网关在 demo 默认配置下不校验任何请求。

### 8.11 设计决策与理由

1. **内存模式是一等公民，不是降级路径。** `DatasetReadWriteStore` 的返回类型写成 `T | Promise<T>`，路由按接口编程、两种后端都能注入（`services/svc-datasets/src/store/dataset-store.ts:53-84`）。取舍：demo 与全部单元测试不需要 Postgres，启动零依赖；代价是 Pg 路径长期缺少「跑真库」的测试，`committed_at` 与 `path NOT NULL` 两个缺陷就是这么活下来的。同一决策在 `svc-aip` 上做得更彻底——`ModelStore` 的所有方法在两种实现里都是 async（`services/svc-aip/src/models/model-store.ts:8-16`），所以不存在「同步测试掩盖异步实现」的可能。

2. **文件元数据与内容分开存。** 元数据进 `dataset_files`，内容留进程内 `contentCache`，注释写明未来会换成 S3/GCS（`services/svc-datasets/src/store/pg-file-store.ts:34-44`）。取舍：把「内容放哪」收敛到一个 Map，换对象存储时只动一个类；代价是 PG 模式重启后下载返回 0 字节的 200——一个静默的数据丢失，而不是显式错误。

3. **事务只做元数据状态机，不覆盖文件写入。** 这样不需要为每个路径维护版本目录，也不需要 MVCC；代价是与 Foundry 的写入语义不一致（见〈写路径〉）。代码选择了「诚实失败」而不是「假装支持」：v1 的 branch/transaction 作用域参数被显式拒绝（`services/svc-datasets/src/routes/v1/files.ts:50-65`），文件列表只支持 `pathPrefix`，注释解释了为什么 branch 与 transaction 区间过滤会给出错误答案（`services/svc-datasets/src/routes/v2/files.ts:70-78`）。

4. **v1 与 v2 各写一份 serializer，即使字段完全相同。** `File` 在两个版本里都是 `path`/`transactionRid`/`sizeBytes`/`updatedTime` 四个字段，仍分别投影（`services/svc-datasets/src/routes/v1/serializers.ts:100-126`）；`Branch` 的分歧更明显——v1 的键是 `branchId`，值是分支名（`services/svc-datasets/src/routes/v1/serializers.ts:51-53`）。取舍：重复代码与两处维护；换来的是「v2 改一个字段不会悄悄改变 v1 契约」，以及像 `contentType` 这种 store 有、Foundry 没有的字段可以只在一侧省略。

5. **AIP 允许无 key 运行，但 mock 的能力边界写死在代码里。** 没有 `OPENAI_API_KEY`/`ANTHROPIC_API_KEY`/`OLLAMA_URL` 时工厂返回 `MockLlmClient`，它靠关键词分支和可选的真实对象读取作答。取舍：`bash start.sh` 之后 AIP 页面立刻可用，演示不需要任何外部账号；代价是 mock 的答案与真实 LLM 不可比，而且它的 4 维 embedding 与 pgvector 的 `vector(1536)`（`db/migrations/005_phase2_aip.sql:30`）根本不兼容——一旦同时开 `DATABASE_URL` 且无 key，`/aip/embeddings` 在表存在时会因维度不符写库失败；Compose schema 下更早一步死在表不存在（静态可判定：`PgEmbeddingStore.store` 直接把 4 个浮点数拼成向量串写入，`services/svc-aip/src/store/pg-embedding-store.ts:62-80`）。

6. **models 表要双 schema 一致性测试，其他表不要。** `ml_resources`/`ml_model_versions` 是唯一被逐字复制进 `scripts/migrate.sql` 并加文本断言的表（`services/svc-aip/tests/models-pg-schema.test.ts:19-21`），因为它们是最后加的、且 store 是纯 async 的，作者用「文本一致 + 双后端契约测试」把漂移堵死。取舍：多一份要同步维护的 SQL 与一个只在 CI 有 Postgres 时才跑的测试（`MODELS_TEST_DATABASE_URL` 未设置时整个 Postgres 分支 `skipIf`）；换来的是 models 段与 connectivity 段成为全仓库仅有的两个「两套 schema 都真的跑过 store」的数据面。

7. **connectivity 只服务 18/20，未服务的 2 个用注释记录而不是返回假 RID。** `File Import execute` 与 `Table Import execute` 需要返回一个构建 RID，而这里既没有构建系统也没有连接器运行时，所以宁可不注册（`services/svc-datasets/src/routes/v2/connectivity.ts:34-49`）。取舍：API 覆盖率低 10%，但每个返回的字段都对应真实状态；同类决策在 `models`（24/32 不服务）和 `v1 files`（upload 不服务）上重复出现。

### 8.12 已知限制与坑

1. **Compose 初始化后再跑 `pnpm db:migrate` 必然失败。** Compose 把 001–008 标记为已应用（`scripts/migrate.sql:963-971`），009 却 `ALTER TABLE roles`——`roles` 只在 `db/migrations` 里创建。PGlite 复现为 `42P01 relation "roles" does not exist`。触发条件：任何「先 `docker compose up`、再补迁移」的部署。
2. **`PgDatasetStore` 在两套 schema 上都不完整。** Compose 下 `commitTransaction` 报 `42703 committed_at`；`db/migrations` 下 `createDataset` 报 `23502 path NOT NULL`。两者都已用仓库原样 SQL 在 PGlite 上复现。根因是没有任何测试在真实数据库上执行 `PgDatasetStore`。
3. **PG 模式上传文件必失败。** 路由把 `transactionRid` 硬编码为空字符串（`services/svc-datasets/src/routes/v2/files.ts:133`），而 `dataset_files.transaction_rid` 有外键指向 `dataset_transactions`，两套 schema 都返回 `23503`。
4. **`PgFileStore` 的 `updated_at` 历史缺陷与其修复。** AGENTS.md:175 记录的 `42703` 已由 `010_dataset_file_updated_at.sql` 修掉，回归测试同时守住「两套 schema 都有列」与「回填用 `created_at`」。这条现在只在读旧笔记时出现，代码已不含该缺陷。
5. **v2 数据集路由在 PG 模式下不可用。** `services/svc-datasets/src/server.ts:162` 的 cast 抹掉了异步性，`services/svc-datasets/src/routes/v2/datasets.ts` 里 5 处调用没有 await；`GET /datasets`、`GET .../branches` 会 500，`POST/GET /datasets/:rid` 返回 `{}`，删除失败会变成 unhandled rejection。v1 与文件路由没有这个问题。
6. **数据集 store 没有租户上下文。** 009 给 `datasets`/`dataset_branches`/`dataset_transactions`/`dataset_files` 都加了 `org_rid`，但只为 `datasets` 开了 RLS（其余三表没有策略）；数据集 Pg store 都不设 `app.org_rid`。以非超级用户连接时，读不到行、写被 `WITH CHECK` 拒绝；demo 之所以正常，是因为 Compose 的 `openfoundry` 是超级用户，天然绕过 RLS。
7. **文件内容不持久。** PG 模式下 `getFile` 找不到缓存内容时返回 `new Uint8Array(0)`（`services/svc-datasets/src/store/pg-file-store.ts:98`），`listFiles` 同样（`:120`）。重启后列表看起来正常、下载是空文件。
8. **`/tmp/openfoundry-data` 是固定路径且与数据集无关。** 它服务本体、对象与授权（`services/svc-ontology/src/store/ontology-store.ts:64` 等），无环境变量可覆盖，两个 checkout 会互相覆盖；数据集在无 `DATABASE_URL` 时纯内存，重启即丢，两者不要混为一谈。
9. **AIP 的能力边界。** 会话只在进程内（`conversationStore` 注入后未使用，`PgConversationStore` 是死代码）；请求里的 `model` 被忽略；没有流式响应；Anthropic 提供方不支持 embeddings（`/aip/embeddings` 与 `/aip/semantic-search` 会 500）；`/aip/agents` 是 5 条硬编码记录；`/api/v2/aip/*` 没有任何权限钩子。`aip_conversations`/`aip_embeddings` 等表在 Compose schema 里根本不存在，因此 PG 模式的 AIP 会话/向量功能只在 `pnpm db:migrate` 建出的库上才有可能工作。
10. **connectivity 的未实现操作与受限参数。** 20 个操作里 `fileImports/{rid}/execute`、`tableImports/{rid}/execute` 不服务（404）；`parentFolderRid` 与虚拟表的 `parentRid` 只校验格式不校验存在；虚拟表的 markings 不查目录；`snowflake` 的 `externalOauth` 与 `databricks` 的 `workflowIdentityFederation` 被拒绝（`services/svc-datasets/src/routes/v2/connectivity.ts:51-66`）。
11. **v1 数据集可能无法表示。** v2 允许创建没有 `parentFolderRid` 的数据集，而 v1 `Dataset.parentFolderRid` 必填，因此读取这类数据集会返回 500 `DatasetNotRepresentableInV1`（`services/svc-datasets/src/routes/v1/serializers.ts:66-81`）。种子脚本都带了 folder RID，所以 demo 不会碰到。
12. **PG 模式允许重名数据集。** 两套 schema 的 `datasets.name` 都没有唯一约束（`scripts/migrate.sql:343-354`、`db/migrations/003_phase1_tables.sql:13-14`），`PgDatasetStore.createDataset` 里把 `23505` 翻译成 `CONFLICT` 的分支（`services/svc-datasets/src/store/pg-dataset-store.ts:70-77`、`:113-118`）因此永远不会触发；同一份 API 在内存模式拒绝重名、在 Pg 模式接受重名。PGlite 实测两套 schema 都能插入两行同名数据集。
13. **迁移文件的编号与幂等陷阱。** 有两个 `011_` 前缀（`011_aip_embedding_ontology.sql`、`011_user_platform_roles.sql`），字典序执行时都跑，但排序语义不直观；`interface_types` 在 001 与 004 里各建一次，004 用的是 `CREATE TABLE IF NOT EXISTS` 且没有补 `display_name` 列，于是该列在全新迁移库上并不存在（目前没有 store 写它，属于潜在坑）。另有一个无人引用的 `scripts/migrate-rls.sql`，其策略用的是 `app.current_org_rid`，与 009 的 `app.org_rid` 不一致。

### 8.13 关键代码位置

- `services/svc-datasets/src/store/dataset-store.ts:62` — `DatasetReadWriteStore`，`T | Promise<T>` 的调用契约
- `services/svc-datasets/src/store/pg-dataset-store.ts:275` — commit 写 `committed_at`，两套 schema 都没有该列
- `services/svc-datasets/src/store/pg-file-store.ts:34` — 元数据入 PG、内容留内存的设计与理由
- `services/svc-datasets/src/routes/v2/datasets.ts:45` — v2 路由不 await 异步 store 的反例
- `services/svc-datasets/src/routes/v1/serializers.ts:66` — v1 `parentFolderRid` 必填导致的 500
- `services/svc-datasets/src/routes/v1/files.ts:50` — 未服务的 branch/transaction 作用域参数被显式拒绝
- `services/svc-datasets/src/routes/v2/connectivity.ts:40` — connectivity 18/20 的清单与未服务原因
- `scripts/migrate.sql:963` — Compose 把 001–008 标记为已应用，导致 009 无法在其上执行
- `db/migrations/009_multi_tenancy.sql:129` — RLS 策略与会话变量 `app.org_rid`
- `db/migrations/010_dataset_file_updated_at.sql:14` — `updated_at` 修复与回填理由
- `services/svc-aip/src/routes/v2/models.ts:29` — models 8/32 与 24 个未服务操作的理由
- `services/svc-aip/src/routes/v2/aip.ts:93` — 注入却未使用的 `conversationStore`
- `services/svc-aip/src/llm/real-client.ts:20` — 真实 LLM 路径按本体元数据注入 schema

---

## 第 9 章 前端应用

本章导读：OpenFoundry 的浏览器侧是三个互不共享路由、认证与构建产物的 Vite 单页应用——`app-console` 是主控制台，`app-workshop` 是低代码页面构建器，`app-docs` 是静态文档站；它们唯一共用的是 `packages/ui/*` 下的组件、图标与 token。本章回答三个问题：控制台如何把 27 条路由、约 80 个 `fetch` 调用点和一套 Blueprint v5 主题装配成产品；三个应用各自的边界在哪；前端与服务之间哪些契约是错位的。请求进入网关之后的鉴权、代理与数据面实现，分别见〈身份、权限与多租户安全〉与〈总体架构〉，本章只写到 HTTP 边界。

### 9.1 职责与边界

前端负责四件事：路由与页面编排、会话与令牌持有、把服务返回的 RID 呈现为可导航资源、把服务的错误信封翻译成可读文案（`apps/app-console/src/lib/apiError.ts:83`，它消费 `errorName` 与 `parameters` 而非 `message`）。

前端不负责权限判定、数据校验与业务数据持久化。所有页面只经 `svc-gateway` 访问下游：开发态由 Vite 代理 `/api` 与 `/multipass` 两个前缀到 `localhost:8080`（`apps/app-console/vite.config.ts:9-18`），生产态由 nginx 做同样的转发（`deploy/docker/nginx.conf:84-115`）。唯一的浏览器侧持久化是 Workshop 页面与登录会话，二者都写在 `localStorage`。

### 9.2 三个前端应用的定位

| 应用 | 端口 | 技术栈 | 职责 | 质量门禁 |
|---|---|---|---|---|
| `@openfoundry/app-console` | 3000 | React 18、React Router 7、Blueprint 5、Vite 6 | 主控制台，27 条路由 | 7 个 vitest 文件；`tsc --noEmit` |
| `@openfoundry/app-workshop` | 3001 | React 18、React Router 6、Blueprint 5、Vite 5、Storybook 8 | 低代码页面构建器 | `test` 为 `echo 'no tests'`；无 typecheck |
| `@openfoundry/app-docs` | 3001 | React 18、React Router 7、Vite 6，不用 Blueprint | 静态文档站，6 个页面 | 无 test；无 typecheck |

`app-console` 的构建是 `tsc && vite build`（`apps/app-console/package.json:8`），类型错误会挡住构建；`app-workshop` 只有 `vite build`（`apps/app-workshop/package.json:7`），esbuild 不做类型检查，且它没有 `typecheck` 脚本，因此该应用的 TypeScript 错误在整条流水线上无人拦截。`app-docs` 同样只有构建时 `tsc`。

> 端口冲突：`app-workshop` 与 `app-docs` 都声明 3001（`apps/app-workshop/vite.config.ts:7`、`apps/app-docs/vite.config.ts:7`），而根 `dev` 脚本是 `turbo run dev`（`package.json:9`），两个应用会被同时拉起，后启动者会被 Vite 挪到下一个空闲端口（Vite 默认行为，未实测）。`start.sh` 只启动 `app-console`，所以日常演示不会暴露这个冲突。

### 9.3 app-console 的代码结构

下表路径均相对 `apps/app-console/`：

| 路径 | 作用 |
|---|---|
| `src/main.tsx` | 入口：Blueprint 样式、`installAuthFetch()`、`createRoot` |
| `src/App.tsx` | 外壳：`AuthProvider`、导航栏、侧边栏、`ErrorBoundary` |
| `src/routes.tsx` | 唯一路由表，25 个懒加载页面 + 首页/登录 |
| `src/config.ts` | `API_BASE_URL` 的唯一来源 |
| `src/lib/authFetch.ts` | 全局 `fetch` 拦截器与令牌读取 |
| `src/lib/session.ts` | 会话持久化、JWT 载荷解析、过期判定 |
| `src/context/AuthContext.tsx` | `TokenManager`、密码登录、PKCE 登录 |
| `src/hooks/useApi.ts` | 通用 GET 取数钩子（超时、重试、错误文案） |
| `src/hooks/useObjectSetQuery.ts` | ObjectSet `loadObjects`/`aggregate` 查询 |
| `src/styles/app.css` | 190 行全局样式，`@layer` 分层与导航栏覆盖 |
| `src/pages/*` | 28 个页面组件（其中 1 个已无路由，见下） |

### 9.4 启动、Provider 与渲染管线

`main.tsx` 的语句顺序本身就是设计：先按序导入四份 Blueprint 样式表（`apps/app-console/src/main.tsx:5-8`），再导入 `app.css`（`:10`），然后调用 `installAuthFetch()`（`:16`），最后才 `createRoot(...).render()`（`:23-29`）。注释说明了原因——拦截器必须在首次渲染前装好，否则任何页面都可能在挂载时发出未带令牌的请求（`apps/app-console/src/main.tsx:14-15`）。

Provider 只有两层：`StrictMode > BrowserRouter > App`，而 `AuthProvider` 在 `App` 内部包住外壳（`apps/app-console/src/App.tsx:174-179`）。没有 Redux、Zustand、React Query 或 SWR——`package.json` 的依赖里没有任何状态库，`AuthContext` 是唯一的 React Context。

```mermaid
flowchart LR
  A[main.tsx] --> B[installAuthFetch 包裹 window.fetch]
  A --> C[BrowserRouter]
  C --> D[AuthProvider]
  D --> E[AppShell 导航栏与侧边栏]
  E --> F[useRoutes 路由表]
  F --> G[页面组件]
  G --> H[useApi 或直接 fetch]
  H --> I[authFetch 拦截器]
  I --> J[Vite 代理或 VITE_API_URL]
  J --> K[svc-gateway 8080]
  K --> L[下游微服务]
```

图中关键点是 `H` 与 `I` 的分离：页面从不显式携带令牌，令牌由被包裹的 `window.fetch` 注入，因此新增页面天然被覆盖。

### 9.5 路由总表

全部 27 条路由都定义在 `apps/app-console/src/routes.tsx:63-98`。除 `/login` 外每条都被 `protect()` 包一层 `ProtectedRoute`（`:50-52`），未登录时重定向到 `/login` 并保留来源位置（`apps/app-console/src/components/ProtectedRoute.tsx:33-35`）。首页 `Dashboard` 是唯一同步导入的页面（`:68`），其余 25 个页面都用 `React.lazy` 切成独立 chunk（`:12-37`）。

| 路径 | 页面组件 | 主要 API | 下游服务 |
|---|---|---|---|
| `/login` | `Login` | `POST /multipass/api/auth/login`、OAuth2 PKCE | svc-multipass |
| `/` | `Dashboard` | `GET /api/v2/admin/stats` | svc-admin |
| `/ontology` | `OntologyExplorer` | `GET /api/v2/ontologies`、`.../:rid/fullMetadata` | svc-ontology |
| `/ontology/:ontologyRid/objects/:objectType` | `ObjectBrowser` | `GET .../objects/:type` | svc-objects |
| `/objects` | `ObjectExplorer` | `loadObjects`、`aggregate`、`search`、`.../links/:lt` | svc-objects |
| `/datasets` | `DatasetList` | `GET/POST /api/v2/datasets` | svc-datasets |
| `/actions` | `ActionExplorer` | `.../actionTypes`、`.../actions/:apiName/apply` | svc-actions |
| `/functions` | `FunctionList` | `GET /api/v2/functions`、`.../:rid/execute` | svc-functions |
| `/monitors` | `MonitorList` | `GET/POST/PUT/DELETE /api/v2/monitors` | svc-sentinel |
| `/compass` | `CompassExplorer` | `.../compass/resources`、`.../children`、`.../search` | svc-compass |
| `/notifications` | `NotificationsPage` | `.../notifications`、`.../:rid/read`、`.../read-all` | svc-sentinel |
| `/webhooks` | `WebhookList` | `GET/POST/PUT/DELETE /api/v2/webhooks` | svc-webhooks |
| `/admin/users` | `UserManagement` | `GET/POST/PUT /api/v2/admin/users` | svc-admin |
| `/admin/groups` | `GroupManagement` | `.../admin/groups`、`.../groupMembers/*` | svc-admin |
| `/admin/audit` | `AuditLog` | `GET /api/v2/admin/audit` | svc-admin |
| `/pipelines` | `PipelineExplorer` | `.../pipelines`、`.../runs`、`.../execute` | svc-datasets |
| `/graph` | `NetworkGraph` | `.../objects?pageSize=100`、逐对 `.../links/:lt` | svc-objects |
| `/workshop` | `Workshop` | `GET /api/v2/ontologies`、`loadObjects` | svc-ontology、svc-objects |
| `/contour` | `Contour` | `.../fullMetadata`、`loadObjects` | svc-ontology、svc-objects |
| `/data-connection` | `DataConnection` | `.../objectTypes`、`.../objects/:type` | svc-ontology、svc-objects |
| `/lineage` | `DataLineage` | `.../objects/:type?pageSize=1` | svc-ontology、svc-objects |
| `/workbook` | `CodeWorkbook` | `loadObjects` | svc-objects |
| `/scenarios` | `Scenarios` | `loadObjects` | svc-objects |
| `/aip` | `AIPChat` | `.../aip/agents`、`.../aip/chat`、`.../aip/sessions/:id` | svc-aip |
| `/data-health` | `DataHealth` | `.../fullMetadata`、`loadObjects`（前端计算健康分） | svc-ontology、svc-objects |
| `/sanocare` | `SanocareDashboard` | `loadObjects`、`aggregate` | svc-ontology、svc-objects |
| `/workshop/edit` | `WorkshopEditor` | `.../objectTypes`、`loadObjects` | svc-ontology、svc-objects |

`apps/app-console/src/pages/PestControlDashboard.tsx` 存在但没有路由：路由文件里只留了一行注释「Archived: PestControlDashboard replaced by SanocareDashboard」（`apps/app-console/src/routes.tsx:21`、`:90`）。它成了死代码，但仍在 `tsc` 的 `include: ["src"]` 范围内参与类型检查（`apps/app-console/tsconfig.json:13`），只是不会被 Vite 打进产物。

几个核心页面的取数方式各有代表性。Object Explorer 是唯一把 ObjectSet 查询语法完整暴露给用户的页面：先取 `fullMetadata` 拿到对象类型与属性，再用 `loadObjects`/`aggregate`/`search` 三种 ObjectSet 请求驱动表格与聚合（`apps/app-console/src/pages/ObjectExplorer.tsx:197-213`、`:355-373`）。Datasets 与 Monitors 是标准的「列表 + 创建对话框」：`useApi` 拉列表，创建用裸 `fetch` 后 `refetch()`（`apps/app-console/src/pages/DatasetList.tsx:28-30`、`:57-67`）。Actions 先取本体再取 `actionTypes`，提交走 `apply` 并展示校验错误。AIP Chat 自己维护消息数组与 session id，三个端点分别是 agents、chat、sessions（`apps/app-console/src/pages/AIPChat.tsx:53-58`、`:930`）。Admin 三页全部落在 `svc-admin`：统计卡读 `/admin/stats`，用户与分组页读写 `/admin/users`、`/admin/groups`，审计页读 `/admin/audit`。Monitors 与 Notifications 虽然 UI 分属两页，数据都来自 `svc-sentinel`（网关把两个前缀都代理到同一服务，`services/svc-gateway/src/routes/v2/monitors.ts:4-6`、`services/svc-gateway/src/routes/v2/notifications.ts:4-6`）。

### 9.6 数据获取与状态管理

控制台没有全局缓存层，取数分三档：

- `useApi<T>(url)`：组件级 GET 钩子，30 秒超时、GET 失败最多重试 2 次（仅 5xx）、按状态码生成人话错误，`refetch()` 靠自增 `tick` 触发（`apps/app-console/src/hooks/useApi.ts:12-18`、`:105-107`、`:172-173`）。16 个页面共 22 处用它。
- `useObjectSetQuery`：把 ObjectSet 的 `loadObjects`/`aggregate` 包装成钩子，`POST` 带 JSON body（`apps/app-console/src/hooks/useObjectSetQuery.ts:52-64`）。
- 裸 `fetch`：写操作（创建、更新、删除、执行）大多由页面直接调用。全仓共 79 个 `fetch` 调用点（不含测试），扣除上面两个钩子与 `AuthContext`、`GlobalSearch` 等封装后，页面里的裸调用集中在 `CompassExplorer`（8）、`WebhookList`（6）、`MonitorList`（6）、`ObjectExplorer`（4）等。

轮询只有两处是自动的：`DataHealth` 每 60 秒重跑全部检查（`apps/app-console/src/pages/DataHealth.tsx:128`、`:563`），`SanocareDashboard` 每 5 分钟重载（`apps/app-console/src/pages/SanocareDashboard.tsx:140`、`:190`）。其余页面依赖用户点「刷新」。这个模式与网关默认的 100 请求/60 秒限流叠加时很敏感——`NetworkGraph` 对每个对象×每个 link type 各发一次请求，正是被限流打成 429 的页面（GUIDE 的已知限制第 3 条，`apps/app-console/src/pages/NetworkGraph.tsx:241`）。

### 9.7 authFetch：一个拦截器覆盖 79 个调用点

设计动机写在文件头注释里：登录后令牌已存下，但约 80 个调用点都直接摸全局 `fetch`，令牌从未上过线；与其把认证客户端穿过每一层，不如在 `window.fetch` 上装一个拦截器，新调用点写下的当天就被覆盖（`apps/app-console/src/lib/authFetch.ts:3-13`）。

拦截器只做四件事：判断目标是否是 OpenFoundry API、取令牌、在缺少 `Authorization` 时补上、透传。判定规则是安全敏感的部分（`:101-116`）：绝对 URL 必须与 `GATEWAY_ORIGIN` 同源且下一字符是 `/`、`?`、`#` 或结尾，避免 `https://gateway.example.com.evil` 被误认；相对 URL 只有 `/api/`、`/multipass/` 两个前缀才加令牌；`/multipass/api/oauth2/` 与 `/multipass/api/auth/login` 两个发令牌的端点被显式排除（`:25-28`）。令牌来源优先是 `AuthContext` 注册的 `TokenManager`（会刷新即将过期的令牌），刷新失败再退回 `localStorage`（`:52-62`）。安装标记用 `Symbol.for("openfoundry.authFetch")` 打在函数对象上而不是模块级布尔量，这样「是否已安装」问的是当前的 `globalThis.fetch`，而不是哪一份模块副本在问（`:124-131`）。

```mermaid
sequenceDiagram
  participant P as 页面
  participant H as useApi
  participant I as authFetch
  participant S as TokenManager
  participant G as svc-gateway
  P->>H: 传入 /api/v2/ontologies
  H->>I: fetch API_BASE_URL 加 url
  I->>I: isApiRequest 判定同源或代理前缀
  I->>S: hasToken 后 getToken
  S-->>I: accessToken 或抛出刷新失败
  I->>I: 无 Authorization 时补 Bearer
  I->>G: 原始 fetch 透传
  G-->>I: 200 或 401
  I-->>H: Response
  H-->>P: data 或格式化后的 error
```

时序图里没有出现「页面设置令牌」这一步，这正是设计目标。代价是隐式全局副作用：任何绕过 `window.fetch` 的调用（例如 `XMLHttpRequest`、`sendBeacon`）都不被覆盖，且拦截器无法从类型上约束调用方。

`AuthContext` 负责另一半：`TokenManager` 每次换票都把带绝对过期时间的会话写进 `openfoundry_token`（`apps/app-console/src/context/AuthContext.tsx:103-113`），密码登录走 `POST /multipass/api/auth/login`（`:134-138`），生产登录用 `createAuthorizationUrl`/`exchangeCodeForToken` 走 PKCE，`redirectUri` 固定为 `${origin}/login`，client id 读 `VITE_OAUTH_CLIENT_ID`、默认 `openfoundry-console`（`:66-75`）。

### 9.8 会话生命周期、错误呈现与可访问性

会话持久化有三条规则，都在防同一类故障——「界面显示已登录，请求却全部 401」：

- `restoreSession()` 要求 token 与 user 两把键同时存在，只有一半时直接清空（`apps/app-console/src/lib/session.ts:88-93`）。注释解释：token 单独存在就足以让拦截器继续发一个死令牌。
- `isRestorable()` 只接受两种会话：未到期，或已到期但带 refresh token；否则清空，避免把操作员留在「看似登录、每请求 401、又回不到登录页」的死角（`apps/app-console/src/lib/session.ts:58-70`）。
- `takeOAuthCallback()` 同步读取并立刻用 `history.replaceState` 抹掉 URL 上的 `code`。注释点明原因：React StrictMode 会把回调 effect 跑两遍，若 URL 清理晚于异步换票，两次会换同一个一次性 code，第二次必然失败（`apps/app-console/src/lib/session.ts:110-124`）。

错误呈现分三层：`useApi` 把网络错误、超时、401/403/404/5xx 翻译成固定文案（`apps/app-console/src/hooks/useApi.ts:30-62`）；写操作失败时页面调用 `apiErrorMessage(res)`，把服务的 `errorName` + `parameters` 拼成「Group member not found: xxx」这类句子（`apps/app-console/src/lib/apiError.ts:83-100`）；未捕获的渲染异常由 `ui-components` 的 `ErrorBoundary` 接住，离线状态由 `NetworkStatusBanner` 提示（`apps/app-console/src/App.tsx:11`、`:36`、`:159`）。控制台没有任何前端遥测或错误上报依赖，失败只存在于当前浏览器。

可访问性与主题是两条轻量实现：外壳提供可聚焦的「Skip to main content」链接，聚焦时用内联样式移到左上角（`apps/app-console/src/App.tsx:44-80`）；深色模式只是给根节点加 `bp5-dark` 类，由 `app.css` 的 `.bp5-dark` 变量覆盖（`apps/app-console/src/styles/app.css:19-24`）；侧边栏支持上下方向键在菜单项间移动并同步焦点，每项带 `aria-current="page"`（`apps/app-console/src/components/Sidebar.tsx:70-91`）。GUIDE 记录的「监视器卡片上两个图标按钮没有可读名称」正属于这类实现的遗漏（GUIDE 已知限制第 10 条）。

### 9.9 样式体系与 `@layer` 的坑

`app.css` 第一行声明 `@layer blueprint, app`（`apps/app-console/src/styles/app.css:4`），随后把几乎全部规则包进 `@layer app { ... }`（`:29-167`）。意图很清楚：让应用样式稳定压过 Blueprint。

这个意图没有实现，因为 `main.tsx` 把 Blueprint 的四份 CSS 当普通样式表导入（`apps/app-console/src/main.tsx:5-8`），它们不带 layer。CSS 级联里**未分层的规则优先于任何分层规则**，与选择器特异性无关。于是 `@layer app` 里的规则永远赢不了 Blueprint。作者最终在文件末尾把导航栏覆盖规则放到 `@layer app` 之外，并用一段 17 行注释记录了原因（`apps/app-console/src/styles/app.css:169-190`）：

```css
/* 这段规则刻意放在 @layer app 之外：Blueprint 的 CSS 未分层导入，
   未分层规则胜过任何 layer，与特异性无关。见 main.tsx。 */
.app-shell .bp5-navbar {
  display: flow-root;
  height: auto;
  min-height: var(--navbar-height);
}
```

要覆盖的根因是 Blueprint 用浮动布局的两组导航项在窄视口换行后溢出、盖住页头的操作按钮，点击会误触 Logout（`apps/app-console/src/styles/app.css:172-180`）。这也解释了为什么 `packages/ui/tokens/src/tokens.css` 的 `@layer tokens` 方案没有落地——全仓没有任何入口 import 它（见〈packages/ui：三个包的现状〉）。

### 9.10 Blueprint v5 的组件事实

控制台锁定 `@blueprintjs/core`、`icons`、`select`、`table` 均为 `^5`（`apps/app-console/package.json:14-17`）。v5 的 API 变化有两处会直接编译失败，代码里都能找到正确写法：

- `HTMLTable` 用 `compact` 而不是 v4 的 `condensed`：`apps/app-console/src/components/DataTable.tsx:64`、`apps/app-console/src/pages/DataHealth.tsx:788` 等。
- `Tag` 没有 small 尺寸。v5 把它类型化为 `size?: NonSmallSize = Exclude<Size, "small">`；全仓 `Tag` 用法里没有任何 `small`（抽查 `DataHealth`、`SanocareDashboard`、`ObjectExplorer`、`GroupManagement`），而 `Button` 仍接受 `small`（`apps/app-workshop/src/components/Canvas.tsx:161`）。

### 9.11 app-workshop：低代码页面构建器

`app-workshop` 是一个纯浏览器端的页面构建器，没有后端。数据模型只有两层：

```ts
// apps/app-workshop/src/widgets/widget-registry.ts:39-44
export interface WidgetInstance {
  id: string;                        // w_<ts>_<rand>，仅页内唯一
  type: WidgetType;                  // TABLE|CHART|METRIC|TEXT|FILTER|BUTTON|IMAGE
  config: Record<string, unknown>;   // 由 configSchema 驱动表单编辑
  position: WidgetPosition;          // 12 列网格的 x/y/w/h
}
```

页面对象 `WorkshopPage` 是 `rid + name + description + widgets[] + createdAt/updatedAt`，RID 用 `ri.workshop.page.<毫秒>-<随机>` 生成（`apps/app-workshop/src/store/page-store.ts:4-18`）。存储层是一个单例 `PageStore`：内存 `Map` 为真源，每次变更 `notify()` 通知订阅者并同步写 `localStorage["openfoundry_workshop_pages"]`（`:24-31`、`:113-132`），React 侧用 `useSyncExternalStore` 订阅（`:140-160`）。因此页面不跨设备、不跨浏览器，也不进服务端。

七种 widget 各自带 `defaultConfig` 与 `configSchema`，`ConfigPanel` 完全由 schema 生成表单（`apps/app-workshop/src/components/ConfigPanel.tsx:105-115`、`:140-210`）；`WidgetRenderer` 是一个 `switch(type)` 分发到七个组件（`apps/app-workshop/src/widgets/WidgetRenderer.tsx:16-34`）。画布是 CSS Grid 12 列，拖拽与缩放用手写 `mousemove` 计算列宽 `gridRect.width / 12`、行高固定 68px（`apps/app-workshop/src/components/Canvas.tsx:47-79`、`:99-109`），没有引入任何拖拽库。

它与 `packages/ui` 的关系只有一层：`.storybook` 的 9 个 story 直接消费 `ui-components`（如 `apps/app-workshop/src/stories/ObjectTable.stories.tsx`）与 `ui-icons`，而应用本体并没有 import `ui-components`。`ui-tokens` 虽然在 `package.json` 里声明为依赖（`apps/app-workshop/package.json:16`），但全仓源码没有任何地方 import 它。

### 9.12 app-docs：文档站

`app-docs` 是最简单的一个：6 条路由，`/` 重定向到 `/getting-started`（`apps/app-docs/src/routes.tsx:9-20`），左侧导航按 Introduction / Guides / Reference 三组分栏（`apps/app-docs/src/components/DocNav.tsx:8-30`），正文是直接写死的 TSX 页面，配一个 `CodeBlock` 组件。内容覆盖入门、架构、API 参考、本体指南、SDK 指南与部署指南，是仓库内唯一面向使用者的英文文档界面；产品级用户指南仍是根目录的印尼语 `GUIDE.md`。页头的 Console 链接硬编码 `http://localhost:3000`（`apps/app-docs/src/components/DocLayout.tsx:25`）。

### 9.13 packages/ui：三个包的现状

| 包 | 导出 | 实际消费者 |
|---|---|---|
| `@openfoundry/ui-components` | 13 个组件（ObjectTable、FilterBar、ActionForm、PropertyRenderer、RidLink、StatusTag、PaginationBar、ConfirmDialog、LoadingState/ErrorState/EmptyState、ErrorBoundary、NetworkStatusBanner）+ `useRetry` + 4 个 toast 函数（`packages/ui/components/src/index.ts:1-45`） | `app-console` 的 `ErrorBoundary`/`NetworkStatusBanner`（`apps/app-console/src/App.tsx:11`）；`app-workshop` 的 story |
| `@openfoundry/ui-icons` | `iconMap`（23 个语义名到 Blueprint 图标名的映射）+ `FoundryIcon`（`packages/ui/icons/src/icon-map.ts:1-25`） | 仅 `app-workshop` 的 story |
| `@openfoundry/ui-tokens` | 6 个 TS 模块（colors/spacing/typography/shadows/breakpoints）+ `tokens.css`（`@layer tokens`，含明暗两套变量） | 无源码引用；只有自己的测试读取 `tokens.css` |

`ui-components` 的组件在实现上仍是 Blueprint 的薄封装（`StatusTag` 只是把状态映射到 `Intent` 再渲染 `<Tag minimal round>`，`packages/ui/components/src/StatusTag.tsx:30-38`），`ObjectTable` 的主键字段默认值是 `"id"`（`packages/ui/components/src/ObjectTable.tsx:41`）——这正好踩中下一节的 `rid`/`id` 错位。

### 9.14 前端到服务的契约错位

**其一：服务端叫 `rid`，控制台读 Foundry 的 `id`。** `svc-admin` 对分组同时发两个字段并留了注释（`services/svc-admin/src/routes/v2/groups.ts:11-14`），但用户序列化只发 `rid`（`services/svc-admin/src/routes/v2/users.ts:10-20`），而 `UserManagement` 的保存逻辑读 `editUser.id`（`apps/app-console/src/pages/UserManagement.tsx:87`），`GroupManagement` 的行键也用 `r.id`（`:216`）。结果是对用户行点保存会请求 `/api/v2/admin/users/undefined`；行键为 `undefined` 的表格不会报错，只会静默地按同一键渲染。

**其二：无 body 的请求带着 `Content-Type: application/json`。** Fastify 对声明了 JSON content-type 却没有 body 的 POST/PUT 抛 `FST_ERR_CTP_EMPTY_JSON_BODY`，网关把它作为 4xx 透传成 400 `InvalidRequest`。`PipelineExplorer` 的「执行」按钮就是这种写法（`apps/app-console/src/pages/PipelineExplorer.tsx:233-237`）。反之，`NotificationsPage` 的无 body POST 没有带这个头（`apps/app-console/src/pages/NotificationsPage.tsx:113-114`），因此不受影响——这解释了为什么同一类缺陷只出现在部分页面。

**其三：构建期 API 地址有三个名字，代码只读其中一个。** 控制台读 `VITE_API_URL`，缺省 `http://localhost:8080`（`apps/app-console/src/config.ts:7-9`）；`.env.example` 与 `deploy/docker/Dockerfile.console` 设置的是 `VITE_API_BASE_URL`（`.env.example:91`、`deploy/docker/Dockerfile.console:46-47`）；Helm 的 console Deployment 设的是 `NEXT_PUBLIC_API_URL`（`deploy/helm/openfoundry/templates/console-deployment.yaml:31-32`）。三者互不相认，任何一条交付路径上构建出来的控制台都会回落到 `http://localhost:8080`。

> 文档与实现不一致：GUIDE 的「Batasan yang Diketahui」第 16 条已记录 `VITE_API_BASE_URL` 不被读取；实现里还多出 Helm 的 `NEXT_PUBLIC_API_URL` 这一变体，GUIDE 未提及。

### 9.15 设计决策与理由

1. **用拦截器而不是认证客户端。** 覆盖 79 个既有调用点只需一次安装，新代码零成本继承；取舍是隐式全局副作用、无法从类型上约束、测试必须 mock `globalThis.fetch`（`apps/app-console/src/lib/authFetch.test.ts` 里 16 处 fetch 就是为此存在）。
2. **不引入全局状态库。** 页面各自持有 `useState`，只有认证进 Context；取舍是页面间零共享缓存、切页即重取、重复请求会撞上网关限流，但依赖树极小、没有缓存失效语义要维护。
3. **路由全量懒加载，只把首页做成同步导入。** 25 个 chunk 让首屏只带 Dashboard 与外壳；取舍是每个页面首次进入有一次 Spinner 闪烁（`apps/app-console/src/routes.tsx:40-44`），且构建产物更碎。
4. **Workshop 用 `localStorage` 当后端。** 无需服务端 schema 与权限模型就能演示低代码闭环；取舍是页面无法共享、无法多人协作、清缓存即丢数据，且 `app-workshop` 与 `app-console` 里的 `Workshop`/`WorkshopEditor` 页面构成两套互不相通的实现。
5. **组件库只做薄封装。** `ui-components` 复用 Blueprint 的交互与可访问性，避免自造轮子；取舍是 Blueprint 的升级会同时穿透组件库与两个应用（v5 的 `Tag`/`HTMLTable` 变更即为例证），而 `ui-tokens` 这套自研 token 体系因为没有入口 import 而实际失效。

### 9.16 已知限制与坑

1. **空 body + JSON content-type**：见上节，`PipelineExplorer` 的 execute 会 400；修法是去掉该请求头或补一个 body。AGENTS 记录「Several pages still pair a bodyless fetch with that header」（`AGENTS.md:160-161`），本次核查确认仅 execute 一处是「无 body 的 POST/PUT 且带 JSON content-type」。
2. **`rid`/`id` 错位**：用户页保存路径为 `undefined`，分组页因服务端双发而幸免；AGENTS 已列为未修（`AGENTS.md:163-164`）。
3. **`@layer` 覆盖失效**：Blueprint 未分层导入，应用层规则永远输；任何需要压过 Blueprint 的规则都必须写在 layer 之外（`apps/app-console/src/styles/app.css:182-190`、`AGENTS.md:170-171`）。
4. **`VITE_API_*` 三个名字**：见上节；影响是所有容器化构建的控制台都会指向 `localhost:8080`（`AGENTS.md:195`、GUIDE 第 16 条）。
5. **`AIPChat` 自己造了一套认证头**：它读 `localStorage.getItem("token")`（`apps/app-console/src/pages/AIPChat.tsx:55-58`），而会话实际写在 `openfoundry_token` / `openfoundry_user`（`apps/app-console/src/lib/authFetch.ts:16-19`），因此这段代码永远取不到令牌，只是被全局拦截器兜住；一旦有人关掉拦截器，AIP Chat 会静默变成匿名请求。
6. **`app-workshop` 不在类型检查与测试覆盖内**：`test` 是 `echo 'no tests'`（`apps/app-workshop/package.json:12`），无 `typecheck` 脚本，构建只跑 `vite build`；AGENTS 记录的「唯一没有测试文件的包」即指它（`AGENTS.md:49`）。
7. **`PestControlDashboard` 是孤儿页面**：无路由却仍在 `tsc` 的类型检查范围内，任何对它的修改都不会被用户看到。
8. **`ui-tokens` 是死代码**：`tokens.css` 与 6 个 TS 模块没有任何入口引用，主题实际由 `app.css` 里的 CSS 变量承担（`apps/app-console/src/styles/app.css:9-24`）。

### 9.17 关键代码位置

- `apps/app-console/src/main.tsx:16` — `installAuthFetch()` 在首次渲染前安装
- `apps/app-console/src/lib/authFetch.ts:101-116` — 同源/前缀判定与凭据端点豁免
- `apps/app-console/src/routes.tsx:63-98` — 27 条路由的完整清单
- `apps/app-console/src/hooks/useApi.ts:96-175` — 超时、重试、`refetch` 语义
- `apps/app-console/src/styles/app.css:182-190` — 唯一绕开 `@layer` 的覆盖及原因注释
- `apps/app-console/src/config.ts:7-9` — `VITE_API_URL` 的唯一读取点
- `apps/app-console/src/pages/PipelineExplorer.tsx:233-237` — 空 body 带 JSON 头的实例
- `apps/app-workshop/src/store/page-store.ts:24-160` — localStorage 单例与 `useSyncExternalStore`
- `apps/app-workshop/src/widgets/widget-registry.ts:39-44` — widget 实例的真实类型
- `packages/ui/components/src/index.ts:1-45` — 组件库导出面

---

## 第 10 章 工程化与交付

本章导读：这个仓库把 49 个 workspace 包、13 个微服务与 3 个前端应用塞进一个 pnpm + Turborepo 单体仓库，用一套根级 TypeScript、ESLint、Vitest 配置约束全部代码，再用 4 条 GitHub Actions 工作流和一套按服务拆分的镜像，支撑四种面向使用者的交付形态（本地脚本、单容器、Docker Compose、Helm）。本章按「一条命令如何从源码走到可运行系统」的顺序展开：workspace 与任务图、语言与检查工具、CI、容器与编排。构建数字均来自本仓库实测，文档与实现冲突处单独标注。

### 10.1 职责与边界

工程化层负责：依赖拓扑与增量构建、类型/规范/测试的统一门禁、镜像与编排产物的定义、本地与生产两套启动路径。
它不负责业务语义，也不保证交付产物彼此等价——恰恰相反，本章后半段会说明 Compose 与 Helm 的探测路径、镜像来源和环境变量已经与代码脱节。

### 10.2 仓库与 workspace

`pnpm-workspace.yaml` 只有 5 条 glob（`pnpm-workspace.yaml:1-6`）：`packages/*`、`packages/sdk/*`、`packages/ui/*`、`services/*`、`apps/*`。前三条说明共享库允许两层嵌套，`sdk` 与 `ui` 是人为划出的子命名空间。

| 分组 | 包数 | 命名 | 示例 |
|---|---|---|---|
| `packages/*` | 26 | `@openfoundry/<目录名>` | `errors`、`permissions`、`db`、`search` |
| `packages/sdk/*` | 4 | `@openfoundry/sdk*` | `sdk`、`sdk-cli`、`sdk-oauth`、`sdk-react` |
| `packages/ui/*` | 3 | `@openfoundry/ui-*` | `ui-components`、`ui-icons`、`ui-tokens` |
| `services/*` | 13 | `@openfoundry/svc-*` | `svc-gateway`、`svc-objects` |
| `apps/*` | 3 | `@openfoundry/app-*` | `app-console`、`app-workshop`、`app-docs` |
| **合计** | **49** | | |

跨包依赖一律用 `workspace:*`（例如 `apps/app-console/package.json:20-22`），保证本地永远链接源码而非 registry 版本。包管理器与运行时被钉死：`packageManager: pnpm@10.27.0`、`engines.node >= 20`（`package.json:37-40`），镜像里也据此安装 pnpm（`deploy/docker/Dockerfile.service:16`）。49 个包里 42 个是 `private`，只有 7 个（`auth-tokens`、`rid`、`ontology-schema`、`redis`、`object-set`、`errors`、`client`）声明了可发布版本。

> 文档与实现不一致：README 的项目结构写「Shared libraries (28 packages)」「Microservices (13 defined, 10 active)」（`README.md:303-305`），实际是 33 个共享库包（26+4+3）、13 个服务包、3 个应用包，共 49 个 workspace 包；「10 active」只对 `start.sh` 成立，对 `docker-compose.yml` 与 Helm 都不成立。

### 10.3 构建管线

```mermaid
flowchart TD
  A[pnpm install --frozen-lockfile] --> B[turbo run build]
  B --> C[按 ^build 先构建被依赖的共享包]
  C --> D[各包 tsc 输出 dist]
  D --> E[services 与 apps 再构建]
  E --> E1[app-console tsc 加 vite build]
  E --> E2[app-workshop 仅 vite build]
  E --> E3[app-docs tsc 加 vite build]
  B --> F[turbo run test 依赖 build]
  B --> G[turbo run typecheck 依赖 ^build]
  H[eslint . 不经 turbo] --> I[0 error 即通过]
```

关键在 `C`：`build` 声明 `dependsOn: ["^build"]`（`turbo.json:6`），Turborepo 按包依赖拓扑排序，先产出 `packages/*` 的 `dist`，服务与应用才能通过 `dist/index.js` 导入它们。`test` 用的是非脱字符的 `dependsOn: ["build"]`（`turbo.json:15`），意思是「先构建本包自身，再跑测试」——这也是 CI 里先 `pnpm run build` 再起服务的原因（`.github/workflows/test.yml:43-47`）。`typecheck` 用 `^build`，只要依赖包的声明文件在即可。

### 10.4 Turborepo 任务图

| 任务 | dependsOn | inputs / outputs | 真实执行任务数 |
|---|---|---|---|
| `build` | `^build` | inputs：`src/**`、`tsconfig.json`、`package.json`；outputs：`dist/**` | 49 |
| `test` | `build` | inputs：`src/**`、`tests/**`；env：`MODELS_TEST_DATABASE_URL` | 97 = 48 test + 49 build |
| `typecheck` | `^build` | 继承包内 `tsc --noEmit` | 45 = 24 typecheck + 21 build |
| `generate` | `^build` | outputs：`src/generated/**` | 仅 `api-types` |
| `dev` | 无 | `cache: false`、`persistent: true` | 全部声明 `dev` 的包 |
| `clean` | 无 | `cache: false` | 声明了 `clean` 的包 |

`turbo.json` 还声明了 `globalDependencies: ["tsconfig.base.json"]`（`turbo.json:3`）：改基础 tsconfig 会让所有缓存失效，这是正确的保守做法。`generate` 只服务 `api-types` 的代码生成，根脚本用 `--filter` 收窄（`package.json:19`）。

表里的任务数与 AGENTS 记录的「build 49/49、typecheck 45/45、test 97/97」完全对得上，但需要解释一个容易误读的现象：`turbo run typecheck --dry=json` 会列出 **70** 个节点，`test` 列出 **98** 个——多出来的是没有对应脚本的包，Turborepo 用 `<NONEXISTENT>` 占位。真实执行的任务是 45 与 97，分别等于「有脚本的包数 + 其依赖闭包内的 build 任务」。因此 AGENTS 的数字是可信的，只是它不是「包数」。

失败处理是这套图的硬伤：普通 `turbo run build` 在第一个失败处停止，要看全部失败必须 `npx turbo run build --continue`（`AGENTS.md:10`）。CI 用的是不带 `--continue` 的 `pnpm run build`，所以一次红只会暴露一个包的问题。

### 10.5 TypeScript 配置

`tsconfig.base.json` 是全仓唯一的共享编译基线，关键选项是：

```jsonc
// tsconfig.base.json:3-22（节选）
"target": "ES2022", "module": "ESNext", "moduleResolution": "bundler",
"lib": ["ES2022"],                  // 没有 DOM：Node 与浏览器代码共用基线
"strict": true, "isolatedModules": true,
"declaration": true, "declarationMap": true, "sourceMap": true,
"outDir": "dist", "rootDir": "src",
"noUnusedLocals": true, "noUnusedParameters": true
```

两个后果值得记住。第一，`lib` 不含 DOM，所以 `CryptoKey`、`window`、`document` 这类全局在服务端包中不存在；`CryptoKey` 必须从 `jose` 导入（`AGENTS.md:34`）。第二，`noUnusedLocals`/`noUnusedParameters` 把未使用的 import 与局部变量变成硬错误，只有**参数**可以用 `_` 前缀豁免，局部变量不行（`AGENTS.md:29-30`）。

每个包用 `extends` 继承，并按需覆盖：服务端包只加 `outDir`/`rootDir`（`services/svc-gateway/tsconfig.json`）；前端包补 `jsx: "react-jsx"`、`lib: ["ES2022","DOM","DOM.Iterable"]`、`noEmit: true`（`apps/app-console/tsconfig.json:4-12`）；`ui-components` 补 JSX 与 DOM 但保留 `declaration: true`，因为它要产出 `.d.ts`（`packages/ui/components/tsconfig.json`）。

任何使用 Node 内置模块或全局的包必须自己声明 `@types/node`，没有 hoist 兜底；42 个包（26 个 `packages/*`、13 个服务、3 个应用）里只有 16 个声明了它，漏声明的表现是 `Cannot find module 'node:fs'` 或 `Cannot find name 'Buffer'`（`AGENTS.md:36-38`）。因为 `build` 是其他任务的依赖，这一处缺口会同时让 build、typecheck、lint、test 四条流水线变红。

前端包则必须自己覆盖 `lib`：`app-console`、`app-workshop`、`app-docs`、`ui-components` 都追加了 `DOM` 与 `DOM.Iterable`（`apps/app-console/tsconfig.json:7`、`packages/ui/components/tsconfig.json:5`）。这条覆盖是逐包显式的，没有共享的「browser 基线」——新增一个前端包时忘记覆盖，`document` 就会是未定义标识符。

### 10.6 版本与依赖一致性

单仓最容易失守的是「同一个东西装了两份不同大版本」。当前实测到的分叉：

| 依赖 | 根/多数 | 例外 | 影响 |
|---|---|---|---|
| `vitest` | `^3.2.0`（根 devDeps） | 三个 `ui/*` 包声明 `^1.6.0` | 这些包跑的是各自解析到的 v1 |
| `vite` | `^6.0.0`（console、docs） | `app-workshop` 用 `^5.4.0` | 两个 Vite 大版本共存 |
| `react-router-dom` | `^7.0.0`（console、docs） | `app-workshop` 用 `^6.20.0` | 两个 API 面 |
| `@blueprintjs/core` | `^5.0.0`（console、ui） | `app-workshop` 用 `^5.13.0` | 同一主版本，风险低 |
| `typescript` | `~5.5.4`（28 个包） | `^5.0.0`/`^5.3.0`/`^5.4.0`/`^5.5.0` 共 13 个包 | pnpm 去重后实际单版本 |
| Node 运行时 | CI 与 `engines` 为 20 | `Dockerfile.service`/`Dockerfile.console` 用 `node:22-alpine`，根 `Dockerfile` 用 `node:20-alpine` | 本地、CI、镜像三种 Node |

这不是一份错误清单，而是「没有集中版本管理」的直接后果：根 `package.json` 没有 `pnpm.overrides`，也没有 pnpm 10 的 `catalog:` 协议，版本一致性完全靠评审。

### 10.7 ESLint 与格式化

`pnpm run lint` 直接是 `eslint .`（`package.json:11`），不经 Turborepo，也没有任何包级 `eslint` 脚本或配置。这是刻意的：一份根 `eslint.config.mjs` 意味着没有包能悄悄退出检查——AGENTS 记录了前任任务「21/21 全绿却检查了 0 个文件」的教训（`AGENTS.md:12-15`）。

扁平配置的结构（`eslint.config.mjs:4-47`）：

- 忽略 `**/dist/`、`**/node_modules/`、`**/*.js|mjs|cjs`，所以被检查的只有 TS 源码。
- `typescript-eslint` 的 recommended 规则集，外加四条自定义：`no-unused-vars`（warn，参数允许 `_` 前缀）、`no-explicit-any`（warn）、`consistent-type-imports`（error，强制 `import type`）、`no-console`（warn，仅允许 `warn`/`error`）。
- `react-hooks` 只启用 `rules-of-hooks`（error）与 `exhaustive-deps`（warn）两条，且只作用于 `apps/**`。注释说明了两点原因：React 应用里有 `eslint-disable-next-line react-hooks/exhaustive-deps` 注释，规则本身必须存在，否则 lint 会把抑制注释报成错误；插件的 `recommended` 预设还会带上 React Compiler 的四条规则，会给控制台新增 46 个 error，采纳与否是独立决策（`eslint.config.mjs:18-30`、`AGENTS.md:22-25`）。
- `no-console` 在 `scripts/`、`services/`、`tests/` 三个目录整体关闭，另外单独豁免三个「本职工作就是打印」的库模块：`packages/logging/src/logger.ts`、`packages/redis/src/client.ts`、`packages/sdk/sdk-cli/src/output.ts`（`eslint.config.mjs:32-46`）。做法是收窄作用域而不是逐行 disable。

本次在快照上实测：`npx eslint . --format json` 检查 **644** 个文件，**0 error、225 warning**，分布为 181 个 `no-explicit-any`、33 个 `no-unused-vars`、11 个 `react-hooks/exhaustive-deps`，与 AGENTS 的告警构成完全一致。只有 error 会让 lint 失败，因此警告数只可见、不受控。

> 文档与实现不一致：AGENTS 写「632 个 .ts/.tsx/.mts 文件」，实测 `find` 与 ESLint 都是 644 个。差异应是 AGENTS 写作时的快照计数，不是配置差异。

Prettier 有配置（`.prettierrc:1-10`，`printWidth: 100`、`trailingComma: all`、双引号）但完全不设卡：`pnpm run format` 只写文件，CI 里没有 `--check`，多数文件也不符合 Prettier 格式。因此不要因为 `prettier --check` 报错就整文件重排（`AGENTS.md:26-27`）。

### 10.8 Vitest

单元测试由 Turborepo 的 `test` 任务驱动，`pnpm run test` 是 `turbo run test`（`package.json:10`）。根 `vitest.config.ts` 做两件事：为 `packages/*` 的每个目录生成 `@openfoundry/<dir>` 到 `src/index.ts` 的别名，让测试直接跑源码；声明 `include: ["**/tests/**/*.test.ts", "**/src/**/*.test.ts"]`（`vitest.config.ts:7-28`）。别名循环只遍历 `packages/` 一层（`:9-17`），`packages/sdk/*` 与 `packages/ui/*` 不在表内，这两个子目录里的包仍经 pnpm 链接到 `dist` 解析。

这里有一个只在单仓里才会出现的坑：这些 glob 是相对于**本次运行的 root** 解析的。根级运行 root 是仓库根，所以 `packages/**` 也能匹配；但某个包执行 `vitest run` 时继承同一份配置，root 变成包目录，任何根锚定的 glob 都匹配不到东西，每个包都会报 "No test files found"——而那并不意味着没有测试（`AGENTS.md:43-48`）。48 个声明了 `test` 的包里，47 个确实有测试文件，只有 `app-workshop` 没有；用 `--passWithNoTests` 把它压绿，等于让整个单仓的测试静默失效。

`apps/app-console` 是这条规则的例外：它没有 `vitest.config.ts`，`vitest run` 会解析到 `apps/app-console/vite.config.ts`，那里没有 `test` 块，于是 Vitest 退回自己的默认 glob，而不是根配置的 `**/tests/**` + `**/src/**`（`AGENTS.md:52-54`）。它的 7 个测试文件都在 `src/lib/`，恰好被默认 glob 覆盖，所以一直是绿的。

集成测试是另一套配置：`pnpm run test:integration` 用 `-c tests/vitest.config.ts`（`package.json:23`），该配置把 `root` 设为 `./tests`、`include` 设为 `integration/**/*.test.ts`（`tests/vitest.config.ts:24-27`）。CI 的集成任务却直接跑 `npx vitest run tests/integration/`（`.github/workflows/test.yml:98`），用的是根配置加路径过滤——两条路径并不等价，只是当前目录布局下结果相同。

### 10.9 CI：四条工作流

| 工作流 | 触发 | 作业 | 说明 |
|---|---|---|---|
| `ci.yml` | PR 与 push 到 `master` | lint、typecheck、test、build | test 带 `postgres:16` service 与 `MODELS_TEST_DATABASE_URL` |
| `test.yml` | PR 与 push 到 `master` | 集成测试 | 构建后启动 9 个服务，seed 后跑 `tests/integration` |
| `docker.yml` | push `master`、`v*` tag、手动 | 12 个镜像矩阵 | 推 ghcr.io，仅打 `github.sha` 标签 |
| `release.yml` | push 到 `master` | changesets 发布 | 因无 `NPM_TOKEN` 必然失败 |

```mermaid
flowchart LR
  A[push 或 PR 到 master] --> B[ci.yml]
  A --> C[test.yml]
  A --> D[docker.yml 仅 push 与 tag]
  A --> E[release.yml 仅 push]
  B --> B1[lint 全仓 644 文件]
  B --> B2[typecheck 45 个任务]
  B --> B3[test 97 个任务 带 Postgres]
  B --> B4[build 49 个包]
  C --> C1[build 后启动 9 个服务]
  C1 --> C2[seed-pest-control]
  C2 --> C3[vitest tests/integration]
  D --> D1[12 个 svc 镜像推 ghcr]
  E --> E1[changeset publish 失败]
```

`ci.yml` 的四个作业彼此独立、各自 `pnpm install`，没有共享缓存；`build` 作业构建全部 49 个包，`typecheck` 作业因为 `^build` 还会顺带执行依赖闭包内的 21 个 build 任务，这是用时间换简单。集成任务额外用 `OPENFOUNDRY_CLIENT_ID/SECRET=admin/admin123` 作为非浏览器调用方的凭据，并在 seed 前用 `test -f` 断言脚本存在——注释写明「静默跳过的 seed 会让测试在空库上通过」（`.github/workflows/test.yml:19-28`、`:90-95`）。

`docker.yml` 有两条必须照抄的规则。其一，镜像仓库名必须从小写的 `GITHUB_REPOSITORY` 推导，不能直接用 `github.repository`，因为 owner 登录名是 `Przyval`，ghcr.io 拒绝任何大写字母（`.github/workflows/docker.yml:37-42`、`AGENTS.md:188`）。其二，`Dockerfile.service` 用包名而不是目录名选包：`turbo prune --scope="@openfoundry/${SERVICE}"`（`deploy/docker/Dockerfile.service:24`），矩阵里传的是 `svc-gateway` 这样的服务名，两者恰好同名，但机制是包名匹配，其余 `${SERVICE}` 出现处是文件系统路径。

`release.yml` 是一段死代码：仓库没有任何 Actions secrets，`NPM_TOKEN` 为空，`changeset publish` 每次都在 `ENEEDAUTH` 上失败（`AGENTS.md:190`、`.github/workflows/release.yml:27-32`）。即使有 token，`.changeset/config.json` 的 `baseBranch` 写的是 `main`，而仓库默认分支是 `master`；49 个包里也只有 7 个可发布。这个工作流的现实作用是每次 push 后贡献一个红叉。

### 10.10 容器化与交付形态

| 形态 | 入口 | 构建物 | 暴露面 | 适用场景 |
|---|---|---|---|---|
| 本地脚本 | `bash start.sh` | 无（tsx 直跑源码 + Vite dev server） | 8080–8092、3000 | 开发与演示 |
| 单容器 | 根 `Dockerfile` + `docker-compose.yml` | 1 个镜像，内含 9 个服务 + Vite | 8080、3000 | 一键演示 |
| 服务镜像 | `deploy/docker/Dockerfile.service` × 12 + `Dockerfile.console` | 13 个镜像（CI 只发布 12 个服务镜像） | 每容器一个端口 | Compose / Helm / 发布 |
| Docker Compose | `deploy/docker-compose.yml` | 上述镜像 + postgres/redis/minio | 仅 gateway 与 console | 单机生产 |
| Helm | `deploy/helm/openfoundry` | 同上，镜像从 ghcr 拉 | Ingress 两条 host | Kubernetes |

`Dockerfile.service` 是标准的四阶段：`pruner` 用 `turbo prune --scope=@openfoundry/${SERVICE} --docker` 裁出子图（`:14-24`），`installer` 先用 `out/json` 只装依赖以复用层缓存、再复制全量源码并 `turbo run build --filter` 构建（`:29-45`），`prod-deps` 装生产依赖（`:50-59`），`runtime` 以 uid 1001 的非 root 用户只拷 `dist` 与 `packages/`（`:64-93`）。根 `Dockerfile` 则是两阶段「整仓复制 + `pnpm install` + entrypoint」，`docker-entrypoint.sh` 用 pnpm 起 9 个服务、等 15 秒、按端口表检查、按需 seed、最后以 `--host 0.0.0.0` 起 Vite（`docker-entrypoint.sh:40-45`、`:99-104`）。

`Dockerfile.console` 是交付链上唯一把 SPA 变成静态文件的组件：`turbo prune --scope=@openfoundry/app-console` 裁出子图（`deploy/docker/Dockerfile.console:24-25`），`pnpm install` 后 `turbo run build --filter`（`:49`），把 `apps/app-console/dist` 拷进 `nginx:1.27-alpine`，以 uid 1001 非 root 运行（`:54-80`）。nginx 配置承担四项职责：SPA 回退 `try_files $uri $uri/ /index.html`（`deploy/docker/nginx.conf:126-136`）、把 `/api/` 与 `/multipass/` 反代到 `svc-gateway:8080`（`:84-115`）、对 `/health` 直接回 200（`:77-81`，与 Dockerfile 的 HEALTHCHECK 对应）、以及一组安全头与 CSP（`:68-74`）。CSP 的 `connect-src` 只放行 `'self'`、`localhost:*` 与两个 `openfoundry.*` 域名（`:74`）；如果按上一章说的 `VITE_API_URL` 回落到 `http://localhost:8080`，浏览器会同时被跨源与 CSP 拦住，这是变量错配在生产态的第二重后果。

`deploy/docker-compose.yml` 把每个服务拆成独立容器，只有 `svc-gateway` 与 `app-console` 接到 `external` 网络，其余在 `internal: true` 的桥接网络里（`:604-609`）——这是网络层的租户隔离，与 `AGENTS.md:25-28` 的设计一致。`deploy/docker-compose.dev.yml` 只起基础设施（Postgres 5434、Redis 6381、Traefik、MinIO），应用仍在宿主机跑；根 `package.json` 的 `docker:up` 用的正是它（`package.json:17`）。

Helm chart 的问题最多，且都能从模板直接读出：chart 没有声明 `dependencies`，`values.yaml` 里的 `postgresql`/`redis` 子 chart 配置因此没有载体；`values.yaml` 的 `services` 映射只有 11 个服务，缺 `svc-aip`；迁移 Job 用 `openfoundry/db-migrations:${tag}` 镜像（`deploy/helm/openfoundry/templates/migration-job.yaml:24`），console 用 `openfoundry/app-console:${tag}`（`deploy/helm/openfoundry/templates/console-deployment.yaml:22`），而 `docker.yml` 只构建 12 个 `svc-*` 镜像，这两个镜像永远不会被发布；网关的 ConfigMap 只提供 `SVC_<NAME>_URL`（`deploy/helm/openfoundry/templates/configmap.yaml:12-14`），而网关读的是 `ONTOLOGY_SERVICE_URL`、`OBJECTS_SERVICE_URL` 这一组名字（`services/svc-gateway/src/config.ts:106-116`）。

探测路径则是一处跨形态的系统性错位：所有服务的健康端点在 `/status/health`、`/status/liveness`、`/status/readiness`（`services/svc-objects/src/routes/health.ts:17-27`，13 个服务同构），但 `Dockerfile.service` 的 HEALTHCHECK 打 `localhost:3000/health`（`deploy/docker/Dockerfile.service:90-91`，端口与路径都错），Compose 里除网关外的 12 个服务健康检查打 `/health`（`deploy/docker-compose.yml:254`、`:280` 等），Helm 的 liveness 打 `/health`、readiness 打 `/health/ready`（`deploy/helm/openfoundry/templates/service-deployment.yaml:47-58`）。按路由注册推断，这些探测会稳定 404，`depends_on: condition: service_healthy` 会一直等到超时（未在容器中实测）。

> 文档与实现不一致：AGENTS 称「the container healthcheck uses `/status/health`」（`AGENTS.md:119`）。实际只有 `deploy/docker-compose.yml` 里网关那一处用了它（`:216-217`）；`Dockerfile.service` 与其余 12 个服务、以及全部 Helm 探测，用的都是不存在的 `/health`。

### 10.11 环境变量

| 变量 | 读取方 | 作用 | 默认值 |
|---|---|---|---|
| `DATABASE_URL` | 所有服务、`scripts/migrate.js` | 切到 Postgres 存储；不设则用内存/JSON 文件 | 无（服务各自回落） |
| `REDIS_URL` | 服务与网关 | 缓存、限流、pub/sub | `redis://localhost:6379` |
| `AUTH_PUBLIC_KEY` | svc-gateway | 验签公钥；为空则**不校验任何请求** | 空（demo 默认） |
| `JWT_PRIVATE_KEY` / `JWT_PUBLIC_KEY` / `TOKEN_EXPIRY_SECONDS` | svc-multipass | 签发密钥对与访问令牌有效期；密钥为空则启动时生成临时对 | 空 / 空 / 3600 |
| `AUTH_ISSUER` / `AUTH_AUDIENCE` | svc-gateway | `iss`/`aud` 校验 | `openfoundry-multipass` / `openfoundry-api` |
| `OAUTH_CLIENTS` | svc-multipass | JSON 数组，声明可登录的客户端与 redirect URI | 空（仅开发种子） |
| `ENFORCE_PERMISSIONS` | 权限中间件 | 是否强制权限模型 | `false`（`deploy/` 也保持 false） |
| `VITE_API_URL` | app-console | 前端 API 基址（代码唯一读取的名字） | `http://localhost:8080` |
| `VITE_API_BASE_URL` | 无人读取 | `.env.example` 与 console 镜像设置它 | — |
| `VITE_OAUTH_CLIENT_ID` | app-console | PKCE 登录的 client id | `openfoundry-console` |
| `PORT` / `HOST` | 各服务 | 监听端口与地址 | 各服务不同（如 8082） |
| `MODELS_TEST_DATABASE_URL` | turbo `test` 任务 | 模型相关测试的 Postgres | 无 |
| `OPENFOUNDRY_TOKEN` 或 `_CLIENT_ID`/`_SECRET` | `scripts/lib/auth.*` | 非浏览器调用方的 bearer 凭据 | 空（匿名） |
| `OPENFOUNDRY_ALLOW_OPEN_SIGNUP` | svc-multipass | 开启匿名 signup（CI 专用） | 关 |
| `LOG_LEVEL` / `LOG_FORMAT` | 全部服务 | 日志级别与格式 | `info` / 文本 |

完整清单在 `.env.example:1-121`（键齐全、值为空）与 `deploy/.env.production:1-83`（生产模板，含 `CHANGEME` 占位）。两条纪律：真实凭据只进被 gitignore 的 `.env`；SSH 隧道地址只从 `TUNNEL_DB_HOST`/`TUNNEL_JUMP_HOST` 读，绝不入库（`AGENTS.md:180-182`）。

### 10.12 本地编排：start.sh 与 scripts/

本地有四个入口，用途不同：

| 入口 | 起什么 | 依赖 | 适合 |
|---|---|---|---|
| `bash start.sh` | 10 个服务 + 控制台，按行业 seed | Docker 可选、需要 `lsof` | 演示、端到端联调 |
| `pnpm dev` | turbo 起全部 17 个 `dev` 任务（13 个服务 + `redis` 包 + 3 个前端） | 无 | 只改代码、不需要种子 |
| `bash scripts/bootstrap.sh` | 装依赖 → 构建 → 起 dev Compose → 迁移 → 测试 | Docker 必需 | 新机器首次搭建 |
| `pnpm docker:up` | 只起 Postgres/Redis/Traefik/MinIO | Docker | 服务在宿主机跑、数据在容器 |

`start.sh` 是开发与演示的主入口，职责依次是：加载 `.env`、解析 `--industry`/`--reseed`、确保 Postgres（Docker 可用时）、**按端口杀进程**、起 10 个服务、按端口验证、按需 seed、起 Vite。

端口处理值得逐条对照代码：清理循环覆盖 `8080-8088`、`8092`、`3000`（`start.sh:85`），二次强杀循环却漏了 `8086`（`:95`）；服务清单是 10 个——`svc-ontology`、`objects`、`multipass`、`admin`、`actions`、`datasets`、`functions`、`aip`、`sentinel`、`gateway`（`:118`），**没有** `svc-compass`、`svc-webhooks`、`svc-media`；状态检查只覆盖 9 个端口（`:131`），`8086`、`8089`、`8090`、`8091` 都不在其中，所以 sentinel 起来了也不会被验证。这与 AGENTS 的描述一致（`AGENTS.md:128-130`），补充一点它没写全的：`8089`/`8090` 同样既不被杀也不被查。

`--industry` 有五个合法值：`pest-control`、`healthcare`、`logistics`、`manufacturing`、`all`（`start.sh:36-48`），实际执行的是 `scripts/demo/seed-<industry>.sh`。切换行业时会先删三个 store JSON 再 seed，并把行业名写进 `/tmp/openfoundry-data/last-industry`（`:157-213`）——这是唯一正确的切换方式，因为四个 seed 脚本都复用「第一个已存在的本体」，直接跑第二个只会把另一套对象类型合并进同一个本体（`AGENTS.md:132-133`）。`--reseed` 走同一段清理逻辑。

`scripts/` 里其余可交付脚本：

| 脚本 | 作用 | 关键约束 |
|---|---|---|
| `scripts/migrate.js` | 用 `@openfoundry/db` 的 `runMigrations` 应用 `db/migrations` | 缺省 DSN 是 `localhost:5432/openfoundry`，与 Compose 的 5434 不一致 |
| `scripts/migrate.sql` | 供 Postgres 容器初始化挂载的合并 schema | 自称「incremental migrations 的超集」，实际缺表（见下） |
| `scripts/demo/seed-*.sh` | 四套行业种子（各约 410–490 行） | 复用第一个本体；不读 `ORG_RID` |
| `scripts/seed-dev-data.ts` | 用 `tsx` 跑的最小样例数据 | 经网关 8080，带 `authHeaders` |
| `scripts/bootstrap.sh` | 一键：装依赖 → 构建 → 起 dev Compose → 迁移 → 测试 | 依赖 Docker 与 pnpm |
| `scripts/bootstrap-admin.sh` | 直接调 svc-multipass 的用户 CLI 创建首个 ADMIN | 需要 `DATABASE_URL`；密码交互输入，从不作参数 |
| `scripts/backup.sh` | 用 `pg_dump` 压缩到 `BACKUP_DIR`，可装每日 cron，按天数清理 | 必须设 `DATABASE_URL` |
| `scripts/test-integration.sh` | 探活网关后跑 `tests/integration` | 要求服务已在跑 |

`db/migrations/` 有 16 个文件（编号到 `016_connectivity.sql`，其中 `011` 出现两次：`011_aip_embedding_ontology.sql` 与 `011_user_platform_roles.sql`），由 `runMigrations` 按文件名顺序应用。`scripts/migrate.sql` 的注释声称是这些迁移的「self-contained superset」（`scripts/migrate.sql:1-10`），但对比建表语句，它缺 `aip_conversations`、`aip_embeddings`、`aip_generated_functions`、`aip_messages`、`permission_grants`、`pipeline_runs`、`pipelines`、`role_assignments`、`roles`、`search_index` 十张表，同时多出 `_migrations` 与 `links`。两个 schema 源确实在漂移，只是漂移方向与文档写的不完全一样。

> 文档与实现不一致：AGENTS 与 GUIDE 都说「只有 `db/migrations` 才加 `org_rid` 和 RLS」（`AGENTS.md:173`、GUIDE 第 14 条），但当前 `scripts/migrate.sql` 含 40 处 `org_rid`、21 处 RLS 语句；真正的差异是缺表，而不是缺租户列。

### 10.13 设计决策与理由

1. **一份根 ESLint 配置，取消包级 lint 任务。** 代价是无法按包定制规则、只能串行扫 644 个文件；收益是「没有包能悄悄退出检查」，直接修掉了历史上前任报告全绿却零覆盖的问题（`AGENTS.md:12-15`）。
2. **让 `test`/`typecheck` 依赖 `build`。** 服务通过 `dist/index.js` 消费共享包，不先构建就 `ERR_MODULE_NOT_FOUND`；代价是一个 `@types/node` 缺失会同时打红四条流水线（`AGENTS.md:38`），而且同一份构建在 CI 的四个作业里被重复执行。
3. **Vitest 用「相对 glob + 每包继承」而不是每包一份配置。** 48 个包零配置即可跑测试；代价是 glob 的解析基准随运行目录变化，根锚定的写法会静默匹配零文件，而 `--passWithNoTests` 会把这个信号彻底掩盖（`AGENTS.md:43-50`）。
4. **镜像按服务拆分，另保留一个单容器 demo 镜像。** 生产可独立扩缩（`values.yaml` 给 objects 2 副本、gateway 2 副本），演示只需一条 `docker compose up`；代价是两套编排产物长期无人验证，探测路径与环境变量各自漂移。
5. **健康端点统一放 `/status/*` 而不是 `/health`。** 命名空间清晰、与 `/metrics` 并列；代价是容器与 Helm 的默认约定是 `/health`，模板作者按约定写，于是全线 404（见上）。
6. **发布交给 changesets 但不配置 secrets。** 保留了版本与变更日志的工程习惯，实际从未发布；代价是每次 push 到 master 都产生一次失败记录，真正的发布路径只有 ghcr 镜像。

### 10.14 已知限制与坑

1. **Turborepo 失败即停。** 普通 `build` 只报第一个失败；要看全量必须 `npx turbo run build --continue`（`AGENTS.md:10`）。
2. **Lint 只对 error 设卡。** 225 个 warning 不受控，`no-explicit-any` 独占 181 个；Prettier 完全不校验（`AGENTS.md:16-18`、`:26-27`）。
3. **Vitest glob 的根相对语义**：包内运行时根锚定 glob 匹配零文件；`app-workshop` 没有任何测试，`app-console` 走的是默认 glob 而非根配置（`AGENTS.md:43-54`）。
4. **发布流水线永不发布。** 无 `NPM_TOKEN` → `ENEEDAUTH`；`.changeset/config.json` 的 `baseBranch` 是 `main` 而仓库是 `master`；仅 7 个包可发布（`AGENTS.md:190`）。
5. **容器与编排的健康探测全部打错路径。** 服务只有 `/status/*`，而 `Dockerfile.service` 打 `:3000/health`、Compose 的 12 个服务打 `/health`、Helm 打 `/health` 与 `/health/ready`；`depends_on: service_healthy` 会超时（按路由推断，未实测）。
6. **Helm 交付不闭环。** chart 无 dependencies，`values.yaml` 的 postgresql/redis 配置无载体；`svc-aip` 不在 services 映射里；迁移 Job 与 console 引用的镜像不由 `docker.yml` 构建；网关读 `*_SERVICE_URL`，ConfigMap 只给 `SVC_*_URL`；console 的 `containerPort` 是 3000 而镜像内 nginx 监听 80。
7. **Compose 密码不一致。** 根 `docker-compose.yml` 给 Postgres 设 `openfoundry_dev`（`:7`），应用的 `DATABASE_URL` 却用 `openfoundry`（`:37`），应用连不上库；`deploy/docker-compose.yml` 用变量统一，没有这个问题。
8. **`pnpm db:seed` 指向不存在的文件。** 根脚本是 `node scripts/seed.js`（`package.json:16`），仓库里没有 `scripts/seed.js`，只有 `scripts/seed-dev-data.ts` 与 `scripts/demo/*.sh`（GUIDE 第 17 条）。
9. **`scripts/migrate.js` 的缺省 DSN 与 Compose 端口不一致。** 缺省 `localhost:5432/openfoundry`（`scripts/migrate.js:11-13`），而 dev Compose 把 Postgres 映到宿主机 5434（`deploy/docker-compose.dev.yml:19-20`）；不设 `DATABASE_URL` 时迁移会连错库。
10. **`start.sh` 的端口管理有缺口。** 不清理 `8089`/`8090`/`8091`，二次强杀漏 `8086`，状态检查漏 4 个端口，且 compass/webhooks/media 三个服务根本不启动（`start.sh:85-132`、`AGENTS.md:128-130`）。

### 10.15 关键代码位置

- `pnpm-workspace.yaml:1-6` — 5 条 glob 定义 49 个 workspace 包
- `turbo.json:4-29` — 任务图、缓存输入与输出
- `tsconfig.base.json:3-23` — 无 DOM 的共享编译基线
- `eslint.config.mjs:4-47` — 扁平配置、规则作用域与 `no-console` 豁免
- `vitest.config.ts:19-30` — 包别名与相对 include glob
- `.github/workflows/docker.yml:19-57` — 12 镜像矩阵与小写仓库名
- `.github/workflows/release.yml:27-32` — 因缺 `NPM_TOKEN` 永不成功的发布
- `deploy/docker/Dockerfile.service:14-93` — 四阶段裁剪构建与运行时
- `start.sh:82-132` — 端口清理、10 个服务清单与 9 个端口的检查
- `scripts/migrate.js:9-18` — 迁移入口与缺省 DSN

---

## 第 11 章 横切基础库

### 11.1 本章导读

横切基础库是十三个服务共用的「不写在业务代码里、却决定所有服务行为一致」的那一层：错误信封、资源标识、分页游标、日志、追踪、健康与指标、配置装载、Redis 客户端、测试辅助。本章回答两个问题：哪些库真正被全平台依赖，哪些只是「已写好、尚未接线」；以及这些库在边界上做了哪些取舍、在哪些地方已经开始分叉。

### 11.2 职责与边界

这一层只提供**无业务语义的原语**：它不知道什么是 Ontology、Dataset 或 Action，只负责把「一个错误怎么变成 HTTP 响应」「一个 RID 怎么被解析」「一页数据怎么被切成下一页」这类问题一次性解决。业务路由的注册、鉴权、存储读写分别属于第 3、5、6 章，本章不展开。

反过来，横切库也**不**负责：不负责日志的实际落盘（服务用的是 Fastify 内置的 pino，不是 `@openfoundry/logging`）；不负责跨进程追踪（`@openfoundry/tracing` 没有消费者）；不负责限流（网关的限流器自己动态 import `ioredis`，绕过了 `@openfoundry/redis`）。

### 11.3 横切库总表

判定口径：`直接依赖者` 指在 `package.json` 中声明该 workspace 依赖的其它包数量（不含自身），`TS 引用` 指源码/测试里出现 `@openfoundry/<name>` 的文件数。

| 包 | 职责 | 直接依赖者 | 接线状态 |
|----|------|-----------|----------|
| `@openfoundry/errors` | 错误信封、错误码目录、错误工厂、`SafeArg` 脱敏、ontology scoping 拒绝 | 20 | **强制**，102 个文件引用 |
| `@openfoundry/rid` | RID 值对象、解析/构造/校验、已知服务常量 | 19 | **强制**，47 个文件引用 |
| `@openfoundry/pagination` | `pageSize`/`pageToken`/`nextPageToken` 约定、base64url 游标、异步翻页 | 7 | 强制但只覆盖 4 个服务，另有 10 处本地实现 |
| `@openfoundry/health` | Prometheus 指标插件、健康检查注册表、7 态健康模型 | 1（svc-gateway） | 仅 `metricsPlugin` 被注册 |
| `@openfoundry/config` | 分层配置装载（默认值 → `foundry.config.json` → `OPENFOUNDRY_*`） | 1（sdk-cli） | 服务**不用**，各自手写 `config.ts` |
| `@openfoundry/logging` | 结构化 JSON 日志、`SafeArg` 分类、请求日志 | 0 | 未接线 |
| `@openfoundry/tracing` | Span/TraceId、B3 头注入与提取 | 0 | 未接线，无 OTLP 导出 |
| `@openfoundry/redis` | ioredis 客户端、缓存、分布式锁、pub/sub | 0 | 未接线 |
| `@openfoundry/test-utils` | mock ontology/objects、测试 JWT、断言辅助 | 0 | 未接线 |

拆开「直接依赖者」这一列：`errors` = 13 个服务 + 7 个包（client、logging、permissions、pipeline、sdk、search、test-utils）；`rid` = 13 个服务 + 6 个包（errors、markings、permissions、pipeline、sdk、test-utils）；`pagination` = 6 个服务（ontology、objects、datasets、compass、admin、aip）+ sdk；`health` 只有 svc-gateway；`config` 只有 sdk-cli。

这张表最重要的一行是最后四行：**九个横切库里有四个没有任何消费者**。它们编译、测试通过（合计 193 个用例），但平台上没有一行运行时代码使用它们。文档随后会把这四个库与「真正承重」的三个库分开讲。

### 11.4 依赖关系与为什么不能反向依赖

```mermaid
flowchart LR
  subgraph core["承重层（被广泛依赖）"]
    errors["@openfoundry/errors"]
    rid["@openfoundry/rid"]
    pagination["@openfoundry/pagination"]
  end
  subgraph wired["单点接线"]
    health["@openfoundry/health"]
    config["@openfoundry/config"]
  end
  subgraph unused["零消费者（未接线）"]
    logging["@openfoundry/logging"]
    tracing["@openfoundry/tracing"]
    redis["@openfoundry/redis"]
    testutils["@openfoundry/test-utils"]
  end
  errors -->|"package.json 声明, 源码未 import"| rid
  logging --> errors
  testutils --> rid
  testutils --> errors
  consumers["13 个服务 + 7 个包<br/>errors 102 文件 / rid 47 / pagination 19"] --> core
  gateway["svc-gateway"] --> health
  sdkcli["sdk-cli"] --> config
```

依赖方向被刻意约束成一条单行道：`rid` 是叶子；`errors` 只在 `package.json` 里声明了 `rid`（源码从未 import，等于已失效）；`logging`/`test-utils` 在其上，服务与 SDK 在最外层。**反向依赖会立刻制造环**——`rid` 若依赖 `errors`，而 `errors` 的错误信封又要用 `rid` 格式化标识符，TypeScript 项目引用与 turbo 的 `^build` 拓扑会同时报循环。因此 `rid` 只抛原生 `Error`（`packages/rid/src/rid.ts:127`），不抛 `OpenFoundryApiError`；把它包装成 API 错误的责任留给调用方。代价是 RID 解析失败在服务里若没人 `catch`，会落进 500 分支而不是 400——这正是「解析失败应该由谁负责」的未决问题。

### 11.5 错误信封与 Fastify 映射（errors）

#### 11.5.1 线上格式

所有服务返回的错误体是同一个五字段信封，不是 Fastify 默认的 `{ statusCode, error, message }`：

```ts
// packages/errors/src/api-error.ts:7-13
export interface SerializedApiError {
  errorCode: ErrorCode;        // 分类，如 NOT_FOUND
  errorName: string;           // 具体名，如 FoundryBranchNotFound
  errorInstanceId: string;     // 单次发生的 UUID，便于对日志
  parameters: Record<string, unknown>; // 结构化上下文
  statusCode: number;          // HTTP 状态码
}
```

`OpenFoundryApiError.toJSON()` 把 `parameters` 与 `args` 的安全部分合并后输出这五个字段（`packages/errors/src/api-error.ts:73-84`）。一次真实的 branch 拒绝长这样（字段取自 `packages/errors/src/ontology-scoping.ts:38-50` 的构造）：

```json
{
  "errorCode": "NOT_FOUND",
  "errorName": "FoundryBranchNotFound",
  "errorInstanceId": "0f9f3f6e-8f0a-4a4e-9b0e-6f1f0f2c9d11",
  "parameters": { "branch": "experiment" },
  "statusCode": 404
}
```

#### 11.5.2 错误码目录

`ErrorCode` 是封闭的 TypeScript `enum`，每个码自带规范状态码（`packages/errors/src/error-codes.ts:8-28`）：

| 错误码 | 状态码 | 语义 | 生产使用量（工厂调用） |
|--------|--------|------|----------------------|
| `INVALID_ARGUMENT` | 400 | 参数非法 | `invalidArgument` 141 次 |
| `PERMISSION_DENIED` | 403 | 权限不足 | `permissionDenied` 7 次 |
| `NOT_FOUND` | 404 | 资源不存在 | `notFound` 138 次 |
| `CONFLICT` | 409 | 状态冲突 | `conflict` 50 次 |
| `INTERNAL` | 500 | 未预期错误 | `internal` 1 次 |
| `TIMEOUT` | 504 | 超时 | `timeout` 10 次 |
| `CUSTOM_CLIENT` | 400 | 4xx 兜底 | `customClient` 6 次 |
| `CUSTOM_SERVER` | 500 | 5xx 兜底 | `customServer` 2 次 |

`errorCodeFromStatus()` 负责反向映射（`packages/errors/src/error-codes.ts:42-64`），只精确匹配 400/403/404/409/500/504，其余 4xx 归 `CUSTOM_CLIENT`、5xx 归 `CUSTOM_SERVER`。因此 `401` 与 `429` 都会落到 `CUSTOM_CLIENT`——这也是为什么需要 `RATE_LIMITED` 这类手写码（见「已知限制」）。

#### 11.5.3 一个请求的错误路径

```mermaid
sequenceDiagram
  participant C as 调用方
  participant F as Fastify 服务
  participant H as setErrorHandler
  participant L as request.log
  C->>F: 请求
  F->>F: 路由/校验/存储
  alt 抛出 OpenFoundryApiError
    F->>H: error
    H->>L: warn({ err: error.toJSON(), requestId })
    H->>C: statusCode + 五字段信封
  else schema 校验失败
    F->>H: error.validation
    H->>C: 400 ValidationError + validationErrors
  else Fastify 请求级 4xx（仅网关与 aip）
    F->>H: error.statusCode 为 4xx
    H->>C: 原状态码 + InvalidRequest
  else 其它异常
    F->>H: error
    H->>L: error({ err, requestId })
    H->>C: 500 INTERNAL, parameters 为空
  end
```

四个分支在所有服务里形状一致，但**第三个分支只有两个服务有**：网关（`services/svc-gateway/src/server.ts:138-152`）与 `svc-aip`（`services/svc-aip/src/server.ts:127-141`）。两个文件的注释都写明理由：畸形 JSON、超限 body、不支持的 content type 是调用方的错，报成不透明 500 会掩盖「该修什么」。其余十一个服务直接落到最后的 500 分支。

第四个分支值得单独指出：它返回 `parameters: {}`，把真实原因只留在服务端日志里。这是有意的信息收窄——不把内部堆栈泄漏给调用方，但也意味着**调用方拿到的 500 永远无法自查**，只能凭 `errorInstanceId` 去服务端日志里对。

#### 11.5.4 4xx 透传 vs 500 包装

| 服务 | `OpenFoundryApiError` | schema 校验 | Fastify 请求级 4xx | 其它异常 |
|------|----------------------|-------------|--------------------|----------|
| svc-gateway | 原码 + `toJSON()` | 400 `ValidationError` | **透传** `InvalidRequest` | 500 `INTERNAL` |
| svc-aip | 原码 + `toJSON()` | 400 `ValidationError` | **透传** `InvalidRequest` | 500 `INTERNAL` |
| 其余 11 个服务 | 原码 + `toJSON()` | 400 `ValidationError` | 500 `INTERNAL` | 500 `INTERNAL` |

> 文档与实现不一致：`AGENTS.md` 说网关「passes through any 4xx Fastify request error」，这句话对网关成立，但容易读成平台级行为。实际只有网关与 svc-aip 实现了该分支，其余服务把空 body 配 `Content-Type: application/json` 这类调用方错误统一报成 500。

#### 11.5.5 ontology scoping：单点 fail-closed

Foundry 允许把 ontology 读写限定到 branch / scenario / ontology transaction；本仓库三者都没有实现。`packages/errors/src/ontology-scoping.ts` 因此选择**拒绝而不是忽略**：

- 传入 `branch` → `FoundryBranchNotFound`（404，名字与参数形状对齐官方 SDK 的 `foundry_sdk/v2/core/errors.py`）；
- 传入 `scenarioRid` → `OntologyScenarioNotFound`（404）；
- 传入 `transactionId` → `OntologyTransactionNotFound`（404）；
- 三者都不传（或传空串）→ 直接放行，端点行为不变。

策略点是 `rejectUnsupportedOntologyScoping()`（`packages/errors/src/ontology-scoping.ts:108-120`），被 10 个文件、共 18 处路由调用，其中 `svc-objects` 7 处、`svc-ontology` 9 处、`svc-actions` 2 处。模块头部注释（`packages/errors/src/ontology-scoping.ts:1-26`）写清了取舍：接受参数并返回唯一那份状态，等于对调用方撒谎说 `branch=experiment` 已被尊重。这也是全仓库唯一一处「为了诚实而主动报 404」的地方。

### 11.6 资源标识（rid）

RID 是 `ri.<service>.<instance>.<type>.<locator>` 的不可变值对象（`packages/rid/src/rid.ts:13-30`，构造后 `Object.freeze`）。关键设计是 **locator 允许点号**：`parse` 取前四段，其余全部并入 locator，从而支持 `ri.phonograph2-objects.main.object.v4.abc123.encoded-key` 这类层级子地址（`packages/rid/src/rid.ts:59-87`）。三段前缀各自校验：service/instance/type 匹配 `^[a-z0-9-]+$`，locator 匹配 `^[a-zA-Z0-9._-]+$`（`packages/rid/src/rid.ts:120-139`）。

API 面很小：`Rid.create(service, instance, type, locator)`、`Rid.parse(str)`、`toString()`、`equals()`、`toJSON()`（`toJSON` 让 `JSON.stringify` 直接输出字符串），外加 `generateRid()`（UUID v4 locator）与 `generateDeterministicRid()`（SHA-256 内容寻址，`packages/rid/src/rid-generator.ts:13-40`）。

实际使用的 `service.type` 组合（从 `generateRid`/`generateDeterministicRid` 调用点统计）：

| 前缀 | 资源 | 前缀 | 资源 |
|------|------|------|------|
| `ri.compass.main.folder` | 目录 | `ri.datasets.main.dataset` | 数据集 |
| `ri.compass.main.resource` | 资源引用 | `ri.datasets.main.branch` | 数据集分支 |
| `ri.compass.main.lineage-edge` | 血缘边 | `ri.datasets.main.transaction` | 数据集事务 |
| `ri.ontology.main.ontology` | Ontology | `ri.phonograph2-objects.main.object` | 对象实例 |
| `ri.ontology.main.object-type` | 对象类型 | `ri.multipass.main.user` | 用户 |
| `ri.ontology.main.action-type` | Action 类型 | `ri.multipass.main.group` | 用户组 |
| `ri.ontology.main.link-type` | Link 类型 | `ri.pipeline-builder.main.pipeline` | 流水线 |
| `ri.actions.main.execution` | Action 执行 | `ri.sentinel.main.monitor` | 监控器 |
| `ri.aip.main.conversation` | 会话 | `ri.webhooks.main.webhook` | Webhook |
| `ri.models.main.model` | 模型 | `ri.media.main.media` | 媒体项 |

`KnownServices` 只收录 19 个官方 Foundry 服务名（`packages/rid/src/known-services.ts:7-46`），且**不参与校验**：`generateRid("models", ...)`、`generateRid("magritte", ...)`、`generateRid("aip", ...)` 都用了表外的 service 段。校验只看字符集，不看白名单——这是有意的（注释写明第三方服务仍合法），代价是拼错的服务名不会被发现。

### 11.7 分页（pagination）

约定是 Foundry 的 `pageSize` / `pageToken` / `nextPageToken` 三件套。包内提供三层：

1. **游标编码**：`PageCursor` 是 `{ offset, lastId?, sortValues?, snapshotSize? }`，编码为 base64url 的 JSON（`packages/pagination/src/page-token.ts:10-48`）。解码做完整校验——offset 必须是非负有限数、`lastId` 必须是字符串、`snapshotSize` 必须是非负整数（`packages/pagination/src/page-token.ts:55-133`）。
2. **请求归一化**：`DEFAULT_PAGE_SIZE = 100`、`MAX_PAGE_SIZE = 10_000`，`normalizePageRequest()` 把 `pageSize` 夹到 `[1, 10000]`（`packages/pagination/src/page-request.ts:4-7,26-36`）。
3. **响应构造**：调用方多取一条判断是否还有下一页，`createPageResponse()` 据此决定是否签发 `nextPageToken`（`packages/pagination/src/page-response.ts:27-57`）。

`snapshotSize` 是最值得单独说的一笔：它记录「开始翻页时集合有多大」，后续页最多只读到这个位置，避免翻页途中新增的元素把已读元素挤到下一页。注释明确解释了为什么用计数而不是时间戳（`packages/pagination/src/page-token.ts:17-26`）——这与 `AGENTS.md`「不要用 createdAt 排序」是同一条教训的两次表达。

但分页是本章分叉最严重的地方：

| 实现位置 | token 格式 | pageSize 上限 | 非法 token |
|----------|-----------|---------------|-----------|
| `packages/pagination` | base64url JSON | 10000 | 解码抛错 |
| `svc-ontology`/`datasets`/`compass`/`admin` 的 `pagination-helpers.ts`（4 份**逐字节相同**，md5 `327c083c…`） | base64url JSON | 10000 | `invalidArgument` 400 |
| `svc-objects` 的 `paginate()`（`services/svc-objects/src/routes/v2/objects.ts:215-239`） | base64url JSON，**兼容裸十进制 offset** | 由调用方给 | 回退为 `parseInt` |
| `svc-sentinel`/`webhooks`/`media`/`functions` 的本地 `paginate()` | **裸十进制字符串** | **1000** | 静默归零 |

也就是说：同一个 `/api/v2/*` 面上，`nextPageToken` 有两种线上格式，`pageSize` 有两个上限。`svc-sentinel` 的实现只有十行（`services/svc-sentinel/src/routes/v2/monitors.ts:10-19`），把 `offset` 直接 `String()` 出去；而 `svc-objects` 为了兼容历史调用方，在解码失败时回退成 `parseInt`（`services/svc-objects/src/routes/v2/objects.ts:222-228`）。包里的 `paginate()`（异步迭代器）与 `collectAll()` 反而**没有任何生产调用者**。

### 11.8 日志（logging）

生产日志不是 `@openfoundry/logging`，而是 Fastify 内置的 pino。每个服务在 `Fastify({ logger: { level: config.logLevel } })` 里配置，`NODE_ENV=development` 时挂 `pino-pretty`（`services/svc-objects/src/server.ts:65-71`；`services/svc-gateway/src/server.ts:51-63`）。字段约定是 pino 默认的 `level/time/msg/reqId`，服务自己补 `requestId`：

- 请求 ID：`genReqId: () => crypto.randomUUID()` + `requestIdHeader: "x-request-id"`（十三个服务全部如此），并用 `onSend` 钩子把 `request.id` 写回响应头（`services/svc-objects/src/server.ts:82-87`）；
- 错误日志：`request.log.warn({ err: error.toJSON(), requestId }, message)`，**用的是对外信封而不是 `toInternalJSON()`**；
- 网关把上游 `x-request-id` 透传下去（`services/svc-gateway/src/proxy.ts:101`），这是唯一一条跨进程关联线索。

`@openfoundry/logging` 提供的是另一套东西：`Logger` 类按 `safe`/`unsafe` 两个字段分桶输出（`packages/logging/src/logger.ts:113-139`），`redactUnsafe` 为真时把 unsafe 值整体替换为 `[REDACTED]`（`packages/logging/src/logger.ts:142-148`）。它没有任何消费者。`no-console` 的三个库例外（`packages/logging/src/logger.ts`、`packages/redis/src/client.ts`、`packages/sdk/sdk-cli/src/output.ts`）在 `eslint.config.mjs:38-46` 里按文件路径放行，注释写的是「这些模块的职责就是打印」——与 `AGENTS.md` 的说法一致。

### 11.9 追踪（tracing）

`@openfoundry/tracing` 实现了最小可用的 B3 传播：`Tracer.startSpan()` 生成 32 位十六进制 traceId 与 16 位 spanId，有父 span 时继承 traceId（`packages/tracing/src/tracer.ts:23-31`）；`extract()` 读 `X-B3-TraceId`/`X-B3-SpanId`/`X-B3-ParentSpanId`/`X-B3-Sampled`，`inject()` 反向写出（`packages/tracing/src/tracer.ts:37-73`）。`Span` 支持 `finish()`/`setTag()`/`log()`/`setError()`（`packages/tracing/src/span.ts:38-62`）。

但它**没有接线，也没有导出**：全仓库零个 `@openfoundry/tracing` 引用，没有任何 OpenTelemetry/OTLP 依赖，`.env.example` 里的 `TRACING_ENABLED` / `TRACING_ENDPOINT=http://localhost:9411/api/v2/spans`（`.env.example:108-109`）也没有任何读取者。与 `x-request-id` 的关系是**并行而非包含**：`x-request-id` 是 Fastify 的请求 ID（UUID），用于单进程内日志关联；tracing 的 traceId/spanId 是另一套 ID，服务既不生成也不传播它。

> 文档与实现不一致：`.env.example` 把追踪描述成可通过 `TRACING_ENABLED` 打开、向 Zipkin 端点上报的能力，但代码中不存在任何 span 导出路径。

### 11.10 健康检查与指标（health）

`/status/health`、`/status/liveness`、`/status/readiness` 在每个服务里都是**常量响应** `{ status: "HEALTHY" }`（`services/svc-objects/src/routes/health.ts:16-28`），不查数据库、不查下游。网关的 readiness 注释直接写着「将来可以检查下游依赖并返回 503」（`services/svc-gateway/src/routes/health.ts:51-55`）。`AGENTS.md` 提到容器健康检查用的是 `/status/health`，与这套常量语义吻合：它只证明「进程还能接 HTTP」。

`@openfoundry/health` 的 `HealthChecker` 支持并行跑多个检查、把失败捕获成 `ERROR` 而不是抛出（`packages/health/src/health-checker.ts:28-53`），`combineHealthChecks()` 按 7 态严重度取最坏值（`packages/health/src/health-status.ts:7-15,57-86`）。这套模型**没有被任何服务使用**。

真正被使用的是 `metricsPlugin`，只在网关注册一次（`services/svc-gateway/src/server.ts:184`）。它挂 `onRequest` 记开始时间、`onResponse` 记耗时与状态码，并注册 `GET /metrics`（`packages/health/src/metrics.ts:151-186`）。指标清单：

| 指标 | 类型 | 标签 | 说明 |
|------|------|------|------|
| `http_requests_total` | counter | `method`、`route`、`status` | 全部请求 |
| `http_errors_total` | counter | `method`、`route`、`status` | 仅 4xx/5xx |
| `http_request_duration_seconds` | histogram | 同上 + `le` | 11 个桶（5ms–10s）+ `+Inf` |

标签取 `request.routeOptions.url`（路由模板，如 `/api/v2/objects/:objectType`）而不是原始 URL，注释说明是为了控制基数（`packages/health/src/metrics.ts:168-172`）。这个选择是对的，但它同时意味着**未匹配路由的 404 会以原始 URL 作为标签**——扫描器打随机路径时基数会失控。

网关另有两个指标入口：`/metrics/:service` 反代下游 `/metrics`（`services/svc-gateway/src/routes/health.ts:86-142`），以及 `/status/alerts` 心跳（`services/svc-gateway/src/routes/health.ts:57-80`）。前者依赖下游服务自己暴露 `/metrics`，而下游服务的 `healthRoutes` 只注册了三个 `/status/*`——没有任何服务注册 `metricsPlugin`。按静态代码推断，这条代理路径会把下游的 404 原样透传（未实测）。

### 11.11 配置装载（config）

平台里有**两套互不相干的配置机制**：

| 维度 | `@openfoundry/config` | 各服务 `src/config.ts` |
|------|----------------------|----------------------|
| 装载顺序 | 默认值 → `foundry.config.json` → 环境变量 | 环境变量 → 硬编码默认值 |
| 变量命名 | `OPENFOUNDRY_*`（`packages/config/src/env-mapping.ts:39-66`，23 个映射） | 裸名：`PORT`、`HOST`、`LOG_LEVEL`、`DATABASE_URL`… |
| 校验 | 数字/布尔/数组强制转换，非法数字抛错 | `envInt()` 非法即抛，`env()` 空串视同缺省 |
| 使用者 | 仅 `sdk-cli` | 13 个服务全部 |
| 规模 | 98 行 loader + 78 行 schema + 107 行映射 | 13 个文件、合计 760 行 |

服务侧的重复模式是：一个 `env()`（空串按缺省处理）、一个 `envInt()`（非数字抛错）、一个 `loadConfig()` 返回只读形状。网关的注释专门解释了空串语义（`services/svc-gateway/src/config.ts:72-82`）：`.env` 里一行被清空会产出空串，若当作「显式设成空」就会把 `AUTH_ISSUER` 校验悄悄关掉。`PORT` 的默认值直接写死在每个服务里（`services/svc-objects/src/config.ts:32`、`services/svc-gateway/src/config.ts:96`），所以 `.env.example` 里的 `GATEWAY_PORT`/`ONTOLOGY_PORT` 等**没有任何读取者**。

> 文档与实现不一致：`.env.example:76-86` 用 `GATEWAY_PORT`/`ONTOLOGY_PORT`/`OBJECTS_PORT`… 声明端口，`CONSOLE_PORT` 亦然，但所有服务只读 `PORT`，控制台只读 Vite 的 `server.port`；`LOG_FORMAT`（`.env.example:105`）、`JWT_SECRET` 同样无读取者。控制台读的是 `VITE_API_URL`（`apps/app-console/src/config.ts:8`），而 `.env.example:91` 写的是 `VITE_API_BASE_URL`。

`packages/config` 的默认端口也已过时：`DEFAULT_CONFIG` 声明 multipass 8081、ontology 8082、objects 8083、actions 8084（`packages/config/src/config-schema.ts:70-76`），而服务实际监听 8084/8081/8082/8083（网关 `services/svc-gateway/src/config.ts:104-117` 的下游地址表是准的）。因为该包没有服务在用，这个错误目前不会触发任何故障。

### 11.12 Redis 客户端（redis）

`createRedisClient()` 优先用 `config.url ?? process.env.REDIS_URL`，否则拼 `host/port/password/db`，统一 `maxRetriesPerRequest: 3`，并在 connect/error/close 三个事件上打印（`packages/redis/src/client.ts:13-47`）。同包还有 `RedisCache`、`RedisLock`、`RedisPubSub`。

它同样**零消费者**。真正用到 Redis 的是网关限流器：`RedisRateLimiter` 动态 `await import("ioredis")`，失败就静默回退到内存桶（`services/svc-gateway/src/rate-limiter.ts:142-157`）。`ioredis` 在网关是 `optionalDependencies`，所以未装 Redis 依赖也能启动——这是「Redis 永不必须」的实现方式（`README.md:38`），但它绕过了横切库，重复实现了连接参数。

### 11.13 测试辅助（test-utils）

包内是四组纯数据/纯函数辅助：`createMockOntology()` 与三个预置对象类型、`createMockEmployee()` 等对象工厂、`createTestKeyPair()`/`createTestToken()`/`createExpiredToken()`（ES256，`packages/test-utils/src/mock-auth.ts:43-87`）、`expectApiError()`/`expectRid()` 断言（`packages/test-utils/src/assertions.ts:11-72`）。

> 文档与实现不一致：任务与命名预期中的「测试服务器工厂」并不存在。全仓库没有任何 `createTestServer`/`buildTestApp`；服务测试都是直接调用各服务导出的 `createServer()` 并注入 `new ObjectStore(null)` 之类的空存储（如 `services/svc-objects/tests/objects-service.test.ts:48-55`）。`test-utils` 自身也是零消费者：这 511 行源码只有它自己的测试在引用。

「不落共享文件」的约定因此不在 `test-utils` 里，而在各存储的构造函数签名里：`dataFile: string | null = "/tmp/openfoundry-data/…"`，传 `null` 即纯内存（`services/svc-ontology/src/store/ontology-store.ts:64`、`services/svc-objects/src/store/object-store.ts:308`、`services/svc-objects/src/store/link-store.ts:44`、`packages/permissions/src/authorization-store.ts:82`）。测试里 `new *Store(null)` 共出现 47 次。

### 11.14 设计决策与理由

1. **错误体对齐 Foundry 的生成式 SDK，而不是 Fastify 默认体。** 五字段信封让 `errorCode` 可被程序分支、`errorName` 可被人读、`errorInstanceId` 可对日志。代价是每个服务都要维护一份 `setErrorHandler`（13 份近 30 行的重复），且 Fastify 的 `validation` 分支要手写成同一形状。
2. **`SafeArg`/`UnsafeArg` 用编译期 brand 分型，脱敏发生在序列化期。** `toJSON()` 走 `toSafeRecord()` 把 unsafe 值替换成 `[REDACTED]`，`toInternalJSON()` 才保留全量（`packages/errors/src/safe-arg.ts:99-116`）。好处是「会不会泄漏」由类型决定而不是由调用者记性决定；代价是平台代码里 `unsafeArg` 一次都没用过（只在 logging 包测试里出现），机制存在但从未被触发。
3. **不支持的 ontology scoping 一律 fail-closed 成 404。** 拒绝而不是忽略，是为了不把「你请求了 branch=experiment」和「我给了你唯一那份生产数据」混为一谈。代价是调用方若误传参数会直接失败，而 Foundry 客户端在 `None` 时本就不发参数，所以真实影响面很小。
4. **4xx 透传只在边缘做。** 网关是对外契约的最后一跳，把 Fastify 请求级 4xx 如实透传；下游服务内部则统一 500，避免把内部路由细节暴露给网关之外的调用方。代价是直连服务端口时排障体验差（`AGENTS.md` 也记录了「服务端口不应对外可达」）。
5. **分页 token 设计成不透明 base64url JSON。** 相比裸 offset，它能在不破坏调用方的前提下增加 `snapshotSize` 这类字段。代价是「不透明」没有被贯彻：一半服务直接发十进制 offset，客户端无法用同一套逻辑翻所有列表。
6. **指标标签用路由模板而非原始 URL。** 这是 Prometheus 基数控制的常规做法，直接决定了 `/metrics` 在真实流量下会不会把内存打爆。
7. **每个服务自带 `config.ts` 而不共享 `@openfoundry/config`。** 换来零运行时依赖、启动即失败（非法 `PORT` 直接抛），以及「读哪几个变量」在服务目录内一眼可见。代价是 13 份 45–60 行的重复，以及 `packages/config` 与真实环境彻底脱节。
8. **`rid` 保持不可变值对象、locator 允许点号。** 让 RID 可以安全地当 Map key 与 JSON 值，同时容纳 Foundry 的层级子地址（`v4.abc123.encoded-key`）。

### 11.15 已知限制与坑

1. **四个库零消费者。** `logging`、`tracing`、`redis`、`test-utils` 没有任何 `package.json` 声明或源码引用；它们只被自己的测试覆盖。平台因此没有结构化业务日志约定、没有分布式追踪、没有共享 Redis 抽象。`SafeArg` 除了 `errors` 自己的工厂函数，只在 4 个服务路由文件里被用了 5 次，`unsafeArg` 在生产代码中一次都没出现。
2. **错误码目录不封闭。** `ErrorCode` 只有 8 个值，但服务里手写的信封用了 `UNAUTHORIZED`（`services/svc-multipass/src/routes/auth.ts:54`）、`RATE_LIMITED`（`services/svc-gateway/src/rate-limiter.ts:320`）、`BAD_GATEWAY`（`services/svc-gateway/src/proxy.ts:126`、`services/svc-gateway/src/routes/health.ts:134`）、`SERVICE_UNAVAILABLE`（`services/svc-gateway/src/routes/health.ts:106`、`services/svc-objects/src/routes/v2/semantic-search.ts:134`）、`UPSTREAM_ERROR`（`services/svc-objects/src/routes/v2/semantic-search.ts:95`）。这些码不在目录里，`errorCodeFromStatus()` 也不会产出它们，客户端若按枚举分支就会漏判。
3. **十一个服务把 Fastify 4xx 变成 500。** 只有网关与 `svc-aip` 有透传分支；空 body 带 JSON content-type、超大 body、畸形 JSON 在其它服务里都是 500。
4. **`/metrics` 只在网关存在。** 下游服务没有注册 `metricsPlugin`，所以 `/metrics/:service` 代理到的是一个不存在的端点；容器健康检查用 `/status/health`，Prometheus 抓取配置不在本仓库（`deploy/` 中无 scrape 配置）。
5. **健康检查不检查依赖。** `/status/readiness` 恒为 `HEALTHY`，数据库断开时仍返回 200；`HealthChecker`/`createDatabaseCheck`/`createHttpCheck` 全部未使用。
6. **分页有 10 份本地实现、2 种 token 格式、2 个 `pageSize` 上限**（10000 vs 1000），见前文表格；`pagination` 包的异步迭代器 `paginate()`/`collectAll()` 与 `lastId`/`sortValues` 字段无生产使用者。
7. **`packages/config` 与服务实际配置完全脱节**，且默认端口与真实端口不符；`@openfoundry/errors` 的 `package.json` 声明了 `@openfoundry/rid` 依赖但源码从未 import（幽灵依赖）。
8. **限流的按租户分桶是死代码。** `extractKey()` 读 `request.orgRid`（`services/svc-gateway/src/rate-limiter.ts:273-277`），但 `rateLimitPlugin` 在 `authPlugin` 之前注册（`services/svc-gateway/src/server.ts:96` 与 `services/svc-gateway/src/server.ts:181`），同一上下文里 `onRequest` 按注册顺序执行，`orgRid` 此时必然 `undefined`，于是每个请求都退化成按 Authorization 头哈希分桶——换一次 token 就是一个新桶。`AGENTS.md` 记录该顺序是有意保留的取舍。
9. **`@openfoundry/health` 的文档注释宣传了一个不存在的子路径导出。** `packages/health/src/metrics.ts:147` 写 `import { metricsPlugin } from "@openfoundry/health/metrics"`，但 `packages/health/package.json` 的 `exports` 只有 `"."`。

### 11.16 关键代码位置

- `packages/errors/src/api-error.ts:41-124` — 错误类、`toJSON()`/`toInternalJSON()`、`fromResponse()` 反序列化。
- `packages/errors/src/error-codes.ts:8-64` — 8 个错误码、状态码映射与反向映射。
- `packages/errors/src/ontology-scoping.ts:108-120` — 三个 scoping 参数的唯一拒绝点。
- `services/svc-gateway/src/server.ts:115-166` — 完整四分支错误处理器（含 4xx 透传）。
- `services/svc-objects/src/server.ts:90-125` — 其余服务的三分支版本（无 4xx 透传）。
- `packages/rid/src/rid.ts:59-87` — RID 解析与 locator 点号合并规则。
- `packages/pagination/src/page-token.ts:10-133` — 游标结构与 base64url 编解码/校验。
- `packages/health/src/metrics.ts:28-186` — 指标收集器与 `metricsPlugin`。
- `services/svc-gateway/src/config.ts:72-126` — `env()`/`envInt()` 空串语义与全部环境变量。
- `packages/test-utils/src/index.ts:1-25` — 该包实际导出的全部内容（无服务器工厂）。

---

## 第 12 章 测试体系与质量门禁

### 12.1 本章导读

这一层回答「改动凭什么被相信」：88 个包内测试文件覆盖单个模块，12 个跨服务集成套件覆盖真实 HTTP 链路，5 个 PGlite 套件把内存实现与真实 Postgres 放在同一份契约下执行，最后用 build/typecheck/lint/test 四道门禁把结果钉住。本章的重点不是「测试很多」，而是**哪些门禁真的会失败、哪些只是显示为绿色**。

### 12.2 测试分层

```mermaid
flowchart TD
  U["包内单测<br/>88 个 *.test.ts / 47 个包<br/>vitest run, root = 包目录"]
  I["跨服务集成<br/>tests/integration/*.test.ts, 12 个套件"]
  A["自带 in-process server<br/>7 个套件, 可离线"]
  B["真实网关 localhost:8080<br/>api-endpoints / sdk-compat 硬失败<br/>kelava-sync / tenant-isolation 跳过"]
  C["纯编译器与协议<br/>conjure-pipeline"]
  P["Pg 真实库<br/>PGlite + createPgConfig"]
  S["双 schema 源<br/>db/migrations 与 scripts/migrate.sql"]
  M["models-contract<br/>仅 MODELS_TEST_DATABASE_URL 存在时"]
  U --> I
  I --> A
  I --> B
  I --> C
  U --> P
  P --> S
  P --> M
```

金字塔的底座不是「更多单测」，而是**同一份契约在多个后端上执行**：`describe.each` 把 memory 与 Pg 两个后端跑成两组用例，PGlite 又把「真实 Postgres」压进单测进程，于是「只在 Postgres 上炸」的缺陷不必等到集成环境才暴露。

### 12.3 包内单测

机制只有一条：每个包声明 `"test": "vitest run"`，运行时继承根 `vitest.config.ts`。根配置做两件事——把每个 `packages/<dir>` 映射成 `@openfoundry/<dir>` → `packages/<dir>/src/index.ts` 别名，让测试直接跑源码；以及声明 `include: ["**/tests/**/*.test.ts", "**/src/**/*.test.ts"]`（`vitest.config.ts:19-30`）。

关键在注释里（`vitest.config.ts:24-27`）：`include` 是相对**本次运行的 root** 解析的，而包内 `vitest run` 的 root 是包目录。所以 glob 必须写成相对形式 `**/tests/**`；若写成根锚定的 `packages/**`，每个包都会得到 "No test files found"。`AGENTS.md` 特意提醒读那条报错时要分清：它当时意味着 glob 没匹配上，而不是测试不存在。

资产分布（按 `package.json` 里的 `test` 脚本统计）：

| 维度 | 数量 | 说明 |
|------|------|------|
| 工作区包总数 | 49 | `packages/*`、`packages/sdk/*`、`packages/ui/*`、`services/*`、`apps/*` |
| 声明 `test` 脚本 | 48 | 唯一例外是 `apps/app-docs`（有 build、无 test） |
| 含测试文件的包 | 47 | 唯一没有测试的是 `@openfoundry/app-workshop`，它的脚本是 `echo 'no tests'`（`apps/app-workshop/package.json:12`） |
| 包内测试文件 | 88 | 最多的三个：`svc-gateway` 9、`app-console` 7、`svc-datasets` 6 |
| 横切库自身用例 | 193 | errors 32、rid 32、pagination 27、redis 25、config 20、test-utils 16、logging 15、tracing 13、health 13 |

`apps/app-console` 是 glob 规则的例外：它没有 `vitest.config.ts`，`vitest run` 解析到 `apps/app-console/vite.config.ts`，而该文件没有 `test` 块，于是 vitest 退回自己的默认 include 规则，而不是根配置的 `**/tests/**` + `**/src/**`。它的 7 个测试文件因此全部位于 `src/lib/*.test.ts`，实测 7 文件 57 个用例通过。

### 12.4 跨服务集成测试

`pnpm run test:integration` = `vitest run -c tests/vitest.config.ts`（`package.json:23`）。该配置把 `root` 设为 `./tests`、`include` 设为 `integration/**/*.test.ts`（`tests/vitest.config.ts:24-28`），因此 `tests/sdk-compat/` 目录**不在任何测试命令的覆盖范围内**（见限制）。

| 套件 | 用例数 | 依赖 | 覆盖点 |
|------|-------|------|--------|
| `api-endpoints` | 37 | 真实网关 + **种子数据** | v1/v2 端点逐个走查、鉴权头、错误码 |
| `sdk-compat` | 17 | 真实网关 | 官方 `@osdk/foundry` v2 SDK 契约 |
| `full-platform` | 12 | 自带 server | 多服务组合的主流程 |
| `conjure-pipeline` | 14 | 纯编译器 | Conjure YAML → IR → TypeScript |
| `dataset-lifecycle` | 12 | 自带 server | 数据集/分支/事务生命周期 |
| `kelava-sync` | 9 | 网关（可跳过） | 外部同步脚本的 upsert 与重复拒绝 |
| `functions-execution` | 9 | 自带 server | 函数执行链路 |
| `compass-navigation` | 9 | 自带 server | 目录树与资源导航 |
| `e2e` | 6 | 自带 server | Ontology 端到端 |
| `ontology-sdk` | 6 | 自带 server | Ontology SDK 形状 |
| `sentinel-monitors` | 6 | 自带 server | 监控器与执行日志 |
| `tenant-isolation` | 3（+3 todo） | 网关 + 数据库 | 租户隔离规格 |

「自带 server」的套件直接 import 服务源码里的 `createServer()` 并注入 `new OntologyStore(null)` / `new ObjectStore(null)`，因此不碰 `/tmp/openfoundry-data/`，可以离线跑。`api-endpoints` 与 `sdk-compat` 则直接 `fetch("http://localhost:8080")`，`beforeAll` 里没有探测、没有 `skipIf`——网关不在就整片失败。

两个网关依赖套件的行为差异值得注意：`kelava-sync` 先用一次探测算出 `GATEWAY_UP`，再用 `describe.skipIf(!GATEWAY_UP)` 跳过（`tests/integration/kelava-sync.test.ts:78-114`）；`tenant-isolation` 用 `SUITE_READY` 做同样的事，并且把真正的隔离断言写成 `it.todo`（`tests/integration/tenant-isolation.test.ts:1-17,104,161`）——文件头明说「todo 不是证据」。

> 文档与实现不一致：`AGENTS.md` 说「`api-endpoints` 和 `sdk-compat` need a real gateway」。实际有 4 个套件默认指向 `localhost:8080`：`kelava-sync`（`tests/integration/kelava-sync.test.ts:12`）与 `tenant-isolation`（`tests/integration/tenant-isolation.test.ts:21`）也读同一个 `OPENFOUNDRY_HOST` 默认值，只是它们会跳过而不是失败。只有前两个是「硬依赖」。

#### 12.4.1 种子数据与非幂等

`api-endpoints` 的第一个 Ontology 用例只做「列表里应该有东西」，然后**从响应里取第一条**存进模块级变量：

```ts
// tests/integration/api-endpoints.test.ts:117-127
if (data.data.length > 0) {          // 空库时整段跳过赋值
  const ont = data.data[0];
  ontologyRid = ont.rid;
  ontologyApiName = ont.apiName;
}
```

后续 20 个用例用 `expect(ontologyRid).toBeDefined()` 开头（`tests/integration/api-endpoints.test.ts:131`），于是空库会把一个「没取到数据」放大成 20 个级联失败。正确姿势是先跑任一 `scripts/demo/seed-*.sh`；`AGENTS.md` 记录的基线是有种子时 36/37，唯一失败是 Actions apply 返回 404 而测试期望 400，且该失败早于这条记录。

对运行中的网关，整套集成测试**不幂等**：第二次跑会在已存在的 ontology 上撞 409，对象计数也会被上一次的数据抬高。复位约定是 `rm -rf /tmp/openfoundry-data`（四类存储的默认路径都指向这个目录，见第 11 章），`start.sh` 的 `--industry` 切换则清三个 store JSON 后重新 seed。

### 12.5 Pg 集成测试：PGlite

把真实 Postgres 塞进单测的做法是 `@electric-sql/pglite` + `pglite-pgvector`：`PGlite.create({ extensions: { citext, pgcrypto, pg_trgm, vector } })` 建一个进程内 Postgres，再用 `createPgConfig(db)` 生成一个 `pg.Pool` 的连接配置，让被测的 `Pg*Store` 以为自己连的是真库。

`services/svc-multipass/tests/clients-and-users.test.ts` 是完整范例，且关键在 `describe.each`：

```ts
// services/svc-multipass/tests/clients-and-users.test.ts:195-213（节选）
const SCHEMAS = [
  ["db/migrations", migrationFiles],
  ["scripts/migrate.sql", () => [path.join(REPO_ROOT, "scripts/migrate.sql")]],
];
describe.each(SCHEMAS)("PgUserStore on a database built from %s", (_name, files) => {
  beforeAll(async () => {
    db = await PGlite.create({ extensions: { citext, pgcrypto, pg_trgm, vector } });
    for (const file of files()) await db.exec(readFileSync(file, "utf8"));
    pool = new pg.Pool(createPgConfig(db));
    store = new PgUserStore(pool);
  }, 60_000);
```

同一份 `PgUserStore` 用例跑两遍：一遍建在 `db/migrations/*.sql` 上，一遍建在 `scripts/migrate.sql`（Docker Compose 初始化用的合并脚本）上。这直接对应 `AGENTS.md` 里最贵的一类缺陷——`PgFileStore.putFile` 对着没有 `updated_at` 列的表写了几个月，而内存测试全绿。**用真实库执行，而不是比对 SQL 文本**，是唯一能抓住「代码里的列名在任何 schema 里都不存在」的办法。

`services/svc-aip/tests/models-pg-schema.test.ts` 则专门守另一条线：它是**纯文本**断言，`expect(consolidated).toContain(migration)`（`services/svc-aip/tests/models-pg-schema.test.ts:19-24`），保证 `scripts/migrate.sql` 逐字节包含 `db/migrations/012_models.sql`。两条线分工明确：文本测试守「两份 schema 不许漂移」，PGlite 测试守「store 在任一份 schema 上真能跑」。`models-contract.test.ts` 是后者的加强版：memory 后端永远跑，Postgres 后端仅在 `MODELS_TEST_DATABASE_URL` 存在时通过 `describe.skipIf(PG_URL === "")` 启用（`services/svc-aip/tests/models-contract.test.ts:47,220,872`）。

用 PGlite 的套件共 5 个：`packages/permissions/tests/authorization.test.ts`、`svc-datasets/tests/connectivity-contract.test.ts`、`svc-gateway/tests/search-permissions.test.ts`、`svc-multipass/tests/clients-and-users.test.ts`、`svc-objects/tests/ontology-scoping.test.ts`。

### 12.6 测试运行矩阵

| 命令 | 覆盖范围 | 前置条件 | 种子 / 网关 / PG |
|------|---------|---------|-----------------|
| `pnpm --filter @openfoundry/<pkg> test` | 单包 `tests/` + `src/` | 无 | 均不需要 |
| `pnpm run test` | 48 个 test 任务（+49 个 build 依赖 = 97） | 先构建 | 均不需要；PGlite 自包含 |
| `pnpm run test:integration` | 12 个集成套件 | 视套件而定 | 4 个套件要网关；1 个要种子 |
| `bash scripts/test-integration.sh` | 同上，先探测网关可达性再跑 | 网关在 8080 | 需要网关 |
| `MODELS_TEST_DATABASE_URL=… pnpm --filter @openfoundry/svc-aip test` | models v2 的 Pg 半边 | 可写 scratch 库 | 需要 PG |
| `npx tsx tests/sdk-compat/test-osdk-compat.ts` | SDK 契约手工脚本 | 网关 | 需要网关（无 npm script） |
| `pnpm run lint` / `typecheck` / `build` | 全仓 | 无 | 均不需要 |

超时统一 30 s（两份 config 都设 `testTimeout: 30000`），并发用 vitest 默认线程池（没有 `maxWorkers`/`pool` 覆盖），watch 模式由各包 `test:watch` 提供。

### 12.7 质量门禁

四道门禁都在 `package.json:8-13`，CI 在 `.github/workflows/ci.yml` 里逐条对应：lint（`.github/workflows/ci.yml:24`）、typecheck（`:37`）、test（`:68`）、build（`:81`），全部触发于 `master` 的 push 与 PR。

| 检查项 | 命令 | 当前基线 | 失败后果 |
|--------|------|---------|---------|
| build | `pnpm run build`（turbo） | **49/49** | 阻断；且 `test`/`typecheck` 依赖 build，会一起失败 |
| typecheck | `pnpm run typecheck`（turbo） | **45/45** | 阻断 |
| lint | `pnpm run lint`（`eslint .`） | **0 error / 225 warning** | 仅 error 阻断 |
| test | `pnpm run test`（turbo） | **97/97** | 阻断 |
| prettier | `pnpm run format` | **不校验** | 无 |

这几个数字的构成可以直接从 turbo 任务图核对：build 图 49 个任务、0 个 `<NONEXISTENT>`；test 图 98 个任务，其中 48 个真实 test + 49 个 build 依赖 = **97**；typecheck 图 70 个任务，其中 24 个真实 typecheck + 21 个传递 build = **45**。`apps/app-docs` 有 build 无 test，turbo 仍为它生成一个 `<NONEXISTENT>` 的 test 节点，但不会计入成功数——这解释了 48 个 test 脚本为何汇总成 97 而不是 98。

lint 基线（本次实跑 `npx eslint . -f json`，只读）为 644 个文件、0 error、225 warning，构成与 `AGENTS.md` 完全一致：181 `no-explicit-any`、33 `no-unused-vars`、11 `react-hooks/exhaustive-deps`，集中在 46 个文件里。`AGENTS.md` 说「632 个 `.ts`/`.tsx`/`.mts` 文件」，实际仓库有 644 个——数字过时了 12 个，但警告构成仍然准确。

> 文档与实现不一致：`AGENTS.md` 记的 632 个待检查文件数已过时（实际 644）；`no-console` 的三个库例外、react-hooks 只启用两条经典规则、prettier 不校验这三点与 `eslint.config.mjs` 一致。

### 12.8 设计决策与理由

1. **单一根 vitest 配置 + 每包相对 glob，拒绝 per-package 配置。** 一个根配置意味着没有任何包能悄悄退出测试；glob 写成相对形式又让同一个配置在「仓库根」与「包目录」两种 root 下都成立。代价是 glob 语义反直觉，只能靠注释和 `AGENTS.md` 传承。
2. **契约写一次、多后端执行。** `describe.each` 把 memory / Pg / 两种 schema 源组合成矩阵，而不是为每个后端各写一套断言。好处是断言不会漂移；代价是用例必须抽象到「后端无关」的接口（`ObjectReadWriteStore` 的 `T | Promise<T>` 正是为此存在）。
3. **用 PGlite 而不是 Docker 跑 Pg 半边。** 进程内 Postgres 让 `pnpm run test` 在无 Docker 的机器上也覆盖真实 SQL；CI 里再补一个 `postgres:16` service 与 `MODELS_TEST_DATABASE_URL`（`.github/workflows/ci.yml:44-59`），覆盖 PGlite 覆盖不到的连接池/RLS 场景。
4. **集成测试分两种风格。** 自带 in-process server 的套件快、可离线、能精确注入存储；真实网关的套件慢、要 seed，但能验证代理、鉴权、序列化整条链。两者都要，而不是只留一种。
5. **测试用 `null` 存储路径隔离共享状态。** 默认路径 `/tmp/openfoundry-data/` 是全局单例，测试必须显式关掉，否则前一次运行的残留会让断言随机失败。这条约定写在构造函数签名里而不是测试工具里，所以 47 处调用各自显式传 `null`。
6. **门禁只把 error 当失败，warning 只可见不阻断。** 这让 225 条存量警告不会阻塞任何人，但也意味着警告数可以无限增长而无人负责——`AGENTS.md` 自己就写了「a growing warning count is visible but unguarded」。

### 12.9 已知限制与坑

1. **glob 相对 root 的坑。** 根配置的 `include` 以本次运行的 root 解析；包内运行时 root 是包目录。任何把它改成根锚定 `packages/**` 的「优化」都会让每个跑 vitest 的包同时报 "No test files found"。同一类坑在 `apps/app-console` 反向出现：它因为没有 `vitest.config.ts` 而走 `vite.config.ts`，退回 vitest 默认 glob，既不继承根配置的别名也不继承 `testTimeout`。
2. **`--passWithNoTests` 陷阱。** `AGENTS.md` 记录有人差点用它把「glob 没匹配上」粉饰成绿色。当前 `packages/ui/components`（`packages/ui/components/package.json:19`）与 `packages/ui/icons`（`packages/ui/icons/package.json:17`）仍带着这个 flag——它们今天确实有测试（各 1 个文件），所以 flag 暂时不掩盖任何东西；但一旦某次重构让 glob 失效，这两个包会静默通过。`apps/app-workshop` 用的是 `echo 'no tests'`，性质相同但至少一眼可见。
3. **集成测试对运行中的网关不幂等。** 第二次运行会撞 409、对象计数被上一次抬高；必须 `rm -rf /tmp/openfoundry-data` 复位。`api-endpoints` 还额外依赖**种子数据**，空库会把一个「列表为空」放大成 20 个级联失败。
4. **有种子时 `api-endpoints` 也不是全绿。** `AGENTS.md` 记录 36/37，唯一失败是 Actions apply 返回 404 而测试期望 400；该失败先于这条记录存在，说明它没有被修，只是被记录。
5. **租户隔离没有证据。** `tenant-isolation` 的隔离断言全是 `it.todo`（`tests/integration/tenant-isolation.test.ts:161,184`），文件头明写「neither a todo nor a skipped suite may be read as evidence」。真正跑起来的只有「signup 的 JWT `org` 声明与返回的 orgRid 一致」。
6. **`tests/sdk-compat/` 不在任何测试命令内。** `tests/vitest.config.ts` 的 include 只有 `integration/**/*.test.ts`，而 `tests/sdk-compat/test-osdk-compat.ts`（511 行）与 `.mts` 版本既不在该目录下、也不以 `.test.ts` 结尾；它们只能手工 `npx tsx` 运行（文件头注释给了用法）。真正进 CI 的是 `tests/integration/sdk-compat.test.ts`（331 行，17 个用例）。
7. **CI 与本地不一致。** 本地 `pnpm run test` 默认跳过 models 的 Postgres 半边（无 `MODELS_TEST_DATABASE_URL`），CI 的 test job 配了 `postgres:16` 并注入该变量（`.github/workflows/ci.yml:44-59`）——同一命令在两边覆盖范围不同。集成侧同样：`test.yml` 只起 9 个服务（`.github/workflows/test.yml:56`，无 compass/webhooks/media/sentinel）、只 seed pest-control（`.github/workflows/test.yml:90-95`），然后跑 `npx vitest run tests/integration/`（`.github/workflows/test.yml:98`）——注意它**没有** `-c tests/vitest.config.ts`，走的是根配置 + 路径过滤，与本地 `pnpm run test:integration` 不是同一条解析路径。
8. **没有覆盖率门禁。** 仓库有 `coverage/` 的 gitignore 条目，但没有 `@vitest/coverage-*` 依赖、没有 coverage 脚本、CI 不上传覆盖率，也没有 codecov/sonar 配置。「测试很多」与「覆盖了多少」在本仓库无法回答。
9. **门禁不含集成测试的强制通过。** `pnpm run test` 不含 `tests/integration`；集成由独立的 `test.yml` 承担，而它依赖 `bash` 起 9 个服务、等网关 ready、seed 成功——任一环节脆弱都会让整条工作流红或假绿（seed 步骤甚至写了 `|| echo "Seed script had warnings (non-fatal)"`）。

### 12.10 关键代码位置

- `vitest.config.ts:19-30` — 根配置：别名生成、相对 include glob、30 s 超时。
- `tests/vitest.config.ts:20-28` — 集成配置：`root: ./tests`、`integration/**/*.test.ts`。
- `turbo.json:15-19` — `test` 任务的 `dependsOn: ["build"]` 与 `MODELS_TEST_DATABASE_URL` 环境声明。
- `.github/workflows/ci.yml:12-81` — lint/typecheck/test/build 四个 job，其中 test job 带 Postgres service。
- `.github/workflows/test.yml:49-98` — 起 9 个服务、seed、跑集成。
- `tests/integration/api-endpoints.test.ts:117-131` — 「读第一个已存在 ontology」导致空库级联失败的位置。
- `services/svc-multipass/tests/clients-and-users.test.ts:195-213` — PGlite + `createPgConfig` + 双 schema `describe.each`。
- `services/svc-aip/tests/models-pg-schema.test.ts:19-24` — 两份 schema 源的逐字节包含断言。
- `services/svc-aip/tests/models-contract.test.ts:47,220,872` — `MODELS_TEST_DATABASE_URL` 驱动的 `skipIf` 双后端契约。
- `packages/ui/components/package.json:19` — `--passWithNoTests` 仍在两个 UI 包里的证据。

---

## 第 13 章 架构评估、已知限制与风险

### 13.1 评估方法与分级

本章不再重复前文的实现细节，而是把散落在各章的"会坏的地方"收敛成一份可执行的清单。评估沿五个维度：

| 维度 | 问题 | 判断依据 |
|------|------|----------|
| 正确性 | 功能在它声称支持的模式下是否真的工作 | 路由/store 的调用约定、schema 与实际 SQL |
| 安全性 | 未授权访问与越权的门槛有多高 | 鉴权链、服务端口暴露面、RLS 是否生效 |
| 一致性 | 内存模式与 Postgres 模式是否行为一致 | 同步/异步、返回形状、持久化范围 |
| 可维护性 | 改一处会不会悄悄坏另一处 | 重复实现、死代码、文档漂移 |
| 可交付性 | 能否稳定部署与观测 | 镜像、健康探测、迁移、CI |

风险分四级：**P0** 影响数据正确性或安全边界；**P1** 功能失效但存在绕过路径或只在特定模式出现；**P2** 运维/体验问题；**P3** 代码卫生。

评估的证据来源：第 4–12 章的逐条核实、`AGENTS.md` 的项目记忆、以及本次写作时实际运行的 `pnpm run lint`（结果：`0 errors, 225 warnings`）。

### 13.2 架构上做得好的地方

在列问题之前，先说清楚哪些设计是这套代码真正的资产——它们决定了修复成本的可控性。

**1. 权限模型是单一且可对照的。** Organization → Marking → Role 三层判定集中在 `packages/permissions/src/authorizer.ts`，与 Foundry 的逐条对照写在 `docs/foundry-fidelity.md`，并且同一批用例在内存、共享文件、Postgres 三种存储上跑（`packages/permissions/tests/authorization.test.ts`）。这种"一份模型 + 一份对照文档 + 一组跨存储测试"的结构，比多数同类项目成熟。

**2. 错误信封统一且有脱敏设计。** 所有服务用同一个 `OpenFoundryApiError`，`parameters` 只收 SafeArg，敏感值经 `toJSON()` 被剔除，`toInternalJSON()` 才给日志（`packages/errors/src/api-error.ts:73-101`）。客户端可以按 `errorCode` 稳定分支。

**3. 存储抽象允许两种后端共存。** `ObjectReadWriteStore` 的方法返回 `T | Promise<T>`，路由按接口注册、不做类型强转（`services/svc-objects/src/server.ts:128-160`）。这让内存实现与 Pg 实现可以同时通过同一套路由测试——代价是"漏 `await`"成为一类系统性 bug（见 13.3）。

**4. 一条命令起全栈。** `bash start.sh` 在无 Docker、无 Postgres、无 Redis、无 LLM key 的机器上也能跑起完整演示。这是它作为"模拟器"的核心竞争力，也是很多设计取舍（内存优先、mock LLM、可选 Redis）的来源。

**5. 测试密度高、分层清晰。** 约 100 个测试文件，覆盖包内单测、跨服务集成、SDK 兼容、PGlite 真实数据库四层（第 12 章）。虽然门禁不完整，但"想验证时找得到测试"这件事本身降低了修改风险。

**6. 代码注释记录"为什么"。** 例如网关为什么把 `authPlugin` 直接挂在根实例（`services/svc-gateway/src/server.ts:168-180`）、为什么丢弃 `Content-Length`（`services/svc-gateway/src/proxy.ts:72-79`）、为什么 `ObjectStore` 用插入序而不是时间戳。这些注释让后来者不必重新踩坑。

**7. 契约层有完整工具链，只是没接上。** Conjure YAML → IR → 编译器 → 代码生成器全部实现且有集成测试（`tests/integration/conjure-pipeline.test.ts`）。第 4 章指出的"未接入构建"是接线问题，不是从零建设的问题。

### 13.3 风险登记册

#### P0：数据正确性与安全边界

| ID | 问题 | 影响 | 触发条件 | 证据 |
|----|------|------|----------|------|
| P0-1 | 认证默认关闭 | 任何能访问网关的客户端可读写全部资源 | `AUTH_PUBLIC_KEY` 为空（演示默认） | 第 5 章；`services/svc-gateway/src/middleware/auth.ts:91-115` |
| P0-2 | 服务端口信任 `x-user-*` 且监听 `0.0.0.0` | 绕过网关即可冒充任意用户/管理员 | 任一服务端口对外可达 | 第 5 章；各服务 `config.ts` 的 `HOST` 默认值 |
| P0-3 | 租户隔离端到端未生效 | 多租户部署下可能读到别的组织数据 | 设置 `DATABASE_URL` 但以超级用户连接，或 store 未传 `orgRid` | 第 5、8 章；`tests/integration/tenant-isolation.test.ts:4-9` |
| P0-4 | 动作执行非原子 | 副作用部分成功、部分失败，留下不一致状态 | 动作包含多次对象写入且中途失败 | 第 7 章；`services/svc-actions/src/objects-client.ts` |
| P0-5 | Postgres 模式下多条关键写路径失败 | 数据集上传 23503、admin 用户/组 500 或静默失效、actions v2 500 | 设置 `DATABASE_URL` | 第 7、8 章 |
| P0-6 | 迁移双轨漂移 | 先 Compose 初始化再 `pnpm db:migrate` 必然失败；两套 schema 都不是 `PgDatasetStore` 的完整超集 | 生产部署按顺序执行两条迁移路径 | 第 8 章；`scripts/migrate.sql` vs `db/migrations/*.sql` |

#### P1：功能失效与一致性

| ID | 问题 | 影响 | 证据 |
|----|------|------|------|
| P1-1 | 网关路径嗅探只放行 `apply` 与 `validate` | `applyBatch`、executions 经网关 404，只有直连 8083 可用 | 第 7 章；`services/svc-gateway/src/routes/v2/ontologies.ts:35` |
| P1-2 | 同步接口调用异步 store | 6 个服务在 PG 模式下 500 或静默失败 | 第 7 章 |
| P1-3 | 内存与 Pg store 返回形状不同 | svc-admin 即使补 `await` 也修不好（`StoredUser[]` vs `{items,total}`） | 第 7 章 |
| P1-4 | 部分持久化 | 重启后数据集/函数/AIP 会话清空，而种子不会重跑 | 第 3、8 章 |
| P1-5 | 限流按租户分桶失效 | 每个令牌一个桶，刷新即重置；无法按组织限流 | 第 5 章；`services/svc-gateway/src/rate-limiter.ts:273-290` |
| P1-6 | 未知过滤算子静默放行 | 拼错算子返回全部对象，客户端以为过滤生效 | 第 6 章 |
| P1-7 | `svc-objects` 不解析本体别名 | 用 apiName 访问对象接口返回空列表而非报错 | 第 6 章 |
| P1-8 | AIP 端点无权限钩子 | 在服务端口可达前提下可越权读写会话 | 第 8 章 |
| P1-9 | 函数执行无权限、无沙箱、超时对同步代码无效 | 任意已注册函数可被任意调用者执行 | 第 7 章 |
| P1-10 | 未启动的 3 个服务 | `/api/v2/compass/*`、`/webhooks*`、`/media*` 恒 502 | 第 2、7 章 |
| P1-11 | `@openfoundry/client` 的重试层顺序错误 | 429/503 在到达重试层前已被抛异常，`maxRetries` 形同虚设 | 第 4 章；`packages/client/src/client.ts:28` |

#### P2：运维与交付

| ID | 问题 | 影响 | 证据 |
|----|------|------|------|
| P2-1 | 健康探测路径错误 | Docker/Compose/Helm 探测 `/health`，服务只有 `/status/health` | 第 10 章 |
| P2-2 | `/metrics` 只在网关、且不豁免鉴权 | Prometheus 抓取 401；下游指标不可采集 | 第 5、11 章 |
| P2-3 | `start.sh` 端口管理缺口 | 不清理 8089/8090/8091，不检查 8091，强杀漏 8086 | 第 7、10 章 |
| P2-4 | 集成测试非幂等且依赖种子 | 第二次运行 409/计数膨胀；空库级联失败 | 第 12 章 |
| P2-5 | CI 与本地命令不一致 | CI 用根配置跑集成测试、只起 9 个服务；本地用 `tests/vitest.config.ts` | 第 12 章 |
| P2-6 | 发布流水线永不发布 | 无 `NPM_TOKEN`，`changeset publish` 必失败 | 第 10 章 |
| P2-7 | Helm/Compose 不闭环 | 镜像不由 CI 构建、ConfigMap 变量名与服务读取的不一致、console 端口错 | 第 10 章 |
| P2-8 | 健康检查不检查依赖 | readiness 恒为 HEALTHY，数据库断开仍 200 | 第 11 章 |

#### P3：代码卫生

| ID | 问题 | 影响 |
|----|------|------|
| P3-1 | 4 个库零消费者（`logging`、`tracing`、`redis`、`test-utils`） | 平台没有结构化日志/追踪约定；700 行测试辅助无人使用 |
| P3-2 | 死代码：compass 两套 resource store、`MarkingEvaluator`、`ui-tokens`、`PestControlDashboard`、`PgConversationStore`、`trigger-evaluator` | 读代码时误判能力边界 |
| P3-3 | 重复实现：分页 10 份、cron 2 份、pipeline 执行 2 份、v1 搜索独立 | 修一处漏一处 |
| P3-4 | 错误码目录不封闭 | 手写信封用了 `UNAUTHORIZED`/`RATE_LIMITED`/`BAD_GATEWAY` 等枚举外的值 |
| P3-5 | 11 个服务把 Fastify 4xx 变成 500 | 调用方错误被报成服务端错误 |
| P3-6 | 无覆盖率门禁、lint warning 不受控、prettier 不校验 | 质量只能靠人看 |

### 13.4 已确认不可用或受限的功能

以下每一条都在代码或文档中确认过，不是推测。标注「未验证」的是由代码路径推断、未实际运行。

| 功能 | 状态 | 原因/证据 |
|------|------|-----------|
| 动作删除对象 | 不存在 | `ObjectsClient` 只有 get/create/update；README 的 "delete" 声明有误 |
| 演示用的 3 个函数 | 执行即 `ReferenceError` | 函数体引用未注入的 `objects`；执行器只注入 `args`/`console` |
| `applyBatch` 经网关 | 404 | 网关正则带 `$` 锚点 |
| Compass/Webhooks/Media API | 502 | 服务不在 `start.sh` 名单 |
| 数据集 PG 上传 | 23503 | 路由把 `transactionRid` 硬编码为空串，违反外键 |
| 数据集 v2 路由（PG） | 500 或返回 `{}` | 未 `await` 异步 store |
| admin 用户/组路由（PG） | 500 或静默失效 | 未 `await`，且返回形状不同 |
| Sentinel（PG 模式） | 可能 60 秒内崩溃（未验证） | 调度器对 Promise 调 `.filter` |
| AIP embeddings（Anthropic） | 500 | 提供方不支持 embeddings |
| AIP 会话持久化 | 无 | `PgConversationStore` 是死代码 |
| Pipeline execute | 不写输出数据集 | 只返回数据，不建事务/分支 |
| connectivity | 18/20 | 两个 execute 操作未实现 |
| models API | 8/32 | 其余操作未实现 |
| v1 文件上传 / attachments | 404 | 有意不服务 |
| 租户隔离 | 无端到端证据 | 集成测试 `it.todo` |
| Network Graph 边 | 不显示（GUIDE） | 已知限制清单 |
| `/api/v2/search` 无 Postgres | 不可用 | 网关直连 PG 搜索索引 |
| 集成测试 Actions apply | 404 而非期望的 400 | 注册表与 ontology actionTypes 两份真相源 |

### 13.5 文档与实现不一致汇总

这些是本次写作中发现的、值得单独修的文档缺陷（代码行为本身见前文）：

| # | 文档声明 | 实际 | 位置 |
|---|----------|------|------|
| 1 | README：10 个后端服务、28 个 packages | 13 个服务、33 个库包、49 个 workspace 包 | `README.md` 项目结构段 |
| 2 | README：`POST /api/v2/auth/token`、`GET /api/v2/auth/me` | multipass 无这两条路由，实际 404 | `README.md:275-276` |
| 3 | README：动作副作用含 delete | 无删除能力 | `README.md:96` |
| 4 | README：3 个可执行 TypeScript 函数 | 种子函数引用未定义变量；无 TS 转译 | `README.md:103` |
| 5 | README：13 个算子含 `prefix`/`anyTerm` | 这两个是 v1 语法，v2 不接受 | `README.md:92` |
| 6 | AGENTS：`migrate.sql` 不含 `org_rid`/RLS | 实际含 40 处 `org_rid`、21 处 RLS；真正缺的是表 | `AGENTS.md:173`、`scripts/migrate.sql` |
| 7 | AGENTS：632 个 lint 文件 | 实测 644 个 | `eslint.config.mjs` 覆盖范围 |
| 8 | AGENTS：容器健康检查用 `/status/health` | 只有一处 Compose 用；Dockerfile 与 Helm 用不存在的 `/health` | `AGENTS.md:119` |
| 9 | AGENTS：admin 路由在 PG 下"500" | 只有列表类 500；单资源返回 `{}`、成员增删静默失效 | `AGENTS.md:167` |
| 10 | `.env.example`：`GATEWAY_PORT`/`ONTOLOGY_PORT`… | 服务只读 `PORT`，这些变量无读取者 | `.env.example:76-86` |
| 11 | `.env.example`：`VITE_API_BASE_URL` | 控制台读 `VITE_API_URL`，容器构建一律回退 localhost | `apps/app-console/src/config.ts:8` |
| 12 | `.env.example`：`TRACING_ENABLED`/`TRACING_ENDPOINT` | 无 span 导出实现 | `packages/tracing` |
| 13 | 网关 config 注释：支持 JWKS URL | 只接受 PEM，`AUTH_JWKS_URL` 无读取者 | `services/svc-gateway/src/config.ts:23-26` |
| 14 | deploy 注释：网关不断言 `x-user-*` | `proxy.ts` 已从 claims 写入五个头 | `deploy/.env.production:75-81` |
| 15 | app-docs：maker DSL 链式 API 示例 | 真实 API 是回调式，示例无法编译 | `apps/app-docs/src/pages/OntologyGuide.tsx:136-155` |
| 16 | AGENTS：v2 `OntologyObject` 是裸属性 map | v2 路由返回 `StoredObject`（嵌套 `properties`）；裸 map 只存在于 SDK 类型，造成 SDK 与 wire 漂移 | `AGENTS.md:86`、`services/svc-objects/src/routes/v2/objects.ts:366-371` |
| 17 | app-docs：SDK 使用 `FoundryProvider`/`@openfoundry/sdk/react`、`foundryPlatformFetch` 自动加 trace ID | 实际是 `OpenFoundryProvider`/`sdk-react`；`client` 不加 trace ID；示例路径 `/api/objects/v1/*` 不存在 | `apps/app-docs/src/pages/SdkGuide.tsx:89-332` |
| 18 | `conjure/*.yml` 与实现一致 | `dataset-service.yml` 的 Branch 声明 `datasetRid`（store 无此字段）、compass 的 base-path 拼不出真实路由、`ObjectTypeV2.primaryKey` 与实现的 `primaryKeyApiName` 不一致 | 第 4 章 |
| 19 | `wire/query-params.ts` 注释：数组序列化为逗号连接 | 实现是重复 key | `packages/wire/src/query-params.ts:10` |

> 这些不一致本身不改变运行时行为，但会系统性地误导新加入者。建议把它们作为"文档回归测试"的输入：修完一条删一条，并在 CI 里加一个检查 README 端点清单与实际路由的脚本。

### 13.6 技术债的量化

| 类别 | 规模 | 可观测信号 |
|------|------|-----------|
| 同步/异步错配 | 6 个服务、至少 20 处路由 | 只在 `DATABASE_URL` 模式下暴露，内存测试全绿 |
| 未接入授权的服务 | 13 个中的 8 个 | `grep -r requirePermission services/*/src` 只命中 5 个 |
| 零消费者的库 | 4 个（约 2,100 行） | `grep` 无 `package.json` 依赖 |
| 重复实现 | 分页 10 份、cron 2 份、pipeline 执行 2 份 | 行为漂移，修一处漏一处 |
| 未接线能力 | Conjure 生成链、嵌入钩子、health 依赖检查 | 代码完整但无调用方 |
| 测试不可知部分 | 覆盖率、租户隔离、PG 模式端到端 | 无覆盖率工具、`it.todo`、CI 不跑 PG |
| Lint 债务 | 225 warnings（181 个 `any`） | 本次实测；不阻塞 CI |

### 13.7 总体判断

| 维度 | 评分（1–5） | 说明 |
|------|-------------|------|
| 领域建模 | 4 | 本体、ObjectSet、权限模型的结构与 Foundry 对齐，文档齐全 |
| API 保真度 | 4 | v1/v2 显式分离、错误信封一致；但部分端点缺失或形状与 SDK 类型不符 |
| 可运行性 | 5 | 无外部依赖即可起全栈，演示体验极佳 |
| 生产就绪度 | 2 | 默认无认证、隔离未闭环、PG 模式多条路径失败 |
| 一致性 | 2 | 双存储模式语义差异大，同步/异步错配系统性存在 |
| 可维护性 | 3 | 注释优秀、结构同构；但重复实现与死代码多，契约链未接入 |
| 可观测性 | 2 | 有指标与请求 ID，但日志/追踪库零消费者、健康检查不查依赖 |
| 交付成熟度 | 2 | 镜像、探测、迁移、发布四处不闭环 |

一句话结论：**它是一套优秀的"可运行参考实现"，还不是一套可交付的产品。** 差距不在功能数量，而在"同一行为在两种存储模式下是否一致""部署后是否真的能工作"这两类问题上。

---

## 第 14 章 演进建议

### 14.1 原则

1. **先正确性，后能力。** 在 PG 模式下坏掉的路由比缺少的端点更伤——前者让"支持 Postgres"这句话不可信。
2. **一次只改一种存储模式，并用测试锁死。** 任何同步/异步的修改都必须在 `DATABASE_URL` 模式下跑一遍，而不是只跑内存测试。
3. **把"隐式契约"变成显式检查。** 网关路径嗅探、`x-user-*` 信任、迁移双轨，都是没有编译期/测试期保护的知识。
4. **不为模拟器引入生产级复杂度。** 消息队列、服务网格、分布式事务不在建议范围内；模拟器的价值是可读、可跑、可改。

### 14.2 P0：安全与数据正确性（建议 1–2 周）

| # | 动作 | 具体做法 | 验收标准 |
|---|------|----------|----------|
| 1 | 让认证默认安全 | `deploy/` 默认生成并注入 `AUTH_PUBLIC_KEY`/`JWT_PRIVATE_KEY`；空 key 时启动打显式告警并在 `/status/health` 暴露 `authDisabled: true` | 部署后未带令牌的请求 401；健康检查可见姿态 |
| 2 | 服务端口绑定内网 | 各服务 `HOST` 默认改 `127.0.0.1`（容器内用 `0.0.0.0` 由环境变量覆盖）；Compose 不映射服务端口 | 宿主机无法直连 8081–8092 |
| 3 | 统一 PG 模式调用约定 | 把所有 store 接口改成显式异步（或全部同步包装），逐路由补 `await`；先修 admin/users、actions v2、datasets v2、functions、webhooks、sentinel | `DATABASE_URL` 下跑完整路由冒烟测试全绿 |
| 4 | 对齐两种 store 的返回形状 | 让 `PgUserStore.listUsers()` 与 `UserStore.listUsers()` 返回同一类型（建议统一 `{items,total}`） | 同一测试用例在两种 store 上断言同一形状 |
| 5 | 修数据集 PG 写路径 | `transactionRid` 传真实事务；`PgDatasetStore` 在两套 schema 上跑 PGlite 测试 | 上传→commit→列表→下载全链路在 PGlite 通过 |
| 6 | 收敛迁移入口 | 让 `scripts/migrate.sql` 与 `db/migrations` 由同一份源生成，或在 CI 里对两套 schema 跑同一组 schema 断言 | 两条路径建出的库结构一致；`pnpm db:migrate` 在 Compose 库上幂等 |
| 7 | 租户隔离二选一 | 要么在 Pg store 构造时强制传 `orgRid` 并让服务从请求取 org；要么明确"单租户模式"并在文档与启动日志声明 | `tenant-isolation` 测试从 `it.todo` 变成真实断言 |

### 14.3 P1：一致性与路由（建议 2–4 周）

| # | 动作 | 具体做法 | 验收标准 |
|---|------|----------|----------|
| 8 | 网关路由表化 | 把 `/ontologies/*` 的正则嗅探改成显式路由表（路径模式 → 服务），启动时校验无重叠、无遗漏 | 新增端点只需加一行；`applyBatch` 可达 |
| 9 | 动作幂等与补偿 | 为 `apply` 引入执行日志 + 补偿/重试；至少让部分失败返回可诊断的错误而不是静默继续 | 注入失败后对象状态可回滚或明确标记 |
| 10 | 未知输入不再静默 | 过滤算子、ObjectSet 类型、属性名未识别时返回 400 `INVALID_ARGUMENT` | 拼错算子得到 400 |
| 11 | 别名统一解析 | svc-objects/svc-actions 也接受 ontology apiName 并先解析成 RID | 用 apiName 与 RID 访问结果一致 |
| 12 | 补授权缺口 | 为 aip、functions、compass、media、webhooks、sentinel 注册 `registerAuthorization` 并加 `requirePermission` | 未授权调用返回 403 |
| 13 | 健康与指标补齐 | 所有服务注册 `metricsPlugin`；readiness 检查数据库/下游；`/metrics` 用独立令牌或网络策略 | Prometheus 可抓取；断库时 readiness 失败 |
| 14 | 修 start.sh | 端口清单从配置派生；启动全部 13 个服务或明确 `--services` 选择；检查 8091 | 二次启动无残留进程；状态表覆盖全部服务 |

### 14.4 P2：交付与运维（建议 1–2 月）

| # | 动作 | 具体做法 |
|---|------|----------|
| 15 | 统一健康路径 | 服务统一 `/status/health`，Dockerfile/Compose/Helm 全部改过来 |
| 16 | 镜像与 Helm 闭环 | `docker.yml` 构建 console 与全部服务；Helm ConfigMap 用服务实际读取的变量名；console 端口与 nginx 对齐 |
| 17 | 发布可用 | 配置 `NPM_TOKEN` 或删掉 release 工作流；`changeset` 的 `baseBranch` 改 `master` |
| 18 | 测试可重复 | 集成测试自建数据并在结束时清理；`/tmp/openfoundry-data` 支持 `DATA_DIR` 覆盖；CI 用同一 `tests/vitest.config.ts` |
| 19 | 质量门禁 | 加 `@vitest/coverage-v8` 与最低阈值（先 40% 起步）；把 lint warning 冻结在基线并只允许下降 |
| 20 | 文档回归 | 用脚本比对 README 端点清单与 `conjure/*.yml`/路由注册，不一致即失败 |

### 14.5 P3：能力补齐（按需）

只有在前面几项稳定后再考虑，否则是在流沙上盖楼：

- **接通 Conjure 生成链**：让 `api-types` 由 `conjure/*.yml` 生成并入库，删掉手写镜像；在 CI 里检查生成物与 YAML 一致。
- **补齐 v1 端点**：文件上传与 attachments，或明确在 README 标注"不服务"。
- **AIP 真实化**：会话持久化、流式响应、按模型路由、embeddings 提供方对齐。
- **Pipeline 真执行**：写 `outputDataset`、建事务与分支，删掉第二份 `executeStepLocal`。
- **Sentinel 调度**：统一 cron 实现（UTC），副作用走真实存在的路由。
- **前端契约层**：让控制台改用 `@openfoundry/sdk` 或至少共享类型，消灭 `rid`/`id` 错配。

### 14.6 不建议做的事

| 提议 | 为什么不做 |
|------|-----------|
| 引入消息队列/事件总线 | 当前服务间只有 6 条同步调用；引入异步会掩盖同步/异步错配这类真问题，且与"本地可跑"目标冲突 |
| 拆更多微服务 | 13 个服务已经带来共享存储与鉴权边界成本；应先把现有服务的模式一致性做好 |
| 用 ORM 替换手写 SQL | 迁移与 RLS 是当前主要风险点，ORM 不会自动解决双 schema 漂移，反而遮蔽 SQL |
| 追求 100% Foundry 端点覆盖 | 保真度靠"形状正确 + 已实现端点可用"，而不是数量；未实现的 404 比错误的 200 更诚实 |
| 给内存模式加租户隔离 | 内存模式的产品定位是单机演示；应明确声明其边界，而不是伪造隔离 |

### 14.7 分阶段路线

```
第 1–2 周   P0-1/2/3/4/5/6：认证默认安全、端口收敛、PG 调用约定、迁移收敛、隔离决策
第 3–6 周   P1：网关路由表、动作补偿、未知输入 400、授权缺口、健康指标、start.sh
第 2–3 月   P2：健康路径、镜像/Helm、发布、测试可重复、覆盖率门禁、文档回归
之后        P3：Conjure 接线、v1 补齐、AIP/Pipeline 真实化、前端契约层
```

每个阶段结束时的判断标准只有一条：**在 `DATABASE_URL` 模式下，第 12 章列出的集成测试能重复跑通两遍。** 这一条同时覆盖了数据正确性、一致性、可重复性三个最关键的维度。

---

## 附录

### 附录 A 端口与进程清单

| 端口 | 服务 | 包名 | 默认存储（无 `DATABASE_URL`） | `start.sh` | Docker Compose | Helm |
|------|------|------|-------------------------------|------------|----------------|------|
| 8080 | svc-gateway | `@openfoundry/svc-gateway` | 无状态（自实现搜索需 PG） | 是 | 是 | 是 |
| 8081 | svc-ontology | `@openfoundry/svc-ontology` | JSON 文件 | 是 | 是 | 是 |
| 8082 | svc-objects | `@openfoundry/svc-objects` | JSON 文件 | 是 | 是 | 是 |
| 8083 | svc-actions | `@openfoundry/svc-actions` | 内存 | 是 | 是 | 是 |
| 8084 | svc-multipass | `@openfoundry/svc-multipass` | 内存（含密钥与令牌） | 是 | 是 | 是 |
| 8085 | svc-datasets | `@openfoundry/svc-datasets` | 内存 | 是 | 是 | 是 |
| 8086 | svc-compass | `@openfoundry/svc-compass` | 内存 | 否 | 是 | 是 |
| 8087 | svc-admin | `@openfoundry/svc-admin` | 内存 | 是 | 是 | 是 |
| 8088 | svc-functions | `@openfoundry/svc-functions` | 内存 | 是 | 是 | 是 |
| 8089 | svc-webhooks | `@openfoundry/svc-webhooks` | 内存 | 否 | 是 | 是 |
| 8090 | svc-media | `@openfoundry/svc-media` | 内存 / S3 | 否 | 是 | 是 |
| 8091 | svc-sentinel | `@openfoundry/svc-sentinel` | 内存 | 是 | 是 | 是 |
| 8092 | svc-aip | `@openfoundry/svc-aip` | 内存 | 是 | 是 | 否（values 未列出） |
| 3000 | app-console | `@openfoundry/app-console` | —（Vite dev server） | 是 | 是 | 是 |

### 附录 B workspace 包清单（49 个）

**领域与契约库（11）**

| 包 | 职责 |
|----|------|
| `@openfoundry/ontology-schema` | 本体类型系统与校验 |
| `@openfoundry/ontology-ir` | 本体中间表示 |
| `@openfoundry/ontology-generator` | 从 IR 生成 TypeScript |
| `@openfoundry/ontology-maker` | 用 TypeScript DSL 定义本体 |
| `@openfoundry/object-set` | ObjectSet 类型、过滤器、求值引擎、聚合 |
| `@openfoundry/search` | 跨实体搜索索引与查询 |
| `@openfoundry/pipeline` | 管道 DAG 模型与执行 |
| `@openfoundry/rid` | 资源标识符解析与生成 |
| `@openfoundry/pagination` | 分页令牌与迭代器 |
| `@openfoundry/errors` | 结构化错误信封与错误码 |
| `@openfoundry/wire` | Conjure 线上格式序列化约定 |

**契约工具链与 SDK（7）**

| 包 | 职责 |
|----|------|
| `@openfoundry/conjure-ir` | Conjure IR 的 TypeScript 模型 |
| `@openfoundry/conjure-compiler` | YAML → IR 编译器 |
| `@openfoundry/conjure-codegen` | IR → TypeScript 代码生成器 |
| `@openfoundry/api-types` | 手写的 API 类型与端点元组（无消费者） |
| `@openfoundry/client` | 分层 fetch 客户端 |
| `@openfoundry/sdk` | 公共 SDK 门面（`createFoundryClient`） |
| `@openfoundry/sdk-oauth` / `sdk-react` / `sdk-cli` | OAuth 助手 / React hooks / `openfoundry` CLI |

**平台能力库（10）**

| 包 | 职责 |
|----|------|
| `@openfoundry/permissions` | 权限模型、授权存储、路由守卫 |
| `@openfoundry/markings` | Marking 类型与评估器（未被服务引用） |
| `@openfoundry/auth-tokens` | JWT 签发/校验、claims、scope |
| `@openfoundry/db` | 连接池、查询构建、迁移执行、审计、通用 PgStore |
| `@openfoundry/config` | 环境变量装载（与服务实际配置脱节） |
| `@openfoundry/redis` | ioredis 客户端（零消费者） |
| `@openfoundry/health` | 健康检查与 Prometheus 指标插件 |
| `@openfoundry/logging` | 结构化日志与 SafeArg（零消费者） |
| `@openfoundry/tracing` | B3 追踪头（零消费者） |
| `@openfoundry/test-utils` | 测试 fixture（零消费者） |

**前端库（3）**：`@openfoundry/ui-components`（Blueprint 封装）、`ui-icons`、`ui-tokens`（死代码）。

**服务（13）**：svc-gateway、svc-ontology、svc-objects、svc-actions、svc-multipass、svc-datasets、svc-compass、svc-admin、svc-functions、svc-webhooks、svc-media、svc-sentinel、svc-aip。

**应用（3）**：app-console、app-workshop、app-docs。

### 附录 C 关键环境变量

| 变量 | 作用 | 默认 | 读取者 |
|------|------|------|--------|
| `DATABASE_URL` | 切换到 Postgres | 空（内存模式） | 全部服务 |
| `PORT` / `HOST` | 服务监听地址 | 各服务固定端口 / `0.0.0.0` | 全部服务 |
| `AUTH_PUBLIC_KEY` | 网关验签公钥（PEM） | 空（放行） | svc-gateway |
| `AUTH_ISSUER` | 期望的 `iss` | `openfoundry-multipass` | svc-gateway |
| `AUTH_AUDIENCE` | 期望的 `aud` | `openfoundry-api` | svc-gateway |
| `JWT_PRIVATE_KEY` / `JWT_PUBLIC_KEY` | multipass 签发密钥对 | 开发态自动生成 | svc-multipass |
| `OAUTH_CLIENTS` | 声明式客户端注册表 | 空 | svc-multipass |
| `ENFORCE_PERMISSIONS` | 是否强制资源级检查 | `false` | 各服务 |
| `RATE_LIMIT_ENABLED` / `_MAX_REQUESTS` / `_WINDOW_MS` | 网关限流 | `true` / 100 / 60s | svc-gateway |
| `REDIS_URL` | 限流/缓存后端 | 空（进程内） | svc-gateway、redis 包 |
| `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GLM_API_KEY` / `OLLAMA_*` | LLM 提供方 | 空（mock） | svc-aip |
| `S3_ENDPOINT` / `S3_ACCESS_KEY` / `S3_SECRET_KEY` / `S3_BUCKET` | 媒体存储 | 空（内存） | svc-media |
| `*_SERVICE_URL` | 服务间调用地址 | `http://localhost:<port>` | 网关与调用方 |
| `OPENFOUNDRY_TOKEN` / `OPENFOUNDRY_CLIENT_ID` / `OPENFOUNDRY_CLIENT_SECRET` | 脚本调用凭据 | 空（不带认证） | `scripts/lib/auth.*` |
| `VITE_API_URL` | 控制台网关地址 | `http://localhost:8080` | app-console |
| `NODE_ENV` | 生产安全阀 | 未设置 | svc-gateway |

> 注意：`.env.example` 中的 `GATEWAY_PORT`、`ONTOLOGY_PORT`、`VITE_API_BASE_URL`、`LOG_FORMAT`、`JWT_SECRET`、`TRACING_*` 均无读取者（见 13.5 节第 10–12 条）。

### 附录 D 端点概览

按服务统计的路由声明数量（`app.get/post/put/patch/delete/all` 计数，含 v1 与内部路由）：

| 服务 | 路由数 | 主要前缀 |
|------|--------|----------|
| svc-datasets | 58 | `/api/v2/datasets`、`/api/v2/connectivity`、`/api/v2/pipelines`、`/api/v1/datasets` |
| svc-gateway | 46 | `/api/v2/*`、`/api/v1/*`、`/multipass/*`、`/status/*`、`/metrics` |
| svc-ontology | 41 | `/api/v2/ontologies`、`/api/v1/ontologies` |
| svc-admin | 29 | `/api/v2/admin/*`、`/api/v2/resources/:rid/roles` |
| svc-objects | 23 | `/api/v2/ontologies/:rid/objects`、`objectSets`、`links`、`/api/v1/...` |
| svc-compass | 21 | `/api/v2/compass/*` |
| svc-aip | 20 | `/api/v2/aip/*`、`/api/v2/models/*` |
| svc-sentinel | 13 | `/api/v2/monitors`、`/api/v2/schedules`、`/api/v2/effects` |
| svc-multipass | 13 | `/multipass/api/*`、`/api/v2/auth/*` |
| svc-actions | 11 | `/api/v2/ontologies/:rid/actions`、`/api/v1/...` |
| svc-webhooks | 11 | `/api/v2/webhooks*`、`/api/v2/events` |
| svc-functions | 9 | `/api/v2/functions` |
| svc-media | 9 | `/api/v2/media` |

精确的端点与模型以 `conjure/*.yml`（v2）与各服务 `src/routes/`（含 v1）为准；两者的覆盖差异见第 4 章。

### 附录 E 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| 本体 | Ontology | 对象类型、链接类型、动作类型的集合，是平台的建模层 |
| 对象类型 | Object Type | 带属性 schema 的实体类型 |
| 对象 | Object | 对象类型的一个实例，以主键标识 |
| 链接类型 | Link Type | 两个对象类型之间的有向关系 |
| 动作类型 | Action Type | 对对象做受控写操作的定义 |
| 对象集 | ObjectSet | 对一组对象的声明式描述，支持过滤、集合运算、searchAround |
| 主键 | Primary Key | 对象在其类型内的唯一标识 |
| RID | Resource Identifier | `ri.<service>.<instance>.<type>.<id>` 形式的全局标识 |
| Marking | Marking | 强制访问控制标签，必须全部持有才能访问 |
| 角色授予 | Grant | 资源上给某个主体（用户/组）的角色 |
| 策略 | Policy | 资源上的 Organization 与 Marking 约束 |
| 数据集 | Dataset | 带分支与事务的文件集合 |
| 分支 | Branch | 数据集的一个可写版本线 |
| 事务 | Transaction | 数据集上的一次写入会话（开→写→提交/中止） |
| 租户 | Tenant | 以 `org_rid` 标识的组织，由 RLS 隔离 |
| 契约 | Conjure | Palantir 的 IDL，用于声明服务、端点与类型 |
| 线上格式 | Wire format | JSON 序列化约定（枚举、联合、可选值等） |
