# 核心业务流程图

本文档按业务逻辑展示系统链路，避免仅用一张总图概括复杂系统。

## 1. 用户登录与基础能力链路

```mermaid
flowchart LR
    A[用户注册/登录] --> B[Java 后端校验]
    B --> C[生成 Token]
    C --> D[Redis 保存会话]
    D --> E[后续请求携带 Token]
    E --> F[Spring Security / Filter 校验]
    F --> G[用户上下文注入]
    G --> H[访问在线监测 / 实验 / Agent / 历史接口]
```

基础链路支撑的是后续所有业务的数据隔离和访问控制。

## 2. 在线疲劳监测链路

```mermaid
sequenceDiagram
    participant Device as EEG 头环设备
    participant App as Flutter 移动端
    participant Java as Java 后端
    participant Algo as Python 疲劳算法服务
    participant DB as MySQL / Redis

    Device->>App: BLE 连接与 EEG 数据流
    App->>App: 设备状态展示 / 波形展示
    App->>App: 基线校准缓冲
    App->>Java: 上传基线数据
    Java->>Algo: 建立个体基线
    Algo-->>Java: 返回校准结果
    Java-->>App: 进入监测模式
    loop 固定时间窗
        App->>Java: 上传 EEG 窗口
        Java->>Java: 认证 / 限流 / 参数检查
        Java->>Algo: 在线疲劳预测
        Algo-->>Java: 返回疲劳分数
        Java->>Java: 聚合风险等级与趋势
        Java->>DB: 保存疲劳记录
        Java-->>App: 返回状态结果
    end
```

## 3. 离线实验数据采集链路

```mermaid
sequenceDiagram
    participant Web as Web 实验平台
    participant App as Flutter App
    participant Java as Java 后端
    participant Archive as 数据库 / 文件归档

    Web->>Java: 创建实验 session
    Java-->>Web: 返回 session 信息
    App->>Java: 绑定用户与 session
    Web->>Java: Web Ready
    App->>Java: App Ready
    Java-->>Web: 双端 Ready / 统一倒计时
    Java-->>App: 同步开始采集
    par Web 侧实验事件
        Web->>Java: 上传阶段事件 / trial 结果 / 主观评分
    and App 侧 EEG 采集
        App->>Java: 分段上传 EEG chunk
    end
    Web->>Java: 实验结束确认
    App->>Java: 采集结束确认
    Java->>Archive: 归档 EEG 数据、事件、评分和元信息
```

## 4. Agent 智能干预链路

```mermaid
flowchart TD
    A[用户提问或系统触发提醒] --> B[Java 获取当前在线 session]
    B --> C[聚合实时疲劳状态]
    C --> D[补充历史趋势与近期干预]
    D --> E[形成结构化上下文]
    E --> F[Python Agent Service]
    F --> G[RAG / Tool / 模型调用]
    G --> H[规则约束与输出整理]
    H --> I[状态解释 / 休息建议 / 后续入口]
    I --> J[Flutter 展示]
```

## 5. 历史回放与评估链路

```mermaid
flowchart LR
    A[疲劳状态记录] --> D[历史聚合服务]
    B[实验结果记录] --> D
    C[干预事件记录] --> D
    D --> E[当前状态 state]
    D --> F[历史统计 history_analytics]
    D --> G[上下文 context]
    E --> H[用户历史页面]
    F --> H
    G --> I[Agent 上下文]
    D --> J[最近窗口增量数据]
    J --> H
```

## 6. 系统闭环

```mermaid
flowchart LR
    A[在线采集] --> B[疲劳识别]
    B --> C[风险评估]
    C --> D[实时展示]
    C --> E[Agent 干预]
    B --> F[历史记录]
    F --> G[趋势回放]
    G --> E
    F --> H[算法迭代与实验分析]
```

该闭环体现系统从单一预测扩展到长期评估和智能干预的能力。
