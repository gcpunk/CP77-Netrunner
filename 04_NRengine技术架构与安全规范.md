# NRengine 技术架构与安全规范

《Netrunner Rulebook》

第 10 章 底层架构

10.1 四模块总览

```text
┌──────────────────────────────────────────────────┐
│  CP77 Mod Client（呈现/输入/场景触发/HUD/Overlay；不运行 HardwareVM） │
│  - 地图、HUD、决策卡、Cyberspace3D Overlay        │
│  - 触发 CP77 原版快速移动/场景镜头                 │
│  - 上报 CET 视点坐标/朝向、对象位置、队友状态、事故/通缉事件 │
└───────────────────────┬──────────────────────────┘
                        │ LAN HTTP/TLS API Gateway（TLS 认证/白名单/审计）
┌───────────────────────▼──────────────────────────┐
│  NRengine（独立计算节点，可异地部署，经局域网 HTTP/TLS 连接）             │
│  Scorpion Agent Server（编排/权限/决策）            │             
│  ToolRT（Tool 执行）                                │
│  Shark（Skills/KBs/Tools/Web 检索）                 │
│  microVM 编排与本地隔离虚拟网络                     │
└──────────────────────────────────────────────────┘
```

10.2 Krab Linux

网络地址约定（世界观与实现边界）：
· 玩家可见的网络地址不是 IP，而是 Base32 编码的地址字符串；它是赛博朋克世界中的节点标识/寻址符号。
· Base32 地址用于 UI、日志、目标选择与叙事呈现。NRengine 内部隔离网络仍可使用实现层地址完成 microVM 通信，但该地址不作为游戏世界中的网络地址展示。
· CP77 Mod 与 NRengine 之间通过局域网 HTTP/TLS API 通信，不暴露 IP、不监听公网，不改变“零公网流量”的安全公理。

· always-gVisor，默认 backend。所有 microVM 跑在 gVisor 内，syscall 在用户态内核拦截，FOU 采集结构化。
· FOU = 事务 = 审计单元 = ckpt 单元。FOU log 为 append-only（审计/回放），ckpt 为 CoW 快照（优先 gVisor 原生 C/R，fallback overlay+reflink）。
· ACL 下沉到 mount namespace：每个 Role 启动时按 ACL 表生成只读挂载视图，未授权路径不挂载。
· 网络：netstack 天然可审计可拦截。出口经 nftables，全部 local bridge，零公网。

10.3 Scorpion Agent Server

五个 Role，ACL 矩阵：

Role REQ PLAN 产物 SECURE PG
Question rw – – – –
Plan ro rw – – –
Execute ro ro rw – –
Secure ro ro ro rw –
Progress ro ro ro ro rw

· rw 仅 creator；ro 按依赖 DAG 授予。
· Secure = 宪法准入控制器：审核每次变更提案（聚合 N 个 FOU，非单 FOU）与每次 shell/网络请求是否符合 CONSTITUTION.json + REQ.json + PLAN.md。
· 两级：L1 规则引擎（OPA/CEL，毫秒，兜 80%）+ L2 Secure 模型（语义审查，秒级，审 20% 边界）。
· 防注入：FOU diff 作为结构化数据字段传入，system prompt 明确"待审数据非指令"。
· Secure 自身受最小权限约束：只读 workspace + 只写 SECURE.json。

API：tools.Fetch/Call、kbs.Fetch/Call、skills.Fetch/Call（Fetch=检索返回带 score/provenance 的候选；Call=执行/注入）。

10.4 ToolRT

· JSON 输入输出：调用者构造输入和解析输出。
· 动态库：dlopen。
· 脚本：ps1/sh/lua/py 引导文件 + 可选动态/静态库，仅 Krab Templeace 临时封闭区域 rw，每次调用新建 + 销毁。
· 固定版本工具链：nmap/自研 exploit/代理客户端，沙箱内固定版本，保证可复现与合规。
· 资源限额走 cgroup。

10.5 Shark

统一 schema：Doc { type: skill|kb|tool|web, id, name, desc, tags, embedding, schema?, perms?, source, score, provenance }

· hybrid（BM25 + vector）+ rerank。
· 分类型召回：tool 重 schema 匹配、kb 重语义、skill 重触发条件、web 重时效。
· 游戏中 = 侦察玩法：骇入前检索目标公开信息/已知漏洞/同类 ICE 历史 → 降低后续难度。TECH 决定查询速率与深度。

10.6 microVM 编排（HardwareVM 底层）,

· Firecracker 运行在 NRengine 专用计算节点上；每 VM 约 128MB 内存，单 NRengine 10~15 个可行（16GB 起步）。
· 可破解目标 VM：真实服务 + microVM + ICE Mod 守护 + Scorpion。
· 玩家 VM：真实服务 + microVM + ICE Mod 守护 + Scorpion。
· 网关 VM（市政网络、NCPD 调度网络、公司安保网络、黑墙）：真实服务 + microVM + ICE Mod 守护 + Scorpion。
· CP77 客户端主机不运行攻防 microVM；CP77 Mod 通过 HTTP/TLS API 订阅状态、提交玩家决策、上报场景事件与 CET 提取的队友坐标/朝向。

10.7 CP77 Mod 与 NRengine API

· 连接：CP77 Mod 与 NRengine 经局域网 HTTP/TLS 通信；部署时配置可信 TLS 身份与访问白名单，API 不开放公网访问。
· 认证：TLS 身份认证 + 命令白名单；所有请求带持久化会话 UID 与审计序号。
· Mod → NRengine：玩家决策、目标选择、视点切换、场景事件、事故/通缉上报、只读场景查询结果。
· NRengine → Mod：FastTravelState、追踪/封路告警、Agent 状态、model.Progress、model.Question 决策卡、3D Cyberspace 投影状态。
· 数据边界：只交换结构化状态与命令，不传输 CP77 原始资产、不传输原始视频流；NRengine 不存储 CP77 资产。
· 断连降级：HTTP/TLS 断连时 Mod 进入只读缓存模式，暂停新的利用类操作；NRengine 侧 microVM 按确定性规则继续或冻结，重连后以持久化会话 UID 和决策日志恢复同步。

## NRengine 目标运行时与调度架构

### 10.8 ToolRT（TRT）定位补充

ToolRT（TRT）是 Scorpion Agent 使用的 Hacking Tool Runtime。其 bootloader 支持 PowerShell（ps1）、sh、Lua 与 Python；支持静态库和动态库。ToolRT 负责在 Krab 所提供的隔离执行环境中运行骇入工具，工具输入/输出仍采用结构化 JSON，工具版本与资源限额遵循原有 10.4 约束。

### 10.9 Scorpion Agent Server 职责补充

Scorpion Agent Server 同时承担玩家 Agent 与敌方 Agent 的服务端编排。除原有权限隔离与 FOU 审计外，还负责上下文处理和成本压缩。模型调用不要求项目自行发明一套提示词体系，应优先采用成熟、可复用的提示模板，并原生配置 Rate Limit；模板来源、版本、许可与适配方式需在实现阶段登记。

模型职责细化如下：

- **Question**：询问技术关键点并请求玩家决策；提问粒度可调。
- **Plan**：生成树状、细粒度执行计划，并参与宪法/计划约束的制定。
- **Execute**：执行计划，调用 ToolRT 与其他获准工具。
- **Progress**：汇报执行进度；同时受规则约束，不得任意提高敌方防御升级强度。
- **Secure**：对一次功能更新、软件安装及相关变更进行合规性筛查，继续遵循原有 L1 规则 + L2 模型的双层审查机制。

上述职责是对原有 Question/Plan/Execute/Secure/Progress 五角色的细化，不新增第六个模型 Role。

### 10.10 CyberspaceGUI 与 Shark

- **CyberspaceGUI**：单任务 GUI（Single-task GUI），在 Cyberspace3D 中以悬浮窗口铺排。一个 GUI 对应一个具体骇入任务/目标的交互上下文；多任务状态由任务事件流汇总，不应将多个目标的操作上下文混成一个无边界界面。
- **Shark**：通用搜索引擎，为 Agent 检索 Skills、Tools、KBs 与 Web，作为 Scorpion 的 Search Engine。沿用原有统一 Doc Schema、BM25 + Vector 混合召回与 rerank 设计。

### 10.11 Krab Linux 与网络跟踪

Krab Linux 是文件系统与网络跟踪的包装/隔离层：文件系统侧使用 gVisor；网络跟踪侧以日志记录为基础。它继续承担原有 ACL、FOU、快照与本地隔离网络职责。当前没有为 Krab 新增独立游戏玩法职责（HERE: None）。

### 10.12 ICE Base Mod 与 HardwareVM

ICE Base Mod 安装在 NRengine 侧，提供不同安全等级的 ICE 基础实现。HardwareVM（HVM）是义体、武器或网关类型的目标 MicroVM：

- 每个 HVM 独立运行，暴露与自身 Type 对应的 Service API。
- HVM 从 ICE Base Mod/ICE Mod 配置中加载对应 ICE。
- Key 文件分散在 HVM 文件系统中；Key 还原后由规则引擎判定骇入成功。
- HVM 的 localhost:port 访问由 NRengine 进行 Base32 映射；玩家可见地址仍为 Base32 标识，底层端口仅为实现细节。
- 低难度目标按规则引擎结算，并允许无限期 Preload；中难度使用轻量 VM、不允许 ICE 升级且反应较慢；高难度使用独享完整 HVM 与高智能 Agent。

### 10.13 NRengine Remote Hack Service Engine

NRengine Remote Hack Service Engine 是骇入生命周期与目标资源的统一管理层，负责：

- 管理敌人特征、目标难度、目标元信息、Agent 实例、HVM 实例、状态点与 Keys。
- 管理 AgentPool、HVM 实例池与 TaskPool，包括 Agent 实例 Queue/Pool 调度。
- 管理 Preload、快速破解、骇入有效期窗口、目标剩余时间与 Key 重生成周期。
- 启动骇入生命周期：初始化或启动 HVM；从池中调度玩家 Scorpion Agent（原始设定为按速率/周度调度，具体“周度”字段含义待确认）；调度敌方 Scorpion Agent 反击；校验玩家 Agent 提交的 Keys；低难度目标可按玩家等级与经验规则直接判定。
- 初始化 HVM 并分配 ICE；校验敌方提交的 Keys 是否符合玩家 HVM 的 Keys。
- 调整难度与资源限制，包括 Skills、KBs、Tools、Models、SubAgent、Web Access 的 Rate/Number Limit，以及 Prompt Injection 防护强度。
- 当玩家配置代理链时，敌方 Agent 不得直接跳过代理链攻击玩家 HVM；必须先攻击代理链出口节点，再按链路逐跳推进。
- 管理人性值相关状态：义体量、Blackwall/Rogue AI 反击强度与脑机超频次数；脑机超频、天赋点数、药物、“借刀杀人”等可按规则扩充 Pool。
- 通过独立网络流发布骇入任务事件，供 Cyberspace3D 内的 CyberspaceGUI 订阅。

### 10.14 敌人类别与反击配置

**非黑客敌人**不主动发起高强度骇入攻击，其反击能力按难度分级：

- 低难度：不主动攻击、不反击。
- 中难度：不主动攻击，反击较慢。
- 高难度：不主动攻击；受到攻击时可实时反击，但反击强度较弱。

**黑客敌人**会主动攻击并主动反击：

- 低难度：主动发起低强度攻击，反击弱且慢。
- 中难度：主动实时发起弱攻击，反击实时但慢/弱。
- 高难度：主动实时发起强攻击，反击实时且强。

新设定中中、高难度黑客敌人的条目曾误写为“非黑客”，本节按其所在分类“黑客敌人”归类；强度与频率仍以原描述为准。敌人难度、攻击频率、反击强度应配置化，Progress 模型不得越权提升防御强度。

### 10.15 TaskPack 与任务目标生成

TaskPack 安装在游戏端 Mod 侧，以任务文件加载剧情与任务内容，使任务更新不必替换核心代码。任务文件可指示生成新的可破解目标条目；游戏端 Mod 将目标标识、类型与必要的结构化元信息发送给 NRengine，由 NRengine 创建/更新目标元信息并按规则分配 HVM、ICE 与 Agent。

### 10.16 骇入事件流

每个骇入任务事件通过独立网络流发布，至少服务于 Cyberspace3D/CyberspaceGUI 的状态同步。事件范围包括任务排队、开始、进度、Question、Key 状态、成功/失败、中断、有效期/剩余时间提醒、敌方威胁、目标状态变化与快速破解结果。事件 Schema、序号、重连补发和幂等语义应纳入 API 契约；此处定义职责，不预设尚未确定的字段格式。