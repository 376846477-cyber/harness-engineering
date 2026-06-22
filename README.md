---
name: harness-engineering
description: Harness工程化开发多Agent协作系统 - 端到端的软件工程开发流程管理体系。Harness Agent统筹调度6个子Agent（SA/Design/Coder/Review/Tester/CMO），覆盖需求分析→规格编写→架构设计→编码实现→一致性审查→质量验证→配置管理与发布全生命周期。适用场景：(1) 新项目启动；(2) 功能开发；(3) 代码重构；(4) 质量提升；(5) 部署发布。触发关键词：harness开发模式、工程化开发、多Agent协作、软件开发流程、DevOps、CI/CD、质量门禁、配置管理
---

# Harness工程化开发 - 多Agent协作系统

## 概述

Harness工程化开发是一套完整的多Agent协作软件工程开发体系，融合了**Harness工程开发模式**和**规范驱动开发(SDD)**的核心能力。通过Harness主Agent编排6个专业子Agent，实现从需求到交付的全链路闭环管理。

## 系统架构

```
用户需求
    │
    ▼
┌──────────┐
│ Harness  │  主Agent（primary）- 统筹编排调度
│  编排者   │
└────┬─────┘
     │ Task 工具调度
     ├──▶ @sa-agent     → 需求理解 → 需求拆解 → PRD/Spec编写
     ├──▶ @design-agent → 存量解析 → 组件规格 → 架构设计 → 接口设计 → 结构设计 → 流程设计
     ├──▶ @coder-agent  → 阅读设计 → 编码实现
      ├──▶ @review-agent → 阅读设计和代码 → 一致性检查 → 漏洞检测 → 架构合规 → 分层规范
      ├──▶ @tester-agent → 测试用例生成 → 运行用例 → 修代码 → 检查
      └──▶ @cmo-agent    → 入库前门禁 → 版本构建 → 发布部署 → 发布验证
```

## 核心特性

### 1. 全生命周期管理
```
需求分析 → 规格编写 → 架构设计 → 编码实现 → 一致性审查 → 质量验证 → 配置管理与发布 → 交付
```

### 2. 规范驱动开发(SDD)融合
- Spec与Design严格分离："构建什么" vs "如何构建"
- EARS格式验收条件
- 需求→设计→代码→测试全链路可追溯

### 3. 质量左移
- 编码后自动触发一致性审查（Review Agent）
- 代码-设计一致性验证
- 安全漏洞检测和架构合规检查

### 4. 闭环修复
- Tester Agent发现问题时可直接修复代码
- 修复后重新验证，确保问题解决

### 5. 人工审核模式选择
支持两种运行模式，在流程开始时由用户选择：
- **需要人工审核**：每个阶段出口门禁通过后暂停，等待用户确认后进入下一阶段（默认模式）
- **无需人工审核**：仅执行系统门禁检查，各阶段自动流转（适用于自动化流水线或无人值守场景）

### 6. 固定技能加载
每个Agent根据任务类型和项目语言，固定加载对应的技能：
- **Coder**：根据语言加载编码规范技能（Python/Java/C++/C）
- **Review**：根据语言加载代码质量技能（可读性、可维护性、安全性）
- **Design**：根据设计阶段加载设计技能（架构设计、详细设计、接口设计）
- **Tester**：加载测试验证技能（已整合测试用例生成）

## Agent配置

| Agent | 角色 | 模式 | 职责 | Skill |
|-------|------|------|------|-------|
| harness-agent | Harness Agent | primary | 统筹编排调度子Agent | harness-orchestrator |
| sa-agent | SA Agent（需求分析师） | subagent | 需求理解→拆解→PRD/Spec编写 | analyst-prd |
| design-agent | Design Agent（设计工程师） | subagent | 存量解析→组件规格→架构设计→接口/结构/流程设计 | design-engineering |
| coder-agent | Coder Agent（编码工程师） | subagent | 阅读设计→编码实现 | coding-skill |
| review-agent | Review Agent（审查工程师） | subagent | 一致性检查→漏洞检测→架构合规→分层规范 | review-check |
| tester-agent | Tester Agent（测试工程师） | subagent | 测试用例生成→运行用例→修代码→检查 | test-verify |
| cmo-agent | CMO Agent（配置管理工程师） | subagent | 入库前门禁→版本构建→发布部署→发布验证 | config-management |

## 工作流程

Harness Agent编排顺序：

1. **需求分析** — Harness拆解需求，创建任务列表
2. **PRD/Spec编写** — 调用 `@sa-agent`，产出 `docs/prd.md` + `.sd/specs/{feature}/spec.md`
3. **系统设计** — 调用 `@design-agent`，产出 `docs/design.md` + `.sd/specs/{feature}/design.md` + `.sd/specs/{feature}/tasks.md`
4. **编码实现** — 调用 `@coder-agent`，产出源代码和测试代码
5. **一致性审查** — 调用 `@review-agent`，产出 `docs/review-report.md`
6. **质量验证** — 调用 `@tester-agent`，产出 `docs/qa-report.md`（含代码修复）
7. **配置管理与发布** — 调用 `@cmo-agent`，产出 `docs/deploy-report.md`（入库前门禁→版本构建→发布部署→发布验证）
8. **交付汇总** — Harness汇总产出物，向用户报告

## 产出物

完整流程执行后产出：

```
docs/
├── prd.md              # 产品需求文档（sa-agent产出）
├── design.md           # 技术设计文档（design-agent产出）
├── review-report.md    # 一致性审查报告（review-agent产出）
├── qa-report.md        # 质量验证报告（tester-agent产出）
└── deploy-report.md    # 部署报告（cmo-agent产出）

.sd/
└── specs/
    └── {feature-name}/
        ├── spec.md      # 需求规格（sa-agent产出）
        ├── design.md    # 技术设计（design-agent产出）
        └── tasks.md     # 实现任务（design-agent产出）

src/                    # 源代码（coder-agent产出）
tests/                  # 测试代码（coder-agent产出）
```

## 安装部署

### ⚠️ 重要：配置 model ID

`opencode.json` 中的 `model` 字段默认值为 `"glm-5.1"`，**你必须将其替换为你自己 opencode 配置中实际使用的 model ID**。

查看你的 model ID：打开 `~/.config/opencode/opencode.json`（全局配置），找到 `provider` 下你使用的模型名称，格式通常为 `provider名/model名`（如 `bailian-glm/glm-5.1`）或直接是 `model名`（如 `glm-5.1`）。

将 `opencode.json` 中所有 7 个 agent 的 `model` 字段替换为你的实际 model ID。

### 方式一：项目级安装（推荐）

1. 复制 `agents/` 目录到项目 `.opencode/agents/`：
   ```
   你的项目/.opencode/agents/
   ├── harness-agent.md
   ├── sa-agent.md
   ├── design-agent.md
   ├── coder-agent.md
   ├── review-agent.md
   ├── tester-agent.md
   └── cmo-agent.md
   ```

2. 复制 `skills/` 目录到项目 `.opencode/skills/`：
   ```
   你的项目/.opencode/skills/
   ├── harness-orchestrator/SKILL.md
   ├── analyst-prd/SKILL.md
   ├── design-engineering/
   │   ├── SKILL.md
   │   ├── architecture-design/
   │   ├── detailed-design/
   │   └── interface-design/
   ├── coding-skill/
   │   ├── SKILL.md
   │   ├── python/
   │   ├── java/
   │   ├── cpp/
   │   └── c/
   ├── review-check/SKILL.md
   ├── test-verify/
   │   ├── SKILL.md
   │   ├── assets/
   │   └── references/
   ├── config-management/SKILL.md
   └── templates/
   ```

3. 复制 `opencode.json` 到项目根目录，**并修改 model ID**

### 方式二：全局安装

1. 复制 Agent 定义到全局目录：
   - Windows: `%USERPROFILE%\.config\opencode\agents\`
   - Linux/macOS: `~/.config/opencode/agents/`

2. 复制 Skills 定义到全局目录：
   - Windows: `%USERPROFILE%\.config\opencode\skills\`
   - Linux/macOS: `~/.config/opencode/skills/`

3. 将 `opencode.json` 中的 agent 配置合并到全局 `~/.config/opencode/opencode.json`，**并修改 model ID**

### ⚠️ 注意事项

- **permission 必须使用扁平格式**（如 `"bash": "allow"`），不要使用嵌套格式（如 `"bash": {"git *": "allow", "*": "ask"}`），OpenCode Desktop 不支持嵌套 permission，会导致子 Agent 启动失败
- **agent md 文件只保留 description 和 system prompt**，不要在 frontmatter 中重复配置 model/permission/mode 等字段，这些由 opencode.json 统一控制
- **model ID 必须与你的 provider 配置匹配**，否则子 Agent 无法连接 LLM

## 使用方式

### 启动Harness Agent
在OpenCode中按 `Tab` 键切换到 `harness-agent` Agent

### 直接使用子Agent
```
@sa-agent 帮我写一个用户登录功能的PRD
@design-agent 分析当前项目代码结构并设计用户认证架构
@coder-agent 实现用户登录接口
@review-agent 检查代码与设计的一致性
@tester-agent 运行测试并检查质量
@cmo-agent 执行版本构建和部署发布
```

### 触发完整流程
向Harness提出需求：
```
我需要开发一个用户注册登录功能，支持手机号和邮箱两种注册方式
```
Harness自动编排所有子Agent完成全流程。

### 可用技能列表

#### 编码规范技能（支持Python/Java/C++/C）
- **基础规范**：命名、格式、注释、异常处理、并发
- **安全规范**：输入校验、加密安全、序列化安全
- **Clean Code特征**：可读性、可维护性、可靠性、可测试性、性能、可移植性

#### 设计技能
- **design-engineering**：综合设计技能，整合架构设计、详细设计、接口设计
  - `architecture-design/`：分层架构、模块划分、技术选型、依赖管理
  - `detailed-design/`：数据库设计、业务流程设计、安全设计
  - `interface-design/`：RESTful API设计、入参出参、错误码体系

#### 测试技能
- **test-verify**：质量验证，测试用例生成，修复代码问题，验证修复结果

#### 配置管理技能
- **config-management**：入库前门禁检查、版本构建、发布部署、发布验证

### 技能加载示例
```
# Coder编码时自动加载编码规范技能
@coder-agent 实现用户登录接口（Python项目）
# 加载：coding-skill（自动检测项目语言并加载对应编码规范）

# Review审查时自动加载代码质量技能
@review-agent 检查用户登录接口代码
# 加载：review-check（自动检测项目语言并加载对应代码质量规范）

# Design设计时自动加载设计技能
@design-agent 设计用户认证系统架构
# 加载：design-engineering（整合所有设计子规范）

# Tester测试时自动加载测试技能
@tester-agent 生成用户登录功能的测试用例
# 加载：test-verify（已整合测试用例生成功能）

# CMO部署时自动加载配置管理技能
@cmo-agent 构建版本并部署到生产环境
# 加载：config-management（入库前门禁→版本构建→发布部署→发布验证）
```

## 模型配置建议

| Agent | 推荐模型类型 | 原因 |
|-------|-------------|------|
| harness-agent | 强推理模型 | 需理解需求、编排任务 |
| sa-agent | 强推理模型 | 需深度理解需求并结构化输出 |
| design-agent | 强推理模型 | 需分析代码、设计架构、SDD流程 |
| coder-agent | 代码能力强模型 | 需高质量代码输出 |
| review-agent | 严谨模型 | 需细致审查、不遗漏问题 |
| tester-agent | 代码+测试模型 | 需运行测试并修复代码 |
| cmo-agent | 严谨模型 | 需细致检查配置、构建、部署步骤 |
