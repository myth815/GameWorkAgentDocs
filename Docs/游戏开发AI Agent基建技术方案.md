# 游戏开发 GameWorkAgent 基建技术方案

## 0. 文档体系与上下文传递

为了保证整个工程在不同阶段和不同参与方之间的信息一致性和上下文传递，本项目采用分层文档体系，每一层文档都有明确的角色和边界：

### 0.1 文档层级

-   **L1：全局/系统级文档** -- 描述整个 GameWorkAgent
    系统的愿景、架构、技术选型和治理原则，是所有后续文档的唯一真相源。本文即属于
    L1 层。
-   **L2：模块级文档** --
    面向子系统或组件（例如工作流编排、权限系统、前端管理台、CLI
    工具）的实现说明。在 L1
    的框架下，明确模块职责、接口、任务拆分和验收口径。
-   **L3：任务级文档** -- 按 L2
    拆解形成的单个任务说明，包含设计上下文、测试要求、交付清单和验收门禁，用于指导
    AI 或开发团队具体实施。
-   **L4：执行级文档** -- 从 L3
    派生出的可执行工单，提供具体操作步骤、输入输出约定、回滚策略和报告产物。执行过程中的结果反馈反哺
    L3/L2，形成闭环。

### 0.2 上下文传递与验收口径

-   **上下文传递**：信息自上而下细化（L1→L2→L3→L4），反馈自下而上汇聚，确保设计意图和业务背景不会在流程中丢失。
-   **AI 驱动验收**：由于大量开发、测试和部署工作由 AI
    助手完成，本项目采用自动化门禁代替人工先期签字。验收标准包括单元/集成/契约/性能/E2E
    测试、静态检查、覆盖率、指标阈值和 AI
    评分，结果由系统自动判定通过或回退。人类运维人员仅在关键阶段审核和干预异常情况。

## 1. 项目定位与愿景

### 1.1 核心理念

    我们不是要让 AI 做创造性工作，而是要自动化重复性劳动
    我们不是要替代人，而是要把人从机械劳动中解放出来
    我们不是要做通用 AI，而是要把游戏开发经验固化成可复用的流程
    我们要让 AI 成为每个开发者的助手，让不懂技术的人也能开发微服务

### 1.2 系统定位

**GameWorkAgent 是一个 AI 驱动的大型任务编排系统**，核心特点： -
将游戏开发经验固化为可执行的 Agent - **原生集成 Claude Code**，AI
辅助微服务开发 - 端到端的 CLI 工具链，一键完成开发到部署 -
可视化工作流编辑器，拖拽式编排 - 服务于游戏开发全流程的自动化

### 1.3 技术愿景

    开发者体验:
      - "我不懂Docker，但我能通过 CLI 一键部署"
      - "我不会写 gRPC，但 Claude 帮我生成了接口"
      - "我只会 Python，但能开发出生产级微服务"
      - "我是美术，但我能组合已有服务解决问题"

## 2. 整体技术架构

### 2.1 技术选型总览

  -------------------------------------------------------------------------------------------------------------------------------------
  类别               选型                             为什么选它                               服务的用户需求
  ------------------ -------------------------------- ---------------------------------------- ----------------------------------------
  **开发语言**       **TypeScript/NodeJS（平台与 CLI  平台统一 TS 降低门槛；微服务以 **gRPC**  既统一体验，又不束缚领域团队的语言选择
                     统一）** +                       为契约实现多语言自由                     
                     **多语言微服务（Python/C++/C#                                             
                     等）**                                                                    

  **后端框架**       NestJS                           企业级架构，模块化清晰                   提供标准化的开发模式

  **前端框架**       **Vue 3 + Vite**                 学习曲线平缓，游戏开发者友好             降低前端开发门槛 50%

  **UI 框架**        **Arco Design Vue**              现代科技风，类似 Logto 风格              美观不死板的界面体验

  **工作流可视化**   **Vue Flow**                     Vue 生态的节点编排                       类 UE 蓝图编辑体验

  **工作流引擎**     Temporal                         支持长任务与故障恢复                     游戏构建数小时级

  **服务通信**       gRPC                             跨语言，高性能                           支持 C++/C# 游戏引擎集成

  **用户认证**       Logto                            TypeScript 原生，现代化                  统一身份认证，支持 OAuth

  **功能权限**       CASL                             声明式权限，前后端共享                   运营/门户侧快速判定

  **资源权限**       SpiceDB                          Zanzibar 模型                            DB/桶/Topic 等资源级权限

  **数据库**         PostgreSQL                       功能丰富，一库多用                       减少运维复杂度

  **对象存储**       MinIO                            S3 兼容，自托管                          存储大型游戏资产

  **日志系统**       **Grafana Loki**                 轻量级，成本低                           简化运维，满足追踪需求

  **分布式追踪**     **OpenTelemetry + Tempo**        行业标准，自动传播                       实现 A.B.C 路径追踪

  **向量数据库**     Milvus                           强大的向量搜索                           **存放代码/文档嵌入以支持长上下文 AI
                                                                                               开发**

  **监控系统**       Prometheus + Grafana             行业标准                                 统一监控面板

  **性能测试**       **K6**                           JS/TS 编写，易集成                       技术栈统一，易于使用

  **API 文档**       **Swagger/OpenAPI**              自动生成文档                             提升开发体验

  **容器化**         Docker +                         本地便捷 + 生产一致                      统一交付形态
                     **Kubernetes（生产统一）**                                                

  **边缘入口**       **Nginx（TLS/静态/基础路由）**   生态成熟、可细粒度控制，与团队经验匹配   统一域名/证书、前端静态托管
  -------------------------------------------------------------------------------------------------------------------------------------

> **负载均衡与弹性伸缩的单一真相源 = Kubernetes**。Nginx
> 作为边缘入口（TLS/前端/基础路由），业务流量由 Nginx 转发至 **K8s
> Ingress/Service**；**健康探针、滚动升级、HPA、金丝雀** 等由 K8s 负责。

### 2.2 架构分层设计

    ┌─────────────────────────────────────────────────────┐
    │                    用户界面层                         │
    │  ┌──────────┬───────────┬───────────┬──────────┐   │
    │  │工作流编辑器│ 任务管理  │ 微服务市场 │ 监控面板 │   │
    │  │(Vue Flow) │  (Vue 3)  │  (Vue 3)  │ (Grafana)│   │
    │  └──────────┴───────────┴───────────┴──────────┘   │
    └─────────────────────────────────────────────────────┘
                              ↓
    ┌─────────────────────────────────────────────────────┐
    │                 应用服务层 (NestJS)                   │
    │  ┌──────────┬───────────┬───────────┬──────────┐   │
    │  │ 用户服务  │ 工作流服务 │ 微服务管理 │ 资源管理 │   │
    │  │ (Logto)  │(Temporal) │  (gRPC)   │(SpiceDB) │   │
    │  └──────────┴───────────┴───────────┴──────────┘   │
    └─────────────────────────────────────────────────────┘
                              ↓
    ┌─────────────────────────────────────────────────────┐
    │       调度中心 (Temporal + Provider/运行态管理)        │
    │   编排/调度 + 故障恢复 + 服务声明/注册 + 资源对接        │
    └─────────────────────────────────────────────────────┘
                              ↓
    ┌─────────────────────────────────────────────────────┐
    │                   微服务层                           │
    │  ┌──────┬──────┬──────┬──────┬──────┬─────────┐   │
    │  │ LLM  │ 编译 │ 测试 │ 部署 │ 分析 │ 自定义  │   │
    │  │服务  │ 服务 │ 服务 │ 服务 │ 服务 │ 服务... │   │
    │  └──────┴──────┴──────┴──────┴──────┴─────────┘   │
    └─────────────────────────────────────────────────────┘
                              ↓
    ┌─────────────────────────────────────────────────────┐
    │                    基础设施层                         │
    │  ┌──────────┬───────────┬──────────┬───────────┐   │
    │  │PostgreSQL│   MinIO    │  Loki    │  Milvus   │   │
    │  └──────────┴───────────┴──────────┴───────────┘   │
    └─────────────────────────────────────────────────────┘

### 2.3 端到端流程：一键开发 → 服务声明 → 调度中心编排 → 按需启动 → 暴露入口

**目标：**让大部分微服务开发者通过 **单一指令**
完成本地环境、示例工程、运行调试、服务声明、打包与上线全链路。

**开发与声明阶段** 1. **一键本地环境**

    $ gwa setup         # 安装与检测本地依赖（Docker/K8s-Local/Proto/语言 Runtime）
    $ gwa init          # 选择语言模板（Python/TS/C++/C#）、示例工程、测试框架
    $ gwa dev           # 本地一键运行（含 .env 模板、示例数据、Mock 依赖）

\- AI
辅助：`gwa ai ``"为服务生成`` gRPC ``接口与`` Dockerfile"`，自动补齐
proto、服务骨架、测试用例。 -
内置"上下文包"：项目结构、规范、常见模板、提示词与 Claude Agent
预设，降低上手成本。

1.  **服务声明（Service Declaration）**

2.  服务 **未启动前**，通过 `service.yaml` 将
    **接口契约、资源需求、proto、公开 class** 等声明到 **调度中心**，供
    **编排器与可视化工作流** 引用与校验。

3.  **可视化编排先行**

4.  工作流编辑器（Vue Flow）即时 **引用已声明的服务节点**（节点内含
    Temporal Activity 元数据），即使服务尚未启动，也可
    **完成流程编排与校验**。

**发布与运行阶段** 4. **构建与发布**

    $ gwa build         # 构建镜像（多语言模板、多阶段 Dockerfile）
    $ gwa push          # 推送至公司私有镜像源
    $ gwa deploy        # 提交部署意图；调度中心按需拉起实例（K8s/VM/Custom）

1.  **按需启动（On-Demand）**

2.  运行时若检测到 **无可用实例**，调度中心依据 `runtime.kind`
    **启动实例**（默认 K8s；也可 VM/Custom Provider）。

3.  实例启动后进行 **运行态注册**（健康/容量/权重）并加入实时路由表。

4.  **统一暴露入口**

5.  **对外统一入口**：**Nginx（TLS/静态/基础路由） → K8s
    Ingress/Service**；

6.  **路由与弹性**：调度中心创建/更新 **K8s Ingress/Gateway**
    规则，对应服务上线/扩缩容；Nginx 配置保持稳定，无需运行期频繁改动。

7.  **资源生命周期**

8.  **Ephemeral 资源**：流程期/运行期动态分配，`ttl/retention`
    到期后自动销毁；**不占用固定命名路径**。

9.  **Persistent 资源**：**先申明先得**，后续申请需审批；支持离职/变更的
    **Owner Transfer** 与管理员强制迁移。

**service.yaml（示例）**

    name: image-processor
    version: 1.0.0
    interface:
      grpc: proto/image_processor.proto
      health: /health
      metrics: /metrics

    resources:
      - type: postgres
        name: game_db
        access: readwrite
        lifecycle: persistent           # persistent | ephemeral
        owner: project:alpha            # 所有者主体（用户/组/项目）
        approval_required: true         # 持久资源后续申请是否触发审批
        transfer_policy: on_approval    # on_approval | admin_only
      - type: minio
        bucket: assets-temp
        access: write
        lifecycle: ephemeral            # 临时资源：不分配固定路径
        ttl: "72h"                      # 过期自动回收
        retention: "24h"                # 完成后保留期
        destroy_on_complete: true

    runtime:
      kind: k8s                         # k8s | vm | custom
      image: registry/org/image-processor:1.0.0
      replicas: 2
      envFrom: secret/image-processor
      # vm/custom 可附 provider-specific 参数（如模版名、区域、GPU 规格等）

    classes:
      - ResizeTask
      - FormatConvertTask

### 2.4 调度中心的职责

-   **编排调度**：基于 Temporal 的 Workflow/Activity
    调用、重试、补偿、超时与幂等。
-   **声明注册**：管理服务声明、版本、资源需求；校验 proto
    与健康/指标约束。
-   **运行态注册**：服务实例上线/下线、健康状态、容量、权重，形成
    **实时路由表**。
-   **Ingress/Gateway 对接**：创建/更新 **K8s Ingress/Gateway**
    路由，维系对外统一入口；Nginx 仅路由至集群入口。
-   **资源对接**：依据 SpiceDB
    权限模型，为实例挂载/签发资源凭据（DB/桶/队列等）。
-   **DSL/SDK 转换**：将工作流图在 **运行时** 翻译为 Temporal
    Workflow/Activity（SDK/DSL）。
-   **Provider 抽象**：`k8s`（默认）、`vm`（Windows/DX12/UE
    特殊任务）、`custom`（渲染农场/专用硬件）。

## 3. 用户画像与技术需求

### 3.1 平台开发者（核心团队）

#### 3.1.1 角色画像

    团队规模: 3-5 人
    技术背景:
      - 精通 TypeScript/NodeJS
      - 熟悉游戏开发流程
      - 了解 DevOps 和云原生

    工作内容:
      - 搭建和维护基础平台
      - 开发核心微服务
      - 制定技术标准和规范
      - 优化系统性能

#### 3.1.2 技术栈使用

    日常使用的技术:
      开发:
        - NestJS 开发 API 服务
        - Vue 3 开发管理界面
        - Temporal 编写工作流
        - gRPC 定义服务接口
      部署运维:
        - Docker 容器化
        - PostgreSQL 数据管理
        - Grafana Loki 日志分析
        - Prometheus + Grafana 监控
      权限管理:
        - Logto 管理用户认证
        - CASL 定义功能权限
        - SpiceDB 管理资源权限

### 3.2 Agent 开发者（微服务开发者）

#### 3.2.1 角色画像

    团队规模: 20-30 人
    背景差异很大:
      - 游戏程序员: 精通 C++/C#，不熟悉 Web
      - 技术美术: 会 Python 脚本，不懂后端
      - 工具程序员: 会写工具，不懂微服务
      - AI 工程师: 懂算法，不熟悉工程化
    核心诉求:
      - "我想把我的 Python 脚本变成服务"
      - "我不想学 Docker 和 K8s"
      - "我希望有模板可以参考"
      - "调试要方便，部署要简单"

#### 3.2.2 AI 辅助开发体验（CLI + Claude Code）

    场景 1 - 不会写 gRPC 接口:
      开发者: "我有个 Python 函数 process_image"
      Claude: "我来帮你生成 gRPC 接口定义"
      生成: .proto、服务端代码、客户端示例
    场景 2 - 不懂 Docker:
      开发者: "我的服务需要 OpenCV"
      Claude: "我来生成 Dockerfile"
      生成: 多阶段构建 Dockerfile、requirements.txt、docker-compose.yml
    场景 3 - 不会写测试:
      Claude 生成单测/集成/性能（K6）用例

    CLI 体验:
      $ gwa setup
      $ gwa init
      $ gwa dev
      $ gwa ai "添加图像缩放功能"
      $ gwa declare
      $ gwa build && gwa push && gwa deploy

### 3.3 最终用户

#### 3.3.1 策划的使用场景

    需求: 策划案自动实现
    技术支撑:
      - LLM 服务: Claude/GPT 解析策划文档
      - 代码生成: 基于模板生成功能代码
      - 配置生成: 自动生成配置表
    工作流程:
      1. 上传策划文档（Word/Excel）
      2. AI 解析需求，生成任务列表
      3. 调用代码生成服务
      4. 调用配置生成服务
      5. 自动创建测试用例
      6. 通知开发人员 review

#### 3.3.2 美术的使用场景

    需求: 批量处理游戏资产
    技术支撑:
      - MinIO: 存储原始资产和处理结果
      - GPU 服务: 加速图像处理
      - Temporal: 管理批量任务
    典型工作流:
      贴图批量优化:
        1. 选择 1000 张贴图
        2. 配置压缩参数
        3. 选择目标平台(iOS/Android/PC)
        4. 提交处理任务
        5. Temporal 分配到多个 Worker 并行处理
        6. 处理完成后自动打包
        7. 通过 MinIO 下载链接获取结果

#### 3.3.3 程序员的使用场景

    需求: 崩溃自动分析
    技术支撑:
      - Grafana Loki: 存储和搜索崩溃日志
      - LLM 服务: 分析崩溃原因
      - Milvus: 相似崩溃聚类
    处理流程:
      1. 游戏客户端崩溃
      2. 上传 dump 文件到 MinIO
      3. 解析服务提取调用栈
      4. Loki 存储结构化崩溃信息
      5. LLM 分析可能原因
      6. Milvus 查找相似历史崩溃
      7. 自动创建 JIRA 单
      8. 通知相关开发者

## 4. 前端技术架构（Vue 3）

### 4.1 技术栈详细

    核心框架:
      Vue 3:
        - Composition API（渐进式采用）
        - TypeScript 完整支持
        - 更适合游戏开发者理解
    构建工具:
      Vite:
        - 秒级启动
        - HMR 热更新
        - 原生 ESM
    路由管理:
      Vue Router 4:
        - 类型安全/动态路由/路由守卫
    状态管理:
      Pinia:
        - 简洁/TS 友好/有 DevTools

### 4.2 UI 设计系统

    UI 框架:
      Arco Design Vue:
        - 现代科技感，深色模式，动画流畅，组件丰富
    自定义主题:
      色彩:
        primary: '#5E72E4'
        success: '#2DCE89'
        warning: '#FB6340'
        info:    '#11CDEF'
        dark:    '#172B4D'
    设计原则:
      - 卡片化、毛玻璃、微动画、响应式
    图标系统:
      - Iconify + 自定义游戏图标

### 4.3 工作流编辑器（Vue Flow）

    核心功能:
      - 节点拖拽/布局/缩放/小地图
    自定义节点（游戏特色）:
      - 触发/处理/判断/并行
    样式:
      - UE 蓝图风格，状态色（运行/成功/失败），端口类型标识
    可视化→运行时:
      - 节点内置 Temporal Activity 元数据
      - 由调度中心在运行时翻译为 Temporal Workflow/Activity 调用
    微服务市场:
      - 展示可复用节点（服务声明而来，含接口签名/Schema/示例）
      - 支持搜索/收藏/版本切换，"拿来即拖"

### 4.4 项目结构

    src/
    ├── assets/
    │   ├── styles/
    │   └── icons/
    ├── components/
    │   ├── common/         # AppButton/AppCard/AppModal
    │   ├── workflow/       # NodePanel/NodeEditor/FlowCanvas
    │   └── layout/         # AppHeader/AppSidebar/AppFooter
    ├── composables/        # useAuth/usePermission/useWorkflow
    ├── pages/              # dashboard/workflows/services/settings
    ├── stores/             # auth/project/workflow
    ├── api/                # client.ts + services/*
    ├── router/             # 路由
    ├── types/              # 类型定义
    └── utils/              # 工具函数

## 5. 认证与权限体系

### 5.1 用户认证（Logto）

    选择理由:
      - TS 实现/文档清晰/可自托管
    集成方式:
      - 不暴露管理界面，统一 Portal 与其交互
    认证流程:
      1. 用户访问平台 → 重定向统一登录
      2. 企业微信/GitLab OAuth → Logto 验证
      3. 返回 JWT → 前端存储 → 后续请求携带

### 5.2 权限模型（CASL + SpiceDB）

    边界与分工:
      CASL（运营/门户侧）:
        - 组织/项目/角色矩阵（超级/域/项目/成员）
        - 前后端本地判定，毫秒级
      SpiceDB（资源/开发安全）:
        - DB/桶/队列/Topic 等资源的关系授权与继承
        - 以关系模型表达精细策略
      API 密钥（预授权）:
        - CI/CD、客户端上报、定时任务；审批时绑定权限，运行时快速校验
    示例 Schema 片段:
      definition project {
        relation owner: user
        relation member: user
        permission manage = owner
        permission access = owner + member
      }

### 5.3 组织与用户背景

    治理模型:
      超级管理员（平台运营）: 域/租户/策略/镜像仓/证书/资源池
      域管理员: 项目/资源额度/审批流
      项目管理员/成员: CASL 角色纳管 + SpiceDB 资源授权
    离职与权限迁移:
      - 支持 Owner Transfer；紧急情况由超管/域管强制迁移（审计留痕）

## 6. 日志与监控系统

### 6.1 日志系统（Grafana Loki）

    选择理由:
      - 相比 ELK，资源消耗更低，维护简单
    取舍说明:
      - Loki 仅索引"标签"而非全文；复杂全文检索能力较弱
      - 通过良好"标签设计" + 关键词过滤满足排障
    标签设计（示例）:
      env: prod
      project: 王者荣耀
      workflow: daily-build
      node: compile-android
      level: error
      traceId: wf-123-456
    查询示例:
      {workflow="daily-build", level="error"}
      {project="王者荣耀"} |= "OutOfMemory"
      sum(rate({workflow="daily-build"}[5m])) by (node)

### 6.2 分布式追踪（OpenTelemetry + Tempo）

    概念:
      TraceID/SpanID/Path(A.B.C)
    实现:
      @Trace() 自动注入上下文；日志打点带 traceId/path
    存储:
      Tempo（对象存储/低成本，Grafana 原生集成）

### 6.3 监控系统（Prometheus + Grafana）

    系统指标: CPU/内存/磁盘/网络/容器资源
    业务指标: 执行次数/成功率/耗时分布/并发任务数
    自定义指标: 调用次数/API RT/队列长度/错误率
    仪表板: 系统总览/工作流监控/微服务状态/日志查询/告警中心

## 7. 测试体系

### 7.1 API 文档（Swagger）

    @ApiTags('workflows')
    @ApiOperation({ summary: '创建工作流' })
    class WorkflowController {
      @Post()
      @ApiResponse({ status: 201, type: WorkflowDto })
      create(@Body() dto: CreateWorkflowDto) {}
    }

### 7.2 测试工具链

    单元测试: Jest（覆盖率 >80%）
    集成测试: Supertest + Testcontainers
    契约测试: Pact（微服务兼容性）
    性能测试: K6（JS/TS）
    E2E: Playwright（UI 自动化）

### 7.3 性能测试（K6 示例）

    import http from 'k6/http'
    import { check } from 'k6'
    export const options = {
      stages: [
        { duration: '2m', target: 100 },
        { duration: '5m', target: 100 },
        { duration: '2m', target: 0 },
      ],
      thresholds: {
        http_req_duration: ['p(95)<500'],
        http_req_failed: ['rate<0.1'],
      }
    }
    export default function() {
      const res = http.post('http://api/workflows', {
        name: 'test-workflow',
        nodes: []
      })
      check(res, {
        'status is 200': (r) => r.status === 200,
        'workflow created': (r) => r.json('id') !== null
      })
    }

## 8. 微服务生态

### 8.1 平台核心服务

    LLM 服务: Claude/GPT/本地 Llama（负载按能力/成本/质量）
    编译服务: UE/Unity/分布式编译（增量、缓存）
    存储服务: MinIO（分片/断点续传/S3 兼容）
    通知服务: 企业微信/邮件/WebSocket

### 8.2 微服务开发规范

    接口与协议:
      - CPU 密集: gRPC；IO 密集: HTTP/JSON；流式: gRPC Stream
      - 契约: 以 proto 为单一真相源（多语言生成）
    必需接口:
      - /health /metrics /info
    资源声明:
      - 通过 service.yaml 明确资源需求（DB/MinIO/Topic 等）
      - 访问由 SpiceDB 审核/签发；默认绑定项目既有资源，新建需审批
    语言与模板:
      - Python/C++/C# 等模板由 CLI 提供骨架/Dockerfile/示例客户端
    运行 Provider:
      - kind: k8s（默认）/ vm（Windows DX12/UE）/ custom（渲染农场/专用硬件）

## 9. 开发工具链

### 9.1 CLI 工具

    核心命令:
      gwa setup          # 一键安装/检测开发环境
      gwa init           # 初始化项目（多语言模板）
      gwa dev            # 本地开发与调试
      gwa declare        # 依据 service.yaml 声明服务
      gwa test           # 运行测试
      gwa build          # 构建镜像
      gwa push           # 推送镜像
      gwa deploy         # 提交部署意图（调度中心/Provider 拉起）
      gwa logs           # 查看日志
      gwa ai <prompt>    # AI 辅助（接口、Dockerfile、测试、诊断等）
      gwa resource approve/transfer  # 资源审批/所有权迁移

\- **透明化 K8s**：本地/PoC 用 Compose；预发/生产用 K8s，但由
CLI/调度中心封装，对开发者保持"一键"。

### 9.2 Claude Code 集成（上下文工程）

    效率优先:
      - 先保障"快速达成开发目标"（安全能力后续增强）
    上下文工程:
      - 代码/文档嵌入入库 Milvus，按需检索；
      - 预设提示词与专有 Claude Agent：接口生成、Dockerfile、测试、诊断、优化
    落地方式:
      - CLI + IDE 插件；MR/PR 自动代码审查（AI 评分门槛 + 人工复核）

### 9.3 AI 驱动的质量保障

    开发时保障: IDE 实时代码审查/自动修复/安全检查
    CI/CD: AI 自动 Review（评分门槛），PR 自动评论与修复建议
    质量门禁: AI 评分 > 80、测试覆盖率 > 80%、无高危漏洞
    AI 规范: .ai-rules.yaml（命名/错误处理/密钥/性能）

## 10. 部署架构

### 10.1 容器化策略（本地/PoC 参考）

-   `docker-compose.yml` 用于本地/PoC 单机体验与演示；
-   生产与预发统一 K8s（见 10.2）。

**compose 参考**（节选）：

    version: '3.8'
    services:
      postgres:
        image: postgres:15
        environment:
          POSTGRES_PASSWORD: ${DB_PASSWORD}
        volumes:
          - postgres_data:/var/lib/postgresql/data
      logto:
        image: svhd/logto:latest
        environment:
          DB_URL: postgresql://postgres:${DB_PASSWORD}@postgres:5432/logto
        depends_on: [postgres]
      temporal:
        image: temporalio/server:latest
        environment:
          DB: postgresql
          POSTGRES_SEEDS: postgres
        depends_on: [postgres]
      temporal-ui:
        image: temporalio/ui:latest
        ports: ["8080:8080"]
      loki:
        image: grafana/loki:latest
        command: -config.file=/etc/loki/local-config.yaml
        volumes:
          - loki_data:/loki
      grafana:
        image: grafana/grafana:latest
        ports: ["3000:3000"]
        environment:
          - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      api:
        build: ./backend
        environment:
          DATABASE_URL: postgresql://postgres:${DB_PASSWORD}@postgres:5432/agent
        depends_on: [postgres, temporal]
      web:
        build: ./frontend
        ports: ["80:80"]
        environment:
          VITE_API_URL: http://api:3000
    volumes:
      postgres_data:
      loki_data:

### 10.2 生产环境（统一 K8s + 单一 LB 真相）

    统一策略：
      - 生产/预发统一 Kubernetes
      - 入口：Nginx（TLS/静态/基础路由） → K8s Ingress/Service（K8s 负责 LB）
      - 弹性：HPA + Probe + 滚动升级/回滚 + 金丝雀（Ingress/Gateway 或 Service Mesh）
      - 证书：Nginx 边缘证书 +（或）集群内 cert-manager；
      - 发布：镜像推送私库 → ArgoCD/GitOps 或 CLI 触发 K8s 部署
    高可用：
      - PostgreSQL 主从 + 自动故障转移 + 定期备份
      - MinIO 分布式/纠删码
      - 服务多副本 + HPA/垂直弹性
      - Tempo/Loki/Prometheus 高可用部署
    阶段性运行：
      - 本地/PoC：Docker Compose（便捷）
      - 预发/生产：Kubernetes（统一）

## 11. 实施路线图

### 11.1 第一阶段：基础平台（1-2 月）

    目标: 搭建核心基础设施
    Week 1-2: PostgreSQL/Logto/CASL+SpiceDB
    Week 3-4: NestJS/Vue3/Arco
    Week 5-6: Temporal/MinIO/Docker 化
    Week 7-8: Loki/Otel/Tempo/Prometheus+Grafana
    交付: 登录注册/项目管理/微服务注册/简单工作流/基础监控日志

### 11.2 第二阶段：AI 能力集成（3-4 月）

    目标: 集成 AI 能力，降低开发门槛
    Week 9-10: Claude API/LLM 服务/Milvus
    Week 11-12: Claude Code→CLI；AI 项目模板
    Week 13-14: Vue Flow 编辑器（蓝图风）
    Week 15-16: Swagger/K6/测试自动化
    交付: LLM 服务/AI CLI/可视化工作流/API 文档/测试体系

### 11.3 第三阶段：场景落地（5-6 月）

    目标: 解决实际游戏开发问题
    场景: 策划自动化/美术批处理/程序提效
    成功: 3 个核心场景日常使用；效率+70%；满意度>4.0

### 11.4 第四阶段：全面推广（7-12 月）

    推广: 培训/最佳实践/案例/激励
    技术: 更多模板/性能优化/UX 优化/成本优化
    成功: 活跃>100；服务>50；月任务>10000；ROI>3:1

## 12. 关键成功因素

    架构: 模块化/统一 TS/成熟方案/预留演进
    体验: 强大 CLI/AI 辅助/Vue 3 降门槛/文档示例
    运维: Loki 降成本/Compose 简化本地部署/完备监控/自动化

## 13. 风险与应对

    学习曲线: Temporal/gRPC/K8s → 培训+模板+AI 辅助
    性能瓶颈: 提前 K6 压测；HPA/分片/缓存预案
    系统复杂度: 模块化+清晰边界；优先 K8s 主路径，VM/Custom 按需引入
    AI 可靠性: AI+测试+人工复核三重保障；逐步提高门禁

## 14. 技术选型总结

### 14.1 最终技术栈

    前端: Vue 3+Vite / Arco / Vue Flow / Pinia / TS
    后端: NestJS / TS / gRPC / Swagger
    基建: PostgreSQL / MinIO / Milvus / Loki / Temporal
    认证权限: Logto / CASL / SpiceDB
    监控运维: Prometheus+Grafana / OpenTelemetry+Tempo / K6 / Docker / Kubernetes
    工具: Claude Code / 自研 CLI / Jest / Playwright
    边缘入口: Nginx（TLS/静态/基础路由）→ K8s Ingress/Service（LB）

### 14.2 核心优势

1\) 技术栈统一 + 多语言微服务（gRPC 契约） 2) AI 原生（Claude Code
深度集成） 3) 游戏友好（可视化蓝图 + 长任务编排） 4)
成本优化（Loki/一库多用） 5) 现代 UI（Arco 科技风）

## 15. 总结

通过 **统一 TS 平台 + 多语言微服务（gRPC 契约）**、**Temporal
长任务编排**、**Nginx 边缘入口 + K8s
作为负载均衡真相源**、**Loki/Tempo/Prometheus 完整可观测** 与 **Claude
Code 深度集成**，GameWorkAgent
平台为游戏研发团队提供了一套低门槛、高效率、可扩展的 AI Agent 基建。
一键 CLI + 可视化编排 +
资源声明/权限治理，使策划/美术/程序等多角色协同更加顺畅；K8s
统一生产路径保障了弹性与稳健；AI
辅助显著降低了微服务开发与运维门槛。按路线图推进，即可实现"让每个人都能开发微服务"的愿景。

# 附：示意图（Mermaid）

## A. 流量与负载拓扑（Nginx → K8s）

    flowchart LR
      Client((Client / CI / UE Tool)) -->|HTTPS| Nginx["Nginx\nTLS / Static / Basic Routing"]
      Nginx -->|/api/*| Ingress["K8s Ingress / Gateway"]
      Nginx -->|/ (SPA)| SPA["Static SPA (Vue3)"]
      Ingress --> SvcAPI["Service: gwa-api"]
      Ingress --> SvcWF["Service: workflow"]
      Ingress --> SvcLLM["Service: llm-gateway"]
      SvcAPI --> PodAPI1((Pod API #1))
      SvcAPI --> PodAPI2((Pod API #N))
      SvcWF --> PodWF1((Pod WF #1))
      SvcWF --> PodWF2((Pod WF #M))
      SvcLLM --> PodLLM1((Pod LLM #1))
      SvcLLM --> PodLLM2((Pod LLM #K))
      subgraph Kubernetes Cluster
        Ingress
        SvcAPI
        SvcWF
        SvcLLM
        PodAPI1
        PodAPI2
        PodWF1
        PodWF2
        PodLLM1
        PodLLM2
      end
      PodWF1 -. Otel Traces/Logs .-> Tempo[(Tempo)]
      PodAPI1 -. Logs .-> Loki[(Loki)]
      Tempo -. Dashboards .-> Grafana[(Grafana)]
      Loki -. Dashboards .-> Grafana

## B. Service Declaration 审批与 Owner Transfer 流程

    flowchart TB
      dev[开发者] -->|gwa declare| sc[调度中心]
      sc -->|校验 proto/接口/Schema| registry[服务声明库]
      registry -->|引用| market[微服务市场(节点库)]
      dev2[工作流编辑器] -->|拖拽节点| market
      dev2 -->|保存/发布流程| sc
      sc -->|资源需求检查| spicedb[(SpiceDB)]
      spicedb -->|已有绑定?| 
      decision{绑定存在?}
      spicedb --> decision
      decision -- 否 --> approval[审批流(域/项目管理员)]
      approval --> spicedb
      decision -- 是 --> start[可调度]
      start -->|按需启动| provider{Provider}
      provider -->|k8s/vm/custom| runtime[(运行实例)]
      runtime -->|Runtime 注册| sc
      sc -->|更新路由| ingress[Ingress/Gateway]
      ingress -->|统一入口| nginx[Nginx]

      subgraph 变更与离职
        owner[资源/服务 Owner] --> change[发起转移]
        change --> approval2[审批(域/项目管理员)] --> spicedb
        emergency[超管/域管] --> force[紧急强制迁移] --> spicedb
      end

## C. 运行时编排（可视化蓝图 → Temporal）

    sequenceDiagram
      participant U as 用户(编辑器)
      participant FE as Vue Flow
      participant SC as 调度中心
      participant TMP as Temporal
      participant WK as Workers

      U->>FE: 拖拽/连线/配置节点
      FE->>SC: 提交工作流定义(DSL/图)
      SC->>TMP: 生成并启动 Workflow (SDK/DSL)
      TMP->>WK: 调度 Activity (重试/超时/补偿)
      WK-->>TMP: 执行结果/失败重试/幂等
      TMP-->>SC: 状态与 TraceId
      SC-->>U: 实时状态/追踪/日志链接
