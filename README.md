# Feishu Multi-Agents（飞书多 Claw 任务协调系统）

> 基于 OpenClaw 的多部门 AI Agent 协调系统。每个部门一只 Claw，通过飞书多维表格看板实现任务自动分配、领取、执行、汇报。

## 架构

```
项目经理/负责人
    ↓ 创建任务（在飞书多维表格）
飞书多维表格（任务看板）
    ↓ 心跳轮询
各部门 Claw（每部门一只 OpenClaw 实例）
    ↓ 能力匹配 → 领取 → 执行 → 汇报
任务完成，状态回写看板
```

## 核心组件

### 4 个系统 Skill

| Skill | 作用 | 说明 |
|-------|------|------|
| `claw-system` | 系统宪法 | 定义架构、看板结构、部署流程。每只 Claw 首次启动必读 |
| `claw-task-board` | 任务看板交互 | 任务发现 → 能力匹配 → 领取 → 执行 → 汇报 → 异常处理 |
| `claw-coordinator` | 跨部门协调 | 异步（飞书协调表）+ 实时（sessions_send）+ 自动升级 |
| `claw-heartbeat` | 心跳巡逻 | 定时扫描新任务、检查超时、更新自身状态 |

### 飞书多维表格（看板）

所有 Claw 共用一个飞书多维表格，包含 3 张表：

| 表 | 用途 |
|----|------|
| **任务表** | 任务创建、领取、执行、汇报的全生命周期 |
| **协调表** | 跨部门对齐的对话记录 |
| **注册表** | 每只 Claw 的身份、能力、心跳状态 |

### 配置文件

- `claw_config.json` — 每只 Claw 的身份和能力声明
- `claw_config.template.json` — 配置模板（复制后按部门修改）

## 快速开始

### 前置条件

1. 已安装 [OpenClaw](https://github.com/openclaw/openclaw)
2. 已配置飞书应用（有 `bitable:record` 读写权限）
3. 已创建飞书多维表格看板（结构见下方）

> **权限说明：** 看板已设置为「组织内获得链接的人可编辑」，同组织下的飞书应用无需单独添加协作者，拿到看板 URL 即可读写。

### Step 1: 复制 Skill 到 workspace

```bash
# 克隆本仓库
git clone https://github.com/z2993819254-ux/feshu-multi-agents.git

# 复制 4 个系统 Skill 到你的 OpenClaw workspace
cp -r feshu-multi-agents/skills/claw-* ~/.openclaw/workspace/skills/
```

### Step 2: 创建配置文件

```bash
# 复制配置模板
cp feshu-multi-agents/claw_config.template.json ~/.openclaw/workspace/claw_config.json

# 按你的部门修改配置（见下方说明）
```

编辑 `claw_config.json`，修改以下字段：

| 字段 | 说明 | 示例 |
|------|------|------|
| `claw_id` | 全局唯一标识，建议 `部门-claw` 格式 | `marketing-claw` |
| `department` | 所属部门 | `营销` |
| `bitable_app_token` | 飞书多维表格的 app_token | 从表格 URL 中获取 |
| `task_table_id` | 任务表的 table_id | 从表格 URL 中获取 |
| `coord_table_id` | 协调表的 table_id | 从表格 URL 中获取 |
| `registry_table_id` | 注册表的 table_id | 从表格 URL 中获取 |
| `skills` | 该 Claw 拥有的 skill 列表 | 见部门推荐 |
| `human_notify_target` | 遇到问题时通知谁 | `user:ou_xxxx` |

### Step 3: 在看板注册

在飞书多维表格的**注册表**中新增一行：
- Claw ID = 你的 `claw_id`
- 所属部门 = 你的部门
- Skill列表 = 你拥有的 skill
- 状态 = 在线

### Step 4: 配置心跳

在 `~/.openclaw/workspace/HEARTBEAT.md` 中添加：

```markdown
### Claw 任务巡逻
- 读取 claw_config.json 获取配置
- 扫描任务表：筛选「状态=待领取」且「所属部门=自己部门」的任务
- 检查协调表：筛选「接收方=自己」且未解决的记录
- 更新注册表：刷新心跳时间和状态
- 如发现匹配任务，走 claw-task-board 的领取流程
```

### Step 5: 验证

启动 OpenClaw 后，让 Claw 执行一次心跳巡逻。在任务表中创建一个测试任务（状态=待领取，所属部门=你的部门），检查 Claw 是否自动领取。

## 各部门推荐 Skill 配置

| 部门 | claw_id | 推荐 skills |
|------|---------|------------|
| 营销 | `marketing-claw` | `feishu-doc`, `web-search`, `browser-automation`, `xiaohongshu`, `edu-insights` |
| 研发 | `dev-claw` | `coding-agent`, `feishu-doc` |
| 设计 | `design-claw` | `browser-automation`, `feishu-doc` |
| 运营 | `ops-claw` | `feishu-doc`, `web-search`, `xiaohongshu-comments`, `douyin-danmu` |
| 产品 | `product-claw` | `feishu-doc`, `web-search`, `feishu-wiki` |

## 看板字段结构

### 任务表

| 字段 | 类型 | 说明 |
|------|------|------|
| 任务ID | 自动编号 | 唯一标识 |
| 任务名称 | 文本 | 主字段 |
| 任务描述 | 文本 | 详细说明 |
| 任务类型 | 单选 | 文档撰写/数据处理/代码开发/调研分析/设计制作/其他 |
| 所需技能 | 多选 | 对应 skill 标签 |
| 父任务ID | 文本 | 支持任务拆解 |
| 状态 | 单选 | 待分配→待领取→进行中→待审核→已完成/已退回/需人工 |
| 优先级 | 单选 | P0紧急/P1高/P2中/P3低 |
| 所属部门 | 单选 | 营销/研发/设计/运营/产品 |
| 负责人 | 人员 | 飞书用户 |
| 领取的claw | 文本 | 哪只 Claw 领取了 |
| 截止/领取/完成时间 | 日期 | 时间节点 |
| 执行摘要 | 文本 | 100字内总结 |
| 产出物类型 | 单选 | 文档链接/数据文件/代码PR/纯文本 |
| 产出物链接 | URL | 交付物地址 |
| 自评分数 | 数字 | 1-10 |
| 遗留问题 | 文本 | 未完成部分 |
| 退回原因 | 文本 | 被退回时填写 |

### 协调表

| 字段 | 类型 | 说明 |
|------|------|------|
| 对齐ID | 自动编号 | 唯一标识 |
| 关联任务ID | 文本 | 对应任务表 |
| 发起方/接收方 | 文本 | claw_id |
| 问题描述 | 文本 | 具体问题 |
| 对话记录 | 文本 | [claw_id HH:MM] 内容 格式 |
| 结论 | 文本 | 最终结论 |
| 对齐状态 | 单选 | 发起/对齐中/已解决/超时升级 |
| 当前轮次 | 数字 | 上限 5 轮 |
| 创建/解决时间 | 日期 | 时间节点 |

### 注册表

| 字段 | 类型 | 说明 |
|------|------|------|
| Claw ID | 文本 | 全局唯一标识 |
| 所属部门 | 单选 | 部门 |
| Skill列表 | 多选 | 拥有的能力 |
| 最后心跳时间 | 日期 | 存活检测 |
| 今日完成/失败任务数 | 数字 | 当日统计 |
| 状态 | 单选 | 在线/忙碌/离线 |
| 当前执行任务ID | 文本 | 正在做什么 |

## 任务状态流转

```
待分配 → 待领取 → 进行中 → 待审核 → 已完成
                     ↓          ↓
               已退回/需人工  已退回
```

## 人工介入红线

**必须通知人工的情况：**
- 任务执行失败 2 次
- 跨部门对齐超 5 轮未解决
- 涉及金额/合同/对外发布
- Claw 自评信心低于 6 分
- 需求变更或优先级调整

**Claw 可自主处理：**
- 纯技术细节（格式、参数、接口）
- 有 skill 支持的标准化任务
- 文档整理、数据查询

## 落地阶段

| 阶段 | 目标 |
|------|------|
| Phase 0 | 创建飞书多维表格看板，配置字段 |
| Phase 1 | 单 Claw 试点，验证全流程 |
| Phase 2 | 多 Claw 扩展，每部门一只 |
| Phase 3 | 云端 Gateway，跨机器通信 |

## 踩坑记录（实战验证 2026-03-03）

以下是首次完整跑通全流程时遇到的坑，新 Claw 接入前务必看一遍。

### 坑 1：飞书文档 create 不写正文

`feishu_doc create` 的 `content` 参数**不会写入正文**，只会创建一个空文档（仅有标题）。

**正确做法：**
```
1. feishu_doc create → 拿到 doc_token（此时文档是空的）
2. feishu_doc append → 分段写入正文内容
3. 单次 append 内容不能太长，长文档必须分多次 append
```

**错误做法：**
```
feishu_doc create(title="xxx", content="一大段内容...")  ← 内容会丢失！
```

### 坑 2：创建文档后必须开权限

用 `feishu_doc create` 创建的文档，默认只有应用自己能访问。**必须立即给对话人/协作者开权限**，否则对方打不开。

**操作：** 用飞书权限 API 给目标用户添加 `full_access` 权限。参考 `feishu-perm-api` skill。

### 坑 3：任务完成必须回写产出物链接

任务执行完后回写看板时，容易只更新「状态」和「执行摘要」，**忘了填「产出物链接」**。

**完整的回写清单：**
- [ ] 状态 → 待审核
- [ ] 执行摘要 → 100字内总结
- [ ] 产出物类型 → 文档链接/数据文件/代码PR/纯文本
- [ ] **产出物链接 → URL**（这个最容易漏！）
- [ ] 自评分数 → 1-10
- [ ] 遗留问题 → 有就写
- [ ] 完成时间 → 当前时间戳
- [ ] 注册表：状态→在线，当前任务ID→清空，完成数+1

### 坑 4：注册表数字字段不能传字符串

注册表的「今日完成任务数」是 Number 类型，传 `"1"`（字符串）会报错 `NumberFieldConvFail`，必须传 `1`（数字）。

**所有 Number 类型字段都要注意这个问题。**

### 坑 5：多维表格权限问题

新 Claw 的飞书应用默认没有看板的读写权限。

**解决方案（二选一）：**
- **推荐：** 把多维表格设为「组织内获得链接的人可编辑」（API: `PATCH /drive/v1/permissions/{token}/public`，body: `{"link_share_entity":"tenant_editable"}`）
- **备选：** 手动在多维表格「共享」中添加每只 Claw 的飞书应用为协作者

### 坑 6：不要因为一条路走不通就放弃

执行任务时如果某个工具/API 不可用（比如 `web_search` 没配 API key），**不要直接上报"做不了"**。

**正确姿势：**
1. API 不行 → 试已有的 Skill 脚本
2. Skill 不行 → 开浏览器手动操作
3. 浏览器不行 → 试 CLI 工具、其他网站、其他方法
4. 至少尝试 3 种途径后才能说"做不了"

### 坑 7：feishu_doc append 内容过长会 400

单次 `feishu_doc append` 写入内容太多会返回 `400 Bad Request`。

**解决方案：** 长文档拆成多次 append，每次写一个章节（几百字以内比较安全）。

### 坑 8：URL 字段需要 {link, text} 格式

多维表格的 URL 类型字段不能直接传字符串，需要传对象：
```json
{
  "产出物链接": {
    "link": "https://feishu.cn/docx/xxxxx",
    "text": "报告标题"
  }
}
```

### 坑 9：心跳状态文件要初始化

首次启动时记得创建 `claw_heartbeat_state.json`，否则心跳巡逻时读取会报错。

```json
{
  "last_heartbeat": null,
  "last_webhook_received": null,
  "tasks_completed_today": 0,
  "tasks_failed_today": 0,
  "current_task_id": null,
  "pending_coord_ids": []
}
```

## License

MIT
