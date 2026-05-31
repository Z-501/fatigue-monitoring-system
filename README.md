# 疲劳监测与智能干预 Agent 系统

> 本仓库为项目展示仓库，用于说明系统架构、业务链路、前后端协同方式、算法服务接入与 Agent 智能干预闭环。由于项目涉及源代码版权、实验数据与设备采集链路，完整源码暂不公开；本仓库仅展示脱敏后的架构设计、业务流程、接口边界与演示材料入口。

## 1. 项目定位

本项目面向办公、学习、实验研究与长时间脑力劳动等疲劳监测场景，构建了一套从 **EEG 数据采集、疲劳状态识别、风险评估、智能干预到历史回放** 的多端协同系统。

系统不是单一算法 Demo，也不是传统后台管理系统，而是围绕真实设备接入和疲劳监测业务闭环构建的全栈工程系统。整体采用：

- **三端接入**：Flutter 移动端、Web 实验平台、Agent/评估可视化入口
- **后端统一编排**：Java Spring Boot 负责认证、会话、状态同步、限流、缓存、算法调用和结果管理
- **Python 分层计算**：Python 疲劳算法服务负责 EEG 校准与在线推理，Python Agent 服务负责智能解释与干预建议
- **数据分层存储**：MySQL 保存业务数据，Redis/Caffeine 支撑高频状态查询，文件化归档支撑实验数据与后续分析

系统同时支持 **在线疲劳监测** 与 **离线实验数据采集** 两条主链路：前者面向实时风险提示和用户交互，后者面向标准化实验采集、算法适配与科研分析。

## 2. 项目复杂度概览

| 维度 | 说明 |
| --- | --- |
| 前端工程 | Flutter 移动端、Web 实验平台、Agent/评估可视化入口 |
| 后端工程 | Java Spring Boot 业务中台，统一处理认证、会话、缓存、限流、状态同步与服务编排 |
| 算法服务 | Python EEG 疲劳识别服务，支持基线校准、分窗推理与结果标准化 |
| Agent 服务 | Python Agent Service，支持上下文构造、RAG/工具调用、状态解释和干预建议生成 |
| 数据链路 | 在线实时监测数据、离线实验 session、行为事件、疲劳结果、干预记录、历史回放视图 |
| 工程规模 | 项目包含多个前后端与算法服务子工程，核心源码规模约 5-6 万行 |

## 3. 总体架构

### 3.1 系统分层架构

```mermaid
flowchart LR
    subgraph Client[终端交互层]
        App[Flutter 移动端\n设备连接 / 在线监测 / 历史报告 / Agent 交互]
        Web[Web 实验平台\n实验范式 / 行为事件 / 主观评分 / 结果查看]
        Eval[Agent 评估与可视化入口\nPrompt / Trace / Response / Eval]
    end

    subgraph Backend[Java 业务服务层]
        Auth[认证鉴权与访问控制]
        Session[在线监测与实验 Session 管理]
        Orchestrator[任务编排与状态同步]
        RateLimit[接口限流 / 异常封装 / 缓存治理]
        History[历史回放与统计聚合]
    end

    subgraph Intelligence[Python 智能分析层]
        Fatigue[EEG 疲劳算法服务\n基线校准 / 分窗预测 / 结果标准化]
        Agent[健康管家 Agent 服务\n上下文构造 / RAG / Tool / 干预建议]
    end

    subgraph Data[数据资源层]
        MySQL[(MySQL\n用户 / 会话 / 结果 / 报告)]
        Redis[(Redis\n会话 / 缓存 / 限流 / 热点状态)]
        Files[(文件化归档\n实验数据 / EEG chunk / 报告材料)]
    end

    App --> Backend
    Web --> Backend
    Eval --> Agent
    Backend --> Fatigue
    Backend --> Agent
    Backend --> MySQL
    Backend --> Redis
    Backend --> Files
```


### 3.2 系统业务总览

总体分层架构用于说明“有哪些系统角色”，业务总览图用于说明“这些角色如何围绕疲劳监测闭环协同”。

```mermaid
flowchart TD
    Start[用户进入系统] --> Login[用户认证与访问控制]

    Login --> Online[在线疲劳监测链路]
    Login --> Experiment[离线实验采集链路]
    Login --> History[历史回放与评估链路]

    subgraph OnlineFlow[在线疲劳监测]
        Online --> Device[Flutter 连接头环设备]
        Device --> Calib[基线校准与端侧缓冲]
        Calib --> Window[EEG 分窗上传]
        Window --> Predict[Python 疲劳算法推理]
        Predict --> State[Java 聚合疲劳分数 / 风险等级 / 趋势方向]
        State --> Realtime[Flutter 实时展示与提醒]
    end

    subgraph ExperimentFlow[离线实验采集]
        Experiment --> WebTask[Web 实验范式与任务控制]
        Experiment --> AppCollect[Flutter EEG 持续采集]
        WebTask --> Session[Java 实验 Session 同步]
        AppCollect --> Session
        Session --> Archive[实验事件 / EEG chunk / 主观评分归档]
    end

    subgraph AgentFlow[智能干预闭环]
        State --> Context[Java 构建健康状态上下文]
        History --> Context
        Context --> Agent[Python Agent 智能解释与干预建议]
        Agent --> Realtime
        Agent --> Intervention[干预记录沉淀]
    end

    subgraph ReviewFlow[历史回放与评估]
        State --> Record[疲劳记录入库]
        Archive --> Record
        Intervention --> Record
        Record --> Aggregation[统计聚合 / 趋势分析 / 最近窗口刷新]
        Aggregation --> HistoryView[历史趋势 / 报告 / Agent 上下文复用]
    end
```

## 4. 核心业务链路

### 4.1 用户登录与基础能力链路

基础链路负责支撑后续在线监测、离线实验、Agent 交互和历史回放。系统采用 Token + Redis 会话方式管理登录态，Java 后端通过统一过滤链完成身份识别，并在高频接口前引入限流、异常封装和缓存治理能力。

```mermaid
sequenceDiagram
    participant U as 用户 / Flutter App
    participant J as Java 后端
    participant R as Redis
    participant DB as MySQL

    U->>J: 注册 / 登录
    J->>DB: 校验用户信息
    J->>R: 写入 Token 会话
    J-->>U: 返回访问凭证
    U->>J: 携带 Token 访问业务接口
    J->>R: 校验会话有效性
    J-->>J: 注入当前用户上下文
    J-->>U: 返回业务结果
```

该链路的价值不只是“登录”，而是为多端数据隔离、在线监测会话、实验 session 绑定和 Agent 上下文构造提供统一身份基础。

### 4.2 在线疲劳监测链路

在线监测链路面向实时疲劳感知场景。Flutter 移动端负责头环连接、设备状态展示、EEG 数据接收、基线校准缓冲和预测窗口组织；Java 后端负责认证、限流、请求编排、算法服务调用、结果聚合和状态推送；Python 疲劳算法服务负责基线建立和在线推理。

```mermaid
flowchart TD
    A[用户佩戴头环并连接 Flutter App] --> B[App 接收连接状态 / 佩戴状态 / 电量 / 心率 / 双通道 EEG]
    B --> C[基线校准阶段\n约 30 秒个体参考数据]
    C --> D[Java 后端校验请求并转发校准数据]
    D --> E[Python 疲劳算法服务建立个体基线]
    E --> F[进入在线监测模式]
    F --> G[App 按时间窗组织 EEG 数据\n例如固定采样点窗口]
    G --> H[Java 后端限流 / 参数校验 / 会话绑定]
    H --> I[Python 服务执行疲劳预测]
    I --> J[Java 聚合疲劳分数 / 风险等级 / 趋势方向]
    J --> K[Flutter 实时展示疲劳状态]
    J --> L[触发智能干预或历史记录写入]
```

该链路体现的是“设备数据 → 个体基线 → 分窗预测 → 风险状态 → 实时反馈”的完整闭环，而不是简单调用一次模型接口。

### 4.3 离线实验数据采集链路

离线实验链路面向标准化实验采集和算法迭代。Web 实验端负责实验范式、静息段/任务段控制、行为事件、trial 结果和主观评分；Flutter App 负责 EEG 持续采集、chunk 分段上传和结束确认；Java 后端负责 session 创建、双端 ready 判断、统一倒计时、状态推进和最终归档。

```mermaid
sequenceDiagram
    participant Web as Web 实验平台
    participant App as Flutter App
    participant J as Java 后端
    participant Store as 数据库 / 文件归档

    Web->>J: 创建实验 session
    J-->>Web: 返回 sessionId 与实验访问信息
    App->>J: 绑定实验 session 并准备 EEG 采集
    Web->>J: Web 端 Ready
    App->>J: App 端 Ready
    J-->>Web: 双端就绪，进入统一倒计时
    J-->>App: 同步采集开始状态
    Web->>J: 上传实验事件 / trial 结果 / 主观评分
    App->>J: 持续上传 EEG chunk
    Web->>J: 实验结束确认
    App->>J: EEG 采集结束确认
    J->>Store: 归档实验数据、事件与元信息
```

该链路的核心在于保证 **行为事件与 EEG 数据处于同一实验会话边界内**，后续可用于模型适配、特征分析和科研验证。

### 4.4 Agent 智能干预链路

Agent 不被设计成脱离业务数据的普通聊天机器人，而是建立在当前在线监测 session、实时疲劳状态、历史趋势、近期干预记录和用户输入之上。Java 后端负责将业务数据聚合为结构化上下文，Python Agent 服务负责解释、建议和交互响应。

```mermaid
flowchart LR
    U[用户问题 / 系统提醒事件] --> J[Java 聚合上下文]
    J --> Ctx[结构化上下文\n当前疲劳状态 / 趋势方向 / 数据质量 / 历史干预 / 用户输入]
    Ctx --> A[Python Agent Service]
    A --> RAG[RAG / 知识卡片 / 工具调用]
    A --> Rule[规则约束\n高风险 / 主诉冲突 / 重复干预]
    RAG --> Out[状态解释 / 干预建议 / 后续入口]
    Rule --> Out
    Out --> J
    J --> App[Flutter 展示建议与交互入口]
```

该链路强调的是“受业务上下文约束的健康建议生成”，而不是泛化聊天能力。

### 4.5 历史回放与评估链路

历史回放链路用于将实时监测结果、实验结果和干预事件重新组织为可展示、可复用的健康视图。它既服务于用户趋势查看，也服务于 Agent 上下文构造、日报周报生成和系统评估。

```mermaid
flowchart TD
    A[疲劳记录 / 实验结果 / 干预事件] --> B[Java 历史聚合服务]
    B --> C[当前状态层 state\n最近疲劳状态 / 风险等级 / 趋势]
    B --> D[历史分析层 history_analytics\n24h / 7d 风险占比 / 峰值点 / 突变点]
    B --> E[上下文层 context\n最近干预 / 会话信息 / 场景参数]
    C --> F[Flutter 历史页面]
    D --> F
    E --> G[Agent 上下文]
    B --> H[最近窗口增量接口\n支撑前端曲线刷新]
```

历史回放不是简单查询数据库明细，而是将原始记录聚合为可视化、可解释、可复用的数据视图。

## 5. 划分

该展示仓库按照业务逻辑划分。

- 用户登录与基础链路说明系统入口和工程治理能力
- 在线疲劳监测链路说明真实设备数据如何进入业务闭环
- 离线实验采集链路说明 Web 端、App 端和后端如何协同完成标准化数据采集
- Agent 智能干预链路说明 AI 能力如何接入真实业务上下文
- 历史回放链路说明系统如何从单次预测扩展到长期趋势、评估和复用

## 6. 仓库内容

```text
.
├── README.md
├── docs/
│   ├── architecture.md          # 系统分层架构与业务链路设计
│   ├── business-workflows.md    # 按业务逻辑拆分的核心流程图
│   ├── api-overview.md          # 脱敏接口概览
│   ├── testing-strategy.md      # 测试与验证策略概览
│   ├── diagrams/                # 可放置架构图、流程图导出图片
│   └── screenshots/             # 可放置系统截图
├── LICENSE
└── .gitignore
```

## 7. 项目边界说明

本仓库不公开完整源码、真实实验数据、模型权重、密钥配置和可识别用户数据。公开内容仅用于展示系统设计、工程复杂度、业务链路和项目实现思路。

