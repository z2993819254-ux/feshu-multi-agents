---
name: claw-system
description: OpenClaw 多部门任务协调系统总览。定义了整个系统的架构、每只 claw 需要的 skill 组合、看板结构、部署流程。这是系统的"宪法"，每只 claw 首次启动时必读。
---

# OpenClaw 多部门任务协调系统

## 系统架构

```
项目经理/负责人
    ↓ 创建任务
飞书多维表格（任务看板）
    ↓ Webhook + 心跳轮询
各部门 OpenClaw（每部门一只）
    ↓ 能力匹配 → 领取 → 执行 → 汇报
任务完成，状态回写看板
```

## 每只 claw 需要的 Skill

### 必装 Skill（系统级）

| Skill | 作用 |
|-------|------|
| `claw-system` | 系统总览，"宪法"，首次启动必读 |
| `claw-task-board` | 任务看板交互：领取、匹配、执行、汇报 |
| `claw-coordinator` | 跨部门协调：异步(协调表) + 实时(sessions_send) |
| `claw-heartbeat` | 心跳轮询：扫描新任务、处理超时、维护状态 |

### 业务 Skill（按部门配置）

| 部门 | 推荐 Skill |
|------|-----------|
| 营销 | feishu-doc, web-search, xiaohongshu, browser-automation, edu-insights |
| 研发 | coding-agent, feishu-doc |
| 设计 | browser-automation, feishu-doc |
| 运营 | feishu-doc, web-search, xiaohongshu-comments, douyin-danmu |
| 产品 | feishu-doc, web-search, feishu-wiki |

### Skill 沉淀规则

- 同类任务出现 **2次以上** → 必须沉淀为新 skill
- 新 skill 包含：SKILL.md + 脚本/工具
- 新 skill 创建后需**人工审核一次**确认质量
- 沉淀后更新注册表的 skill 列表

## 看板结构（飞书多维表格）

### 任务表（主表）

**基础字段：**
- 任务ID（自动编号）
- 任务名称（文本）
- 任务描述（文本）
- 任务类型（单选）：文档撰写 / 数据处理 / 代码开发 / 调研分析 / 设计制作 / 其他
- 所需技能（多选）：对应 skill 标签
- 父任务ID（文本）：支持任务拆解

**流转字段：**
- 状态（单选）：待分配 → 待领取 → 进行中 → 待审核 → 已完成 / 已退回 / 需人工
- 优先级（单选）：P0紧急 / P1高 / P2中 / P3低
- 所属部门（单选）：营销 / 研发 / 设计 / 运营 / 产品
- 负责人（人员）
- 领取的claw（文本）
- 截止/领取/完成时间（日期）

**交付字段：**
- 执行摘要（文本）：100字内
- 产出物类型（单选）：文档链接 / 数据文件 / 代码PR / 纯文本
- 产出物链接（URL）
- 自评分数（数字）：1-10
- 遗留问题（文本）
- 退回原因（文本）

### 协调表

- 对齐ID（自动编号）
- 关联任务ID / 发起方 / 接收方（文本）
- 问题描述 / 对话记录 / 结论（文本）
- 对齐状态（单选）：发起 / 对齐中 / 已解决 / 超时升级
- 当前轮次（数字）：上限5轮
- 创建/解决时间（日期）

### 注册表

- Claw ID / 所属部门
- Skill 列表（多选）
- 最后心跳时间（日期）
- 今日完成/失败任务数（数字）
- 状态（单选）：在线 / 忙碌 / 离线
- 当前执行任务ID（文本）

## 三层保障

1. **Webhook（优先）** — 看板变更时飞书推送，零延迟
2. **心跳轮询（兜底）** — 每5-10分钟扫描看板
3. **实时通信（协调）** — sessions_send 跨 claw 对话

## 人工介入红线

**必须通知人工：**
- 任务执行失败 2 次
- 跨部门对齐超 5 轮未解决
- 涉及金额 / 合同 / 对外发布
- claw 自评信心低于 6 分
- 需求变更或优先级调整

**claw 可自主处理：**
- 纯技术细节（格式、参数、接口）
- 有 skill 支持的标准化任务
- 文档整理、数据查询

## 落地步骤

### Phase 0: 创建看板
1. 创建飞书多维表格
2. 按上面的字段创建任务表 + 协调表 + 注册表
3. 配置看板视图

### Phase 1: 单 claw 试点
1. 给一只 claw 安装 4 个系统 skill
2. 配置 `claw_config.json`
3. 在 HEARTBEAT.md 加入心跳巡逻
4. 手动创建几个任务，验证全流程

### Phase 2: 多 claw 扩展
1. 每个部门部署一只 claw
2. 各自配置部门信息和 skill
3. 验证跨部门协调流程

### Phase 3: 云端 Gateway
1. 云服务器部署 Gateway
2. 各 Node 配对
3. sessions_send 实时通信验证

## 原始方案文档

飞书文档: https://guanghe.feishu.cn/docx/O248dBCGsoDrgBxG7ARc45Zzn6N
