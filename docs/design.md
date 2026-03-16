### OpenClaw Skills 开源产品整体架构设计

#### 1. 仓库命名与分层结构 (GitHub Repository Hierarchy)

建议仓库命名：`openclaw-skills-universe`

```text
/openclaw-skills-universe
├── /docs                       # 产品设计文档、API 标准、贡献指南
├── /core-protocol              # 核心协议：定义一个 Skill 的输入/输出标准 (基于 MCP 协议)
├── /sdk                        # 开发者工具包：快速开发、测试、调试新 Skill
│
├── /skills                     # 核心技能库 (按等级分层)
│   ├── /L1-atomic              # 原子执行层：定位、压缩、点击、登录态管理
│   ├── /L2-interactive         # 交互演进层：弹窗处理、自我修复、人机协作网关
│   ├── /L3-cognitive           # 认知规划层：任务拆解、Token 路由、长期记忆
│   └── /L4-ecosystem           # 生态协同层：多 Agent 编排、跨应用桥接、安全审计
│
├── /security-guardrails        # 安全护栏：所有的 Skill 必须通过的安全扫描插件
├── /examples-workflows         # 场景化工作流：将多个 Skill 组合成的 SOP (如自动报销流)
└── /registry                   # 技能注册表：元数据管理，方便 Agent 动态检索
```

---

#### 2. 核心技能项集成明细表 (Roadmap 视图)

| 等级 | 模块分类 | **开源 Skill 模块名称 (Skills Product)** | 功能定位与集成价值 |
| :--- | :--- | :--- | :--- |
| **L1** | **基础感知** | `skill-vision-locator` | 提供高精度的像素级坐标对齐，解决“点不准”痛点。 |
| | | `skill-dom-cleaner` | 专为 VLM 设计的网页降噪，节省 80% Token。 |
| | | `skill-session-keeper` | 支持多平台 Cookie 注入与持久化，跳过登录环节。 |
| **L2** | **稳健交互** | `skill-popup-adblocker` | 视觉驱动的智能弹窗识别与拦截，保障流程不中断。 |
| | | `skill-retry-loop` | 包含指数退避算法的视觉重试机制，提升执行稳定性。 |
| | | `skill-captcha-resolver` | 封装外部识别能力，解决自动化中最硬的骨头。 |
| **L3** | **高级规划** | `skill-smart-planner` | 将自然语言目标转化为 OpenClaw 动作序列的微调模型插件。 |
| | | `skill-token-optimizer` | 自动切换大小模型的智能路由，极致优化运行成本。 |
| | | `skill-memory-graph` | 存储用户偏好和历史网页结构的知识图谱。 |
| **L4** | **安全与协同** | `skill-security-guardian` | 动作执行前的实时风险分级与阻断系统（核心安全件）。 |
| | | `skill-workflow-bridge` | 将浏览器操作与 Slack/Discord/飞书 API 联动的桥梁。 |
| | | `skill-action-recorder` | 全程黑匣子记录，生成可供审计的操作轨迹视频与日志。 |

---

#### 3. 技能标准定义 (Skill Manifest Standard)

为了让所有的开源贡献遵循统一标准，每个 Skill 必须包含以下四个部分：

*   **`definition.json` (元数据)**：
    *   名称、版本、演进等级 (L1-L4)。
    *   所需的权限（如：是否需要读 Cookie，是否需要截图权限）。
*   **`executor.py` (执行逻辑)**：
    *   基于 OpenClaw 与 Playwright 的标准实现代码。
*   **`safety_policy.yaml` (安全策略)**：
    *   定义该技能的“动作边界”（例如：不允许点击带有 `delete` 字样的红色按钮）。
*   **`test_suite` (测试集)**：
    *   在模拟环境下的单元测试，确保 Skill 在不同分辨率下的鲁棒性。

---

#### 4. 关键产品特性设计 (PM 视角)

##### ① **插件化 (Plug-and-Play)**
用户可以像安装 npm 包一样安装技能：
`openclaw install skill-dom-cleaner`
Agent 在启动时会自动加载已安装的技能并更新其 `Toolset`。

##### ② **视觉调试器 (Visual Skill Debugger)**
随仓库附带一个可视化工具，显示 Agent 正在调用哪个 Skill，并在屏幕上实时叠加显示坐标定位、DOM 压缩后的预览、以及安全护栏的判定结果。

##### ③ **安全审计中枢 (Safety Audit Hub)**
所有从 GitHub 下载的 Skill 在执行前，都会经过 `Skill_Security_Guard` 的静态扫描，识别是否存在潜在的恶意代码注入或数据外泄风险。

##### ④ **Skill 组合器 (Flow Builder)**
提供一个简单的 JSON 或 YAML 配置方式，让非技术用户能将 L1-L4 的技能串联：
*“使用 `session-keeper` 登录 -> 使用 `dom-cleaner` 读数 -> 使用 `workflow-bridge` 发到钉钉。”*

---

#### 5. 社区增长策略 (Open Source Growth)

1.  **Skill 贡献竞赛：** 悬赏征集对于常用复杂系统（如 SAP、Salesforce、Jira）的专属 Skill 模块。
2.  **兼容 MCP 协议：** 确保所有的 OpenClaw Skills 兼容 Anthropic 提出的 **Model Context Protocol**，这意味着这些技能不仅能在 OpenClaw 用，还能被 Claude Desktop 等其他 Agent 调用，极大扩大影响力。
3.  **Benchmark 榜单：** 发布 **"OpenClaw Web-Navigation Benchmark"**，用这些 Skills 去跑通全球 Top 500 网站的自动化流程，证明其实用性。
