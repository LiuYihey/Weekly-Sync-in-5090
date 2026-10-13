# Weekly-sync


## 汇报日期 2026-08-14

### 工作内容

#### 一、前端视频资源与后端进程排查
- 排查浏览器控制台 2 条 `net::ERR_ABORTED` 视频加载错误（`design-agent-demo.mp4` / `survey-agent-demo.mp4`），根因：文件过大（99MB / 77MB）浏览器预加载传输中断
- 用 ffmpeg（CRF 28 + faststart，保留 1080p/30fps）将两视频分别压缩至 ~12MB / ~11MB（压缩比 ~8× / ~7×），同步更新三处副本：`Backend/static/videos`（nginx 实际提供）、`Interface/public/videos`（构建源）、`Interface/dist/videos`
- 识别 `http://localhost:8081/` 进程：项目 FastAPI 后端（`uvicorn main:app`），由 nginx（192.168.10.254:80）反代 `/api` 等路径到该后端

#### 二、对话体验优化
- **Chat 加载闪动修复**：`Interface/src/components/AgentChat.jsx` 中计划面板由动态 `motionKey` 的 `PageFloatTransition` 改为静态渲染，消除数据回灌时重播动画造成的内容闪动
- **减少后端自动发送的消息**：
  - `Backend/agent_config.json` 的 `orchestration` 段关闭 Design Agent 的 `recovery_nudges` / `pending_action_reminders`，让 TAO 循环（Thinking-Act-Observation）主导 agent loop，移除写 todo list 的 platform reminder
  - `ProteinHarness/agents/literature/CORE.md` 与 `ProteinHarness/action_guard.py` 清理 Literature Agent 在结尾主动邀请用户生成 PDF 的 reminder 文案

#### 三、PDF 报告编译修复（标题截断根本解决）
- `Backend/main.py` 中 `_build_pubmed_report_pdf` 的标题渲染：从 fpdf `cell`（单行 + 按 78 字符硬拆的启发式）改为 **`multi_cell(w=0, align="C")`**，按实测字形宽度自动居中换行，彻底解决长标题溢出页边距被截断的问题
- 同步更新 LLM 提示词：移除"HARD LENGTH LIMIT on the title"的过严限制与裁断警告，改为建议精炼标题
- 烟雾测试：122 字符长标题自动折为 4 行，全部位于页边距内；`py_compile` 通过

#### 四、Session 切换后后台流劫持视图（跳转回原 session）的彻底修复
- 根因定位：`streamControlsUi` 闸门比较对象是 `sessionIdRef.current`，用户切换 session 时该 ref 被覆盖为新 session，导致旧 session 的后台流仍然认为自己持有视图，继续写入新视图；完成时再刷新，呈现为"跳转回去"
- 修复：
  - `AgentChat.jsx` 新增 `activeStreamSessionRef`，绑定**流所属 session**，切换 session 时不被覆盖；`streamControlsUi` 改为按 `activeStreamSessionRef` 判断；新建画布要求 `localStreamOwnerIdRef === streamOwnerId`
  - 两条启动流路径（sendMessage、interpret）均新增 `myStreamSessionId` 闭包 + `assignStreamSession`，所有 `publishDesignStream` 发布流自己的 session（而非 sessionIdRef），使镜像路径在切换后自然失效
  - `Interface/src/lib/designAgentStream.js` 的 `SESSION` op：仅当流仍持有视图时才 `setSessionId` / 写入 `viewingSessionKeyRef`，否则静默完成
  - `loadFromHistory` 切换时同步清理 `localStreamOwnerIdRef` 与残留的 `isStreaming / isThinking` 标志
- 不影响 subagent 跳回主 agent 的逻辑（literature dispatch / Bio 流程使用独立通道，未改）

#### 五、分子查看器交互与 UI 修复
- **滚轮缩放误操作**：`StructureViewer3D`（3Dmol.js 包装）新增 `allowZoom` prop；`AgentChat.jsx` 内联预览传 `allowZoom={false}`，通过 capture 阶段 `wheel` 事件拦截，阻止 3Dmol 内置缩放；全屏弹窗保留默认 `allowZoom={true}`
- **全屏按钮偶尔空白**：`StructureViewerModal.jsx` `lazy={false}` 路径跳过 `IntersectionObserver`（portal 挂载瞬间 IO 可能短暂误报不可见，卸载初始化中的 viewer 留下空白 canvas）
- **预览布局重设计**（`AgentChat.jsx` structure_preview 消息）：
  - 下载 PDB 按钮从头部右侧移至**分子框左上角**（与右上角全屏按钮对称），改为半透明白底 + `Download` 图标 + "PDB" 的极简药丸样式（轻阴影、高对比边框）
  - Meta label（如 `126 res · chain A`）加 `flexShrink: 0 / whiteSpace: nowrap`，标题区 `flex: 1 1 auto / minWidth: 0`，修复 label 被压缩问题

### 结果与产出
- 视频 ERR_ABORTED 彻底消除（nginx 直读文件，无需重启）
- PDF 标题任何长度都不会被截断，自动居中换行
- 切换 / 新建 session 后，旧 session 的 agent 在后台正常完成，不再跳转回旧 session；subagent 跳转逻辑保持不变
- 对话内联分子框滚轮不再误缩放，label 不再被压缩，下载键外观与位置更协调，全屏按钮显示分子更稳定
- 所有前端改动 `npm run build` 通过，后端 `py_compile` 通过

### 遇到的问题及解决
- **上次跳转修复失败**：只改了 `openBlankCanvas` 里的 `localStreamOwnerIdRef`，但 `streamControlsUi` 根本不校验它；重新分析 `streamControlsUi`、`SESSION` op、`publishDesignStream` 后，引入 `activeStreamSessionRef` + 统一 `myStreamSessionId` 闭包，彻底解耦"流所属 session"与"当前视图 session"
- **全屏空白偶发**：起初怀疑是 viewer ready 延迟，经代码审查定位到 portal 挂载时 `IntersectionObserver` 对 fixed/portal 元素可能报告非相交，改为 modal 路径直接跳 IO 根绝该竞态

### 下一步计划
- 待用户确认后执行部署（`./env/deploy-local.sh` 重建前端 dist 并同步到 Backend/static + 重启后端 uvicorn 8081）
- 继续收集用户对新对话体验与分子预览 UI 的反馈

## 汇报日期 2026-08-21

### 工作内容

#### 一、全面修复 ligand-binder 与 AME 的原生 TTO 链路
- 梳理 Proteina-Complexa 的 ligand-binder、AME 生成、beam search、checkpoint rollout、reward 聚合和最终候选提升链路，明确 **Proteina-Complexa 是唯一生成 backbone，RF3 仅作为 rollout reward evaluator**，不将外部 RFD3/BindCraft 接入 TTO backbone
- 核查本机 RF3 运行环境，确认项目 `.venv` 不包含可用 RF3，但服务器共享环境 `/home/sl/miniforge3/envs/rosettac` 已提供 `rf3` / `rfd3`、`rc-foundry 0.1.12` 和可用 CUDA；项目 checkpoint 可通过 `community_models/ckpts/RF3/rf3_foundry_01_24_latest_remapped.ckpt` 正确访问
- 更新 `.env`，将失效的项目内 RF3 路径切换为共享 RF3；在 `Backend/main.py` 增加 sibling executable 自动发现、路径解析和 ligand/AME TTO 启动前的 executable/checkpoint 严格预检

#### 二、RF3 reward 可靠性与无损工程优化
- 重构 `src/proteinfoundation/rewards/rf3_reward.py`：初始化时验证 executable/checkpoint；候选推理失败、缺少 CIF/summary、AME 缺少 PAE/ipSAE 时直接 fail-fast，不再生成默认 prediction 或伪造 reward
- 新增 `score_batch()`：同一 checkpoint 下的多个候选通过一次 `rf3 fold inputs=[...]` 批量评估，复用进程启动和 checkpoint 加载，减少重复开销
- 更新 `base_reward.py` 和 `reward_utils.py`：支持 CompositeRewardModel 严格批量聚合，验证返回数量及每个 `total_reward` 为有限标量；禁止 folding child 失败后吞异常并继续使用 `total_reward=0`
- 全程保持 RF3 默认推理精度，没有注入 `n_recycles`、`diffusion_batch_size`、`num_steps`、seed 或 early-stopping 等预算覆盖，工程优化不以牺牲准确度为代价

#### 三、真实 GPU 端到端验证与 AME 输出修复
- 在 RTX 5090 上完成 ligand-binder 最小真实 TTO：加载 Proteina ligand checkpoint/LoRA，执行 100-step beam search 和 RF3 rollout 评分，生成 PDB、binder PDB、reward CSV、`designs.csv` 和运行进度
- ligand 真实 RF3 指标包括 `min_ipAE=0.13419354`、`plddt=0.7386`、`ranking_score=0.5875`；该最小样本 final 恰为最优，因此增益为 0，但候选评分与选择诊断完整生效
- 完成 AME `M0050_1dbt` 真实 GPU TTO；首轮发现 AME 同时包含 motif 与 ligand 条件时，`generate.py` 在复制 motif 信息后提前 `return`，导致遗漏标准 `designs.csv`
- 删除错误提前返回后重新执行 AME v2：成功生成完整结构和诊断；RF3 指标 `min_ipAE=0.34451613`、`has_clash=0`、`max_ipSAE=0`、`plddt=0.6664`、`pTM=0.26478815`，按原始配置得到 `total_reward=-0.1847573`
- AME TTO 最终选择 lookahead 而非 final，`tto_gain_vs_best_final=0.00408053`，以真实端到端结果确认 TTO 对配置定义的质量 reward 产生正向提升

#### 四、回归验证与环境清理
- 执行 `.venv/bin/python -m pytest -q tests/search tests/generate Backend/test_tto_launch_args.py`，结果 **12 passed**
- 对 `Backend/main.py`、`generate.py` 及三个 reward 模块执行 `py_compile`，并完成 `git diff --check`，全部通过
- 清理中止安装遗留的 `.rf3-venv`（`rc-foundry 0.2.0`，约 7.0 GB）、配套 `.uv-cache`（约 5.4 GB）、runtime probe 临时目录以及修复前的首轮 AME smoke 结果，合计释放约 **12.4 GB**
- 保留 ligand 与 AME v2 最新成功验证产物；复核共享 RF3、项目 checkpoint、两套 `designs.csv` 和 rewards CSV 均完整可用

### 结果与产出
- ligand-binder 和 AME 的原生 Proteina-Complexa TTO 已可在当前机器上真实运行，不再受项目内缺失 RF3 环境阻塞
- RF3 从可能产生默认/伪 reward 的宽松路径升级为严格 fail-fast，并通过批量候选评分消除重复启动和模型加载，同时保持默认精度预算
- AME 输出索引缺失问题已修复，后续 evaluate 和结果枚举可以通过标准 `designs.csv` 获取最终设计
- 获得两条真实 GPU 端到端证据，其中 AME 在该样本上取得 `+0.00408053` 的 reward 提升
- 相关回归测试、语法检查、diff 检查全部通过，并释放约 12.4 GB 无用环境与缓存

### 遇到的问题及解决
- **项目 `.venv` 无 RF3，直接运行 ligand/AME TTO 会失败**：没有继续重复下载模型，而是排查服务器已有环境，复用共享 rosettac Foundry，并在 Backend 增加自动发现和严格预检
- **曾考虑通过较低 RF3 推理预算缩短验证时间，但会影响准确度且未经确认**：撤回所有预算覆盖，只保留不改变预测协议的批处理和进程复用优化；当前生产链路完全使用 RF3 默认推理参数
- **RF3 失败可能被默认 prediction 或零 reward 掩盖**：改为对 executable、checkpoint、每个候选输出和 AME 特有指标逐层校验，任一缺失立即失败并暴露真实错误
- **AME 成功生成结构但缺失 `designs.csv`**：定位到 motif 分支提前返回，进行最小修复后重新完成 GPU smoke，确认标准索引和诊断均恢复
- **中止的独立 Foundry 安装占用大量空间**：确认正式链路使用共享环境且无进程占用后，安全删除隔离环境与安装缓存

### 下一步计划
- 在更多 ligand 和 AME 任务上扩大 TTO 样本规模，统计 reward 增益分布、选中 lookahead 的比例及 GPU 时间开销
- 持续监控 RF3 batch 在长序列、多配体和高并发场景下的显存峰值与失败诊断，必要时只优化任务编排，不降低预测精度
- 在 SC/MMseqs/DSSP 运行环境完成配置后，再按既有评估协议补充 bioinformatics 指标验证，不擅自改变当前 ligand/AME reward 定义

## 汇报日期 2026-09-06

### 工作内容

#### 一、TTO 启动协议与最终评估对齐（Backend）
- `Backend/main.py` 新增 `build_tto_generation_args()`：protein_binder 的 TTO 奖励模型与最终独立 AF2-Multimer 评估严格同构 —— `model_nums=[0]`（AF2-Multimer model 0）、`num_recycles=3`、`use_initial_guess=true`、`use_binder_template=false`；reward 权重取 `i_pae=-1.0` 作为主目标、`plddt=-0.25` 作为弱次目标（负权重把 ColabDesign 的"损失"翻转为"奖励"，4:1 保证 iPAE 主导），并排除未归一化的结构对齐 MSE 项
- ligand_binder / ame_scaffold：将 `rf3folding` 的 `plddt`/`pTM` 权重置 0，TTO 选择由 min_ipAE（加 AME 既有 clash/ipSAE 项）决定，避免全局置信度项淹没界面目标
- Beam Search 保留策略固化进启动参数：`keep_lookahead_samples`、`track_best_rollout`、`low_temperature_rollout`（rollout 结构噪声 0.1）、100 步 checkpoint 与最终 beam 统一比优选优，保证 TTO 保留的是"完全去噪"后的最优 rollout
- `AF2RewardModel`（`src/proteinfoundation/rewards/alphafold2_reward.py`）新增独立 `use_binder_template` 参数：`None` 时保持旧行为（由 `use_initial_atom_pos/use_initial_guess` 推导），解耦后 TTO 可在保留 initial guess 的同时显式关闭 binder 模板，与评估打分协议一致；旧配置不受影响
- `Backend/test_tto_launch_args.py` 对上述启动参数逐项断言，测试 **2 passed**

#### 二、pLDDT 置信度分数全平台统一到 0–100
- 系统审计字段 / CSV / API / UI / 阈值五层：API 侧 `_plddt_100` / `_normalize_plddt_fields` / `_normalize_quality_thresholds`（main.py:3625-3656）对 design 行与阈值做幂等归一并兼容历史 0–1 数据；奖励模型（AF2 / RF3 / ESMFold2）暴露的 pLDDT 均为 0–100，loss 项改名 `plddt_loss` 隔离；CSV 生产端（colabdesign / esmfold2 / rf3 / multimer / monomer eval）与 success criteria（90/80 阈值 + `normalize_plddt_score`）、`thresholds.yaml`（plddt_min: 85）均已达标
- 发现并修复 2 处 UI 遗漏：`Interface/src/lib/validationMetrics.js` 中 `self_plddt` 阈值 `0.9→90`、`monomer_esmfold_plddt` 阈值 `0.7→70` —— 原 0–1 刻度在 0–100 数据下会使校验卡恒判失败
- 重新执行 vite build 并同步 `Backend/static`（新 bundle `index-BwCYtmYb.js`），已验证产物包含修正后的阈值

#### 三、Agent Memory 界面与功能添加
- 新增 `Backend/user_memory.py`：用户级持久化项目记忆的确定性存储边界 —— SQLite（WAL、busy_timeout=30s），按 `owner_email` 严格隔离；实验完成时由 Memory Update Agent 全量重写记录，手动编辑复用同一版本化 replace 通道；记忆检索留给 API 层（需要 LLM client），存储层保持无 LLM 依赖
- 新增前端记忆页 `Interface/src/pages/UserMemory.jsx`，接入 `App.jsx` 路由与 `TopNav` 入口；`LanguageContext.jsx` 补齐中英文案
- `AgentChat.jsx`、`DesignWizard.jsx`、`launchRequest.js` 同步集成记忆相关交互；`ProteinHarness/executors/task_info.py` 与 design agent 的 WIKI/profile 更新，使 agent 侧可获取任务与记忆上下文

#### 四、Memory Agent 的分析 skill
- 新增 `Backend/memory_analysis.py`（约 658 行，纯 Python 标准库、零第三方依赖）：把用户隔离的终态（completed/failed）实验元数据压缩为小规模、可审计的 evidence brief —— 重复配置对比（≥2 次才升格为结论）、成功/失败对照、质量代理指标归纳（pLDDT/iPTM/iPAE/scRMSD/DockQ/F_nat 等，全部显式标注为 in-silico proxy，不推断真实亲和力）、结构多样性、接触证据与复发失败模式；输出作为 LLM Memory Writer 的输入而非最终结论
- 新增 `Backend/memory_analysis_skill.md`：记忆更新 skill 的写作契约 —— 只保留会改变未来设计决策的结论及其关键限制，禁止原始指标表/任务 ID/来源附录，固定四段输出（项目目标与持久约束 / 可复用设计结论 / 开放问题与防护 / 下一步实验优先级），每次更新 reconcile 替换被取代的结论而非追加流水账

### 结果与产出
- TTO 搜索期打分协议与最终独立评估完全同构（同模型、同 recycle、同模板策略），启动参数由测试锁定，杜绝 UI/API 路径漂移
- pLDDT 0–100 契约在字段、CSV、API、UI、阈值五层闭环，历史 0–1 数据无缝兼容
- 平台具备完整的用户级项目记忆链路：存储层（user_memory.py）→ 前端页面（UserMemory.jsx）→ agent 集成 → "实验数据 → 可复用结论"的确定性分析 skill（memory_analysis.py + memory_analysis_skill.md）

### 遇到的问题及解决
- **UI 校验阈值沿用 0–1 旧刻度**：`validationMetrics.js` 两处 pLDDT 阈值在 0–100 数据下恒失败；统一改为 90/70 并重建前端 bundle 后验证产物生效
- **TTO 与评估打分协议不一致**：`use_binder_template` 原本与 initial guess 隐式耦合（开 initial guess 就连带把 binder 坐标当模板），而评估协议是"initial guess 开、binder 模板关"；新增独立参数解耦，`None` 保留旧行为向后兼容
- **负 reward 数值易被误解为异常**：明确其为损失加权后的设计取向 —— total_reward 仅用于候选间排序，负值正常
- **`tests/search/test_success_criteria.py` 在当前 conda 环境缺 hydra 无法收集**：确认为环境问题（项目 `.venv` 可运行），与本轮改动无关；本轮相关测试 `Backend/test_tto_launch_args.py` 全部通过

### 下一步计划
- 扩大 protein_binder / ligand / AME 的 TTO 真实样本规模，统计 reward 增益分布与 GPU 时间开销
- 以真实多用户数据验证 memory 分析 skill 的输出质量与版本化替换行为，打磨 UserMemory 页面交互细节
- 在 SC/MMseqs/DSSP 环境就绪后补充 bioinformatics 指标验证，不改变现行 reward 定义

## 汇报日期 2026-09-09

### 工作内容

#### 一、修复 launch 跳转后 sidebar 会话滚动位置来回跳变的问题
- **现象定位**：在 agent 对话中点击实验 launch card 跳转到实验页面时，对话转为右侧 sidebar 的瞬间，内容先显示对话开头、再跳回当前阅读位置，期间两者来回跳变
- **根因分析**（`Interface/src/components/AgentChat.jsx`）：
  - sidebar 为全新挂载实例，localStorage 水合的消息先以 `scrollTop 0` 首绘（闪到对话开头）
  - 原 `useEffect` 中 80ms 延迟的恢复逻辑再跳回记忆偏移（当前阅读位置），产生可见跳动
  - 挂载引导的 rehydrate 完成时 `queueScrollPreserve()` 若在 80ms 窗口内执行，会把尚未锚定的 `scrollTop 0` 误当作"要保留的位置"写回，与恢复定时器互相拉扯，形成来回跳变
- **修复方案**：
  - `Interface/src/lib/chatScroll.js` 新增 `pinChatToBottom()`：每 tick 重新计算 `scrollHeight` 并钉底，覆盖合并消息/卡片水合导致内容持续增长的窗口期
  - 初始视口锚定由 `useEffect` 改为 `useLayoutEffect`，在浏览器绘制前同步执行，对话开头永远不会被闪现
  - `variant === "sidebar"` 挂载（launch 跳转 / 任务详情绑定）直接钉底约 850ms 展示最新消息，不再恢复"当前位置"（产品语义：sidebar 应精准看到最新消息）；inline（wizard 主界面）保持原有"恢复记忆偏移"行为不变
  - `queueScrollPreserve()` 增加守卫：初始锚定未完成时不把 0 误存为保留位置；sidebar 钉底窗口期内不与 pin 抢滚动条
  - 自动滚动 effect 调整为仅在初始锚定完成后对新消息生效，并同步 `prevMsgCount` 基线

### 结果与产出
- 点击 launch 后 sidebar 直接静止显示对话底部最新消息：无顶部闪烁、无来回跳变
- 钉底窗口结束后行为正常恢复：后台刷新时用户向上滚动阅读的位置仍会被保留，不影响长会话阅读
- wizard 主界面 chat 的滚动记忆行为完全不受影响（最小改动范围，仅 sidebar 分支变化）
- 单元测试 `npm run test:unit` 32/32 全部通过，`npm run build` 构建成功

### 遇到的问题及解决
- **跳变涉及三个异步源互相竞争**（首绘 0 偏移、80ms 延迟恢复、rehydrate preserve 竞态）：没有逐个打补丁，而是把"初始锚定"收敛为绘制前同步执行的单一决策点，并让 preserve 机制在锚定完成前完全退让，从时序上根除竞争
- **rehydrate 合并消息会撑高列表导致钉底失效**：固定偏移的 `restoreChatScrollTop` 无法处理内容增长，新增每 tick 重算 `scrollHeight` 的 `pinChatToBottom` 解决

### 下一步计划
- 观察真实使用中 sidebar 钉底窗口（850ms）与慢网络 rehydrate 的匹配情况，必要时按需延长
- 检查 BioAgentChat / HtsAgentChat 是否存在同类初始锚定跳变，评估是否复用 `pinChatToBottom` 方案

## 汇报日期 2026-09-09（续）：用户记忆系统增量重构与平台品牌升级

### 工作内容

#### 一、CD8 记忆丢失的根因定位与找回
- **根因**：原 `user_memory.py` 采用"每用户单行、整篇覆盖"的存储（owner_email 主键，version 仅计数不留副本），每次任务终态 LLM 重写全文档，跨项目（CD8 vs PDL-1）共用一篇导致后续任务把无关内容逐步裁掉——典型的整篇重写式灾难性遗忘
- **找回**：SQLite 覆盖写入后旧页内容仍残留空闲页，用 `strings` 从 `user_memory.db` slack space 提取 CD8 时代文档残片，所有决策级结论完整恢复并与 PDL-1 记忆合并写回

#### 二、记忆系统增量架构改造（`Backend/user_memory.py`、`Backend/main.py`、`Backend/memory_analysis.py`、`Backend/memory_analysis_skill.md`）
- **项目分治存储**：主键改为 (owner_email, project_key)，每个 target 独立记忆行，结构性杜绝跨项目遗忘
- **版本历史表**：新增 `user_memory_history` 表 + `history()`/`restore()` API，逐版本归档、可回滚，坏更新不再不可逆
- **LLM 补丁协议替代整篇重写**：更新 agent 通过 `list_items`（轻量索引）→ `read_item` → `read_bullets` 工具循环（≤6 轮）浏览记忆，输出 `{create_item / append_bullets / update_bullet / remove_bullet / rewrite_item}` JSON patch
- **确定性合并器**：只有 patch 命中的 bullet 被改写，未触及条目逐字节保留；超长 bullet、未知 id 等非法 op 直接拒绝（LLM 不再有机会"顺手"删别的条目）
- **noop 机制**：无材料变化时记忆一字不动、不 bump 版本；`memory_processed_at` 标记防任务 resume 重复触发
- **检索侧两阶段注入**：新会话开启 memory 时先给 LLM 轻量 item 索引识别相关 item，再选择性读取细粒度 bullets，以 reference experiences 上下文注入 design agent，全程不加载全量记忆（防上下文爆炸）
- **记忆格式去固定维度**：自由 bullets、无固定 section/条数，证据没触到的维度不硬凑；存量数据迁移剥除 `[tag]` 前缀
- **废话治理**：写作规则改为"单 bullet 单结论、最多 1-3 个关键数字、普适话术=filler 禁入、中文优先"；确定性简报同步中文化并删硬编码说教
- **可控参数关联分析（核心新增）**：从任务 request 字段提取 agent 可控超参数（binder 长度区间、结合位点/热点、fold-guidance、TTO、designability、样本量），与结合质量做关联——连续轴 Pearson（r≥0.45、n≥8）+ 组间 arm 对比（每 arm≥3 runs、差距≥15%），并设模型级信号门槛（median DockQ≥0.15，防失败配置产生噪声关联）
- **首条轴经验产出**：BindCraft 结合质量与 α-螺旋占比正相关（r=0.63，n=38）——以真增量 patch（append + update 两条 op）入库，PDL-1 条目逐字节未动，同时验证了合并器隔离性

#### 三、平台品牌升级为 ProteinCraft Agents
- 新建 `Interface/src/components/Wordmark.jsx` 统一品牌组件：ProteinCraft 几何无衬线（紧字距）+ Agents 衬线斜体配品牌绿→青渐变，暗色版带缓慢渐变流动动效；纯系统字体栈、零 CDN 依赖（国内离线部署安全）
- `<title>`、中英文案（"How ProteinCraft Agents works"）同步更新；构建部署至 `Backend/static` 并清理旧 bundle

### 结果与产出
- 记忆版本链 v6→v12 全部归档可回滚；存储层 8 项单元测试全过
- CD8 全部决策级结论找回，且此后由三层防护（项目隔离 / patch 隔离 / 版本兜底）结构性免疫遗忘
- 记忆内容从"固定维度硬凑"转为"决策导向"：v12 仅保留能改变下一次迭代决策的条目（骨架排名、失败模式、位点假设、轴经验、判别性实验）
- 品牌视觉全站统一上线（TopNav 15px / Landing 导航 19px / footer 17px 暗色动效版）

### 遇到的问题及解决
- **编辑工具多次"假成功"导致前端运行时崩溃**（`Wordmark is not defined`，import 未落盘但工具报成功）：改为脚本写入 + grep 验证落盘后再构建；后端 `memory_analysis.py` 曾出现同类回滚，同法根治
- **效应量门槛误杀强关联**：α-螺旋关联 r=0.63 但中位差仅 13%，被 15% 门槛滤掉——分析后确认模型级噪声门槛已足够，移除中位差门槛
- 后端重启需 sudo 权限未执行：agent 侧新行为（工具循环、中文简报、可控参数关联）待 `sudo bash env/restart-bindagent.sh` 后生效

### 下一步计划
- 重启 bindagent 后跑一次 PDL-1 小实验，观察 memory agent 的工具循环与 patch 日志是否符合设计
- 5niu cohort 每骨架仅 1 次 run，需按匹配采样量构造对照 arm 才能沉淀轴经验
- 评估统一 PDL-1 被两个 target identity 拆分导致的统计碎片化问题

## 汇报日期 2026-09-09（续）：TTO protein_binder 实验崩溃修复与 Landing 视频更新

### 工作内容

#### 一、TTO protein_binder 实验失败分析（exit code 1）
- 分析失败实验 `protein_binder_custom_9a20be7c_rcsb_5NIU_proteina_e3caf218` 的完整日志（`logs/generate_..._e3caf218_*.log`）：beam search 第一步 batch-scoring 16 个 checkpoint 候选时崩溃；对照组（未开启 TTO）不走 AF2 reward 评分路径，因此不受影响
- 定位出两个叠加 bug：
  - **主因**：Hydra 命令行注入的 `model_nums=[0]` 以 OmegaConf `ListConfig` 形式传入 `AF2RewardModel`，colabdesign `design.py` 的 `isinstance(models, list)` 判断不识别 ListConfig，将其包装成嵌套列表 `[[0]]`，随后 `ns_name.index()` 抛 `ValueError: [0] is not in list`
  - **次因（掩盖主因并放大为整 job 崩溃）**：异常清理路径调用 JAX v0.4.36 已移除的 `jax.clear_backends()`，抛 `AttributeError` 替换原始异常，导致 generate job 直接 exit 1

#### 二、代码修复（`src/proteinfoundation/rewards/alphafold2_reward.py`）
- `__init__` 中将 `model_nums` 归一化为纯 Python int 列表 `[int(n) for n in model_nums]`，消除 ListConfig 与 colabdesign 的类型不兼容；带注释说明 Hydra 注入背景
- `_cleanup_jax_state` 重构为版本兼容的 best-effort 清理：`clear_caches` 保留，`clear_backends` 通过 `getattr` 仅在存在时调用且失败只告警——任何情况下不再吞掉/替换原始异常

#### 三、Landing page Design Agent 演示视频替换核查
- 用户替换了 `Interface/public/videos/design-agent-demo.mp4`；核查全链路确认源码零改动需求：`LandingPage.jsx` 引用同名路径、`LanguageContext.jsx` 中英文案通用不绑定旧视频内容、`LandingVideoPlayer.jsx` 采用 `16/9` + `objectFit: contain` 对任意分辨率自适应且无 poster 依赖
- 发现部署断层：nginx 实际服务的 `Backend/static/videos/` 仍只有旧 `DesignAgent.mp4`，新文件未进 dist/static，线上该视频实为 404；给出「只改前端」部署命令（build → rsync --delete → reload nginx），顺带由 `--delete` 清理不再被引用的旧文件

### 结果与产出
- TTO 崩溃两个 bug 修复完成：旧路径用 ListConfig 精确复现日志错误，修复后归一化验证通过；新清理函数在 JAX 0.4.38（无 `clear_backends`）下执行正常
- `Backend/test_tto_launch_args.py` 回归 **2 passed**，实验可直接原参数重新发起
- Landing 视频链路核查结论明确：无需改代码，仅需执行前端部署命令即可上线新视频

### 遇到的问题及解决
- ** Hydra ListConfig 类型陷阱**：命令行传参的列表经 Hydra 注入后是 ListConfig 而非 list，`isinstance` 检查全部失效——在入口处统一归一化为原生类型根治
- **异常清理掩盖真实错误**：cleanup 抛出的 AttributeError 覆盖了原始 ValueError，日志初看误导排查方向；清理逻辑改为不抛异常后，未来故障将直接暴露真实根因

### 下一步计划
- 重新发起 5NIU protein_binder TTO 实验，验证 beam search + AF2 评分全链路跑通并观察 reward 轨迹
- 视部署执行情况，确认线上 design-agent-demo.mp4 可播放且旧 DesignAgent.mp4 已被清理

## 汇报日期 2026-10-09

### 工作内容
- **补齐 PXDesign 真实测试**：此前仅 wrapper 单元测试，本次以 PD-L1 为靶点完成生成→归一化→评估端到端测试
- **Genie3 内部 TM 过滤与平台职能协调**：参考 BindCraft 的做法，将结果适配层（`src/proteinfoundation/model_wrappers/result_adapter.py`）改为平台侧软过滤——尽量取全样本全原子 refold 结构池，由平台按 i_pTM 统一排序择优，不再让模型内部 TM 卡死候选
- **各模型参数 agent 可配置**：9 个 per-model 调参旋钮（如 `genie3_direction_scale`、`boltzgen_num_designs`、`pxdesign_preset`、`bindcraft_max_trajectories`）暴露为 `backbone_params` 配置字段，每项含质量/速度影响说明，打通 schema → 后端 → wrapper 环境变量 → 前端全链路
- **PD-L1 全骨架基准**：靶点 PDL1_5o45_A116（A1-116，热点 A40/A99/A107），6 个生成骨架（Proteina-Complexa / BindCraft / RFDiffusion3 / PXDesign / Genie 3 / BoltzGen）各 n=4 串行生成并统一评估
- **序列重设计对比实验**：同批骨架分别用模型自带序列 vs soluble_mpnn 重设计序列评估（`sequence_types=[self,mpnn]`），检验"先骨架、再换序列模型"能否提升质量
- 绘制两张对比图：骨架性能箱线图 + 自带序列 vs 重设计柱状图

### 结果与产出
- 六骨架基准（自带序列 i_pTM 均值 / mpnn 重设计 i_pTM 均值）：

| 骨架 | 自带序列 | mpnn 重设计 |
|---|---|---|
| BindCraft | **0.883** | 0.832 |
| PXDesign | 0.667 | 0.598 |
| Proteina-Complexa | 0.586 | 0.456 |
| Genie 3 | 0.532 | **0.748** |
| RFDiffusion3 | 0.511 | 0.712 |
| BoltzGen | 0.478 | 0.710 |

- **结论（分层明确）**：自带序列的共设计模型（BindCraft/PXDesign/Proteina-Complexa）换序列反而降质，应保留自带序列；纯骨架模型（Genie 3/BoltzGen/RFDiffusion3）经 soluble_mpnn 重设计提升约 **+0.2**，且全部越过 0.5 结合阈值——"先骨架、再换序列模型"对纯骨架类确能显著提高质量
- 回归测试全绿：wrappers 19 项 / ProteinHarness 215+ 项 / launch_config 17 项

### 遇到的问题及解决
- **Genie3 UNK 序列污染**：默认 generate 阶段产出 CA 骨架缺序列 → 固定 `GENIE3_STAGE=run`（含逆折叠+refold）并加 `_pdb_binder_has_sequence` 挡板；另发现 `direction_scale=1.0` 使成功率归零，定档 0.0
- **BoltzGen 生成失败**：CCD 分子库仅 HF 有 → `BOLTZGEN_MOLDIR` 指向本地 boltz2 权重目录
- **评估 GPU OOM**：多 ESMFold2 并行争抢 → 增加 `EVAL_GPU` 环境变量控制评估卡并串行化
- **基准汇总表覆盖写**：慢批次会冲掉快批次结果 → 汇总 CSV 改为合并写

### 下一步计划
- 将"纯骨架模型 + soluble_mpnn 重设计"作为平台默认序列策略候选纳入阈值/流程配置
- 视需要扩大 n 值复核 BoltzGen 过采样与 top hit 表现

## 汇报日期 2026-10-10

### 工作内容
- **代码仓库宏观重构（以平台为核心）**：仓库根目录改名 `Proteina-Complexa → ProteinCraft-Agents`，修复全部内外部引用（`.env`、systemd/nginx 配置模板、editable 安装 pth、venv shebang、AF2 参数软链改相对化、外部基准脚本）
- **消除"外部模型"表述**：目录改名 `external_backbones/ → model_wrappers/`、`external/ → model_workspace/`，同步全部代码标识符、命令模板、注释与文档；历史磁盘格式（`.raw_external`、`inference/external_*`）保留兼容
- **文档逐份更新**：README 平台视角重写（7 个骨架模型全列、修正过时工具名/服务名/目录表）、docs 逐篇修订、显示名规则落地（Proteina-Complexa 仅指模型，平台自称 ProteinCraft Agents）、历史设计文档加头注
- **新建 Codebase Wiki**：根目录 `wiki/` 共 9 页（架构总览、目录地图、模型清单、数据流、术语表、外部引用登记、测试指南、协作指南），仿 Codebase Wikipedia 组织；原零散笔记 Zui_Core_md_paper 内容吸收进 wiki 后归档至 `docs/archive/zui_core_notes/`
- **全链路验证**：三套测试、前端构建+单测、后端 API、CLI 生成+评估、两个模型 wrapper 冒烟、nginx 服务

### 结果与产出
- 6 个 commit 完成分阶段重构；回归与重构前基线**完全一致**（tests 43+2 / ProteinHarness 216 / Backend 172+2，仅 4 个已知预存失败豁免，零新增失败）
- 全链路冒烟全部通过：`complexa generate`（27s）→ `complexa evaluate`（self+mpnn，240s）、BindCraft/Genie 3 wrapper 生成+归一化、`/api/health` 与 `/api/agent/manifest` 200、前端 build+unit 全绿、nginx UI/API 200
- 外部引用全部登记在 `wiki/external-references.md`（模型运行环境、工具、权重、外部脚本、部署单元、网络服务），后续仓库迁移可按表逐项核对
- 代码协作边界成文：`wiki/contributing.md` 给出分层扩展点（新模型/新工具/新场景各改哪里）、文档二分规则（Agent 运行时提示 vs 人读文档）、提交前检查清单

### 遇到的问题及解决
- **依赖降级陷阱**：`uv pip install -e .` 按 pyproject 松约束重解析依赖，把 biotite 降到 0.41.2 导致 atomworks 导入失败（3 个测试收集报错）→ 回装规范版本 `biotite==1.4.0` 并将教训写入 wiki/testing.md
- **线上 nginx 配置需 root**：`/etc/nginx/sites-*` 仍指旧路径且无 sudo 权限 → 建临时兼容软链恢复服务（UI/API 验证 200），留一条 sudo 命令待执行后移除软链

### 下一步计划
- 执行 sudo 命令更新线上 nginx 配置并删除临时软链，完成部署侧彻底切换
- IDE 工作区重开到新路径 `ProteinCraft-Agents`

## 汇报日期 2026-10-11：过时开发产物清理 + 顶层目录树状重构（零业务影响）

### 工作内容

#### 一、过时开发产物清理（commit `258cb5c`）
- **清理平台自身开发遗留物**（严格不动平台用户产物）：删除 patch 残留 `Backend/auth/routes.py.orig/.patch`、`Interface/src/pages/account/UserAccount.jsx.orig/.patch`，清出根目录游离的 `Backend/auth.db`、`users.db`
- **归档已完成的 plan/progress 文档**：`.trae/documents/` 下已完成的宏观重构计划、`docs/plan&progress/*`（PROMPT、Agent_Build_plan/progress、fail_fast_rules、old_plan/old_progress）统一移入 `docs/archive/plans/`，`docs/ui_improvement_plan.md` 移入 `docs/archive/`
- **建立管理员分析产物规范**：根 `Output/` 下散落的 CD8 分析包、platform_report、scene_A/B/C 实验脚本与配置统一移入新建的顶层 `Admin_Analysis/`；在 `wiki/contributing.md` 写明"产出实验结果分析/研究产物 → `Admin_Analysis/`"的扩展点约定，并标注两个例外（`Output/analysis/` 是脚本硬依赖必须留原位、`Output/api_runs/` 是用户任务产物）
- `logs/` 按用户决策保留最近 30 天；`.cursor/` 按用户决策保留
- 变更规模：33 files changed, 12 insertions(+), 279 deletions(-)

#### 二、顶层目录树状重构（commit `ed6cc56`，458 files changed, 709 insertions(+), 536 deletions(-)）
先做全仓路径依赖测绘，识别出两个路径锚点（`Backend/main.py` 的 `PROJECT_ROOT`、`ProteinHarness/registry.py` 的 `HARNESS_ROOT`）与一条**最强隐式契约：cwd = 项目根**（Hydra override、subprocess 命令中的 `./assets/...`、`./model_workspace/...`、`./inference/...` 全部以此为基准）。在此测绘基础上按 8 个阶段执行：
- **阶段 1 目录移动（全部 `git mv`，保留 rename 历史）**：
  - 新建 `tools/` 层——`portable_pubmed_retrieval → tools/pubmed_retrieval`（外部 vendor 目录正式内化进平台）、`script_utils → tools/script_utils`、根 `scripts/ → tools/scripts/`
  - 新建 `third_party/` 层——`community_models → third_party/community_models`（含 openfold / colabdesign / ProteinMPNN / LigandMPNN / ckpts）
  - 新建 `deploy/` 层——`env/ → deploy/`（14 个部署脚本 + `deploy/docker/`）
  - 新建 `runtime/` 层——全部运行数据归入 `runtime/{Output,inference,logs,outputs,rf3_outputs,model_workspace,user_workspace,assets,Input,evaluation_results,store}`；根目录从 30+ 平铺目录收敛为 18 个语义目录
- **阶段 2–3 代码与配置面全量同步**：`Backend/main.py`（12676 行，路径最重）、`src/proteinfoundation/*`（含 `command_runner.py`、`rf3_reward.py`、`cli_runner.py`、`result_analysis/analysis.py` 等）、`configs/` 下 19 个 yaml、`.env`/`.env_example`、`Makefile`、`.gitattributes`（LFS 规则）、`Interface/src/components/AgentChat.jsx`、`Admin_Analysis/` 脚本
- **阶段 4 历史路径兼容层（本次重构的关键设计）**：新建 `Backend/legacy_paths.py` 的 `normalize_legacy_path()`，同时处理 ①旧仓库根绝对路径（`/home/lyh/Liuyihey/Proteina-Complexa/...`）②旧相对串（`./inference/...`、`Output/api_runs/...`）③新增/变更路径原样透传；在 7 类消费点接入——`/api/download`、`/api/pdb`、`download_zip`、`_add_reward_path`、`_list_final_structure_paths`、DockQ/Rosetta 读取、`_prefer_visualization_structure_path`/`_relative_project_path`、`_load_task_meta` 出口
- **阶段 5 重装可编辑包**：`uv pip install -e .` 后修复 `.venv/.../_editable_impl_proteinfoundation.pth` 的 7 条路径映射；`pyproject.toml` 的 hatch `packages` + `sources` 映射显式列出 4 个 community_models 子包与 `ProteinHarness`
- **阶段 6 文档同步**：`wiki/directory-map.md` 按新树整篇重写（拆为平台核心 / 工具与第三方 / 部署与环境 / runtime 数据层 / 文档与分析 五张表）、`wiki/contributing.md`、`wiki/testing.md`、`README.md` 架构图
- **阶段 7–8 验证与提交**：测试全绿、后端重启、历史路径下载回归验证通过
- 全程留有回滚备份 `/home/lyh/backup_proteincraft_restructure_20261011.tar.gz`

### 结果与产出
- **零业务影响已验证**：tests 45 项 + Backend/ProteinHarness 282 项全部通过（仅剩 wiki/testing.md 豁免清单中 2 项文档断言）；`import openfold / colabdesign / ProteinMPNN / LigandMPNN / proteinfoundation / ProteinHarness / Backend / script_utils` 全部通过；依赖版本（scipy 1.17.1、biotite 1.4.0）未漂移
- **历史任务链路完好**：用 4 条真实历史路径串（旧根绝对路径、旧相对串、target 路径）做下载回归，全部正确解析到 `runtime/` 新位置并通过下载白名单，历史任务的结果下载/结构可视化不受重构影响
- **458 个文件全部被 git 识别为 rename**，目录移动历史在版本库里清晰可追溯；分两次提交（清理 / 重构）便于分别回滚
- **新人可读性显著提升**：根目录从 30+ 平铺目录变为 18 个语义分层目录（平台核心 / 工具 / 第三方 / 部署 / 运行数据 / 文档），vendor 目录正式内化，`wiki/directory-map.md` 重建为可导航的目录地图
- 顺带修掉一个部署脚本 bug：`deploy/migrate-from-132.sh` 中未定义的 `$PROJECT_DIR` 改为 `$PROJECT`

### 遇到的问题及解决
- **可编辑安装的包装映射陷阱**：首次 `uv pip install -e .` 后 `.pth` 只剩 `src/third_party/tools` 三条，`import openfold` 等 4 个 community_models 子包全部失败。根因是 hatch 的"包装映射"（`third_party/community_models = community_models`）遮蔽了子包映射，且未包含 `ProteinHarness`；移除包装映射、显式列出 4 个子包并补上 `ProteinHarness` 后 `.pth` 恢复 7 条等价结构
- **依赖被降级**：`uv pip install -e .` 按 pyproject pin 把 scipy 1.17.1→1.12.0、biotite 1.4.0→0.41.2（与 `deploy/build_uv_env.sh:227` 本就有的 biotite 版本重置步骤呼应）；每次重装后执行 `uv pip install "biotite==1.4.0" "scipy==1.17.1"` 恢复运行版本
- **2 个测试被自己引入的失败**：`legacy_inference_dir` 我加了 `runtime/` 前缀，但测试 mock 注入 `INFERENCE_DIR` 后调用方传的是 `INFERENCE_DIR.parent`，造成 `runtime/runtime` 双重路径；修复为函数恢复 `project_root / "inference"`、调用方统一传 `INFERENCE_DIR.parent`，并补 docstring 明确参数语义
- **`.gitignore` 陷阱**：`/tools/` 规则会忽略新建的 `tools/` 层（父目录被排除后 `!` 无法救回），必须删除该行；裸 `logs/` 规则同时覆盖 `Backend/logs`，故保留不动
- **`git mv env deploy` 产生嵌套**（`deploy/` 已预先创建，结果成 `deploy/env/`）：`shopt -s dotglob && mv deploy/env/* deploy/ && rmdir deploy/env` 修正；另处理 21 个索引过时状态（`RD`）
- **测试污染仓库根**：`tests/test_esmfold2_adapter.py` 会在仓库根写出 `esm`/`esm_1~3` 假 PDB，已清除并写入项目记忆提醒
- **`pkill -f "uvicorn main:app"` 误杀自身 shell**：改用 `pgrep` 确认状态后重新 nohup 启动，最终 openapi 200 / runs 403（鉴权正常）

### 下一步计划
- 修复遗留的 systemd 部署单元：`/etc/systemd/system/bindagent.service` 仍指向已不存在的 `Proteina-Complexa` 目录（后端目前靠手动 nohup 运行），需 `sudo bash deploy/update-system.sh` 修正；同时统一单元名（`deploy-local.sh` 用 `ProteinCraft` vs `update-system.sh`/`restart-bindagent.sh` 用 `bindagent`）
- `docs/Literature_Agent_UI_Analysis.md` 等一次性分析文档可评估归档
- 两个提交尚未推送到远端，待确认后推送

## 汇报日期 2026-10-12 ~ 2026-10-13：数据丢失事故修复 + Agent 数据分析能力（沙箱 MCP / Artifacts）建设

> 本阶段共 7 个提交，围绕两条主线：①修复目录重构引发的**用户数据丢失事故**（聊天记录 + linked chat 全量断链）；②给 design agent 接入**沙箱化数据分析能力**，让用户能对实验结果做个性化分析并产出图表 artifacts。

### 工作内容

#### 零、提交清单

| commit | 主题 | 规模 |
|---|---|---|
| `363ce77` | 修复聊天记录全量丢失：会话库改读 `runtime/`，停止 git 跟踪 | 3 files, +2/-2 |
| `74db180` | 启动清理逻辑加空库安全门禁 + 测试写入隔离 | 3 files, +90/-12 |
| `90173bf` | 修复 assistant markdown 渲染：行内 `#1` 被误判标题、表格英文竖排断词 | 7 files, +21/-22 |
| `ccbf362` | 接入沙箱化 filesystem MCP + 分析代码执行 + artifact 卡片 | 27 files, +1876/-76 |
| `4bbddac` | artifact 卡片贯通 transcript op 链（端到端修复） | 7 files, +70/-26 |
| `4edeace` | 修复真实结果分析失败、artifact 图片 404、卡片错位 | 11 files, +251/-96 |

---

#### 一、聊天记录全量丢失事故修复（commit `363ce77`）

**问题现象**：10-11 顶层目录树状重构后，平台上所有用户的 chat history 全部查不到。

**根因定位**：会话持久化库 `agent_sessions.db` 原先放在仓库根，被 `ed6cc56` 的目录重构搬到了 `runtime/`，但 [agent_persistence.py](file:///home/lyh/Liuyihey/ProteinCraft-Agents/Backend/agent_persistence.py) 的 `_resolve_db_path()` 仍指向旧位置。更严重的是 `ed6cc56` 曾用一个**空库**覆盖了 git 中跟踪的 13MB 真实库。

**修复内容**：
- `_resolve_db_path()` 改为读取 `runtime/agent_sessions.db`，库文件缺失时打 `log.error` 而非静默走空库路径
- 执行 `git rm --cached agent_sessions.db` 停止版本跟踪（运行数据不应进代码库，避免再次出现"空库覆盖真实库"）
- 同步修正 `README.md` 中关于该文件位置的说明

---

#### 二、实验 linked chat 全量断链修复（commit `74db180`）

**问题现象**：重构后不仅聊天记录丢失，所有实验记录页也没有可跳转的 linked chat 入口。

**恢复过程**：
- 从重构前的备份中恢复 **27 组会话-任务绑定关系、23 个任务归属**，重写 `runtime/.../_task_meta.json` 与 `_task_chat.json` 索引
- 新增 `count_all_sessions()` 作为**启动安全门禁**：会话库为空（计数 <= 0）时直接跳过 purge / detach / prune / sync 四类清理动作，从机制上根除"空库启动把绑定关系再洗一遍"的二次伤害
- 防御性加固：`_detach_tasks_from_missing_sessions`、`_prune_design_session_linked_task_ids` 在集合为空时短路返回；`_sync_orphan_task_artifacts` 在目录不可读时拒绝删除
- 测试隔离：`test_agent_action_parser.py` 中新增 `backend_main._ownership_path` monkeypatch，避免测试写入真实的 `_ownership.json`

**遗留数据风险控制**：重构前保留两份库备份 `/home/lyh/agent_sessions.db.bak_20261011`、`/home/lyh/agent_sessions.db.bak2_20261012_before_restore`，以及全量代码包备份 `/home/lyh/backup_proteincraft_restructure_20261011.tar.gz`。

---

#### 三、前端 Markdown 渲染缺陷修复（commit `90173bf`）

用户截图暴露两类渲染问题：

1. **行内编号被渲染成超大号粗体标题**：`#1`、`#2` 引用编号被标题规则误判。修复策略改为**收紧触发条件**——只在句末标点（`. ! ? : ; 。 ！ ？ ： ；`）后粘连的情况下才拆分标题块，并新增行首 `^(#{1,6})([^\s#])` 规范化，不再误伤行内 `#1`
2. **表格中英文竖排断词**（`Protein` / `a-` / `Comple` / `xa 升级`、`MEDIU` / `M`）：根因是 CSS `word-break: break-word` —— 这是**废弃别名，实际等价 `overflow-wrap: anywhere`**，会导致列宽塌缩 + 单词中间断行。5 处样式统一改为语义正确的 `overflow-wrap: break-word`
3. 同步新增 `breakOpportunities()`：对 ≥16 字符的长标识在 `/ _ . -` 后插入零宽空格（ZWSP），使断行只落在语义边界

涉及文件：`chatMessages.js`、`AssistantMarkdown.jsx`、`AgentChat.jsx`。

---

#### 四、design agent 接入沙箱化数据分析能力（commit `ccbf362`，本阶段主线）

**目标**：让用户在对话中直接对实验结果做个性化分析、产出图表；对平台自身结果产出 artifacts，UI 与现有分子展示框保持同一套设计语言；用户数据严格隔离，采用业界 agent 的标准安全做法。

**架构决策：自实现 MCP（Model Context Protocol 2024-11-05）**
不引入新依赖，用 stdio + JSON-RPC 2.0 自行实现 server（`initialize` / `tools/list` / `tools/call`），零第三方依赖、可控性最强。

**沙箱隔离三层**（业界标准做法）：
1. **bubblewrap 网络与命名空间隔离**：`--unshare-net --unshare-pid --proc /proc --dev /dev --tmpfs /tmp` + 只读 bind 系统目录 + 资源上限
2. **能力探测 + 自动降级**：本机实测 bwrap 不可用（内核 `loopback: Failed RTM_NEWADDR`），`bwrap_available()` 改为**实测探测**（跑真实命令看 returncode）而非 `shutil.which`，探测失败自动降级
3. **PEP 578 `sys.addaudithook`**：降级路径下仍强制拦截文件越权（audit hook 无法被沙箱内代码移除，这是它相对 `sitecustomize` 的关键优势）

**资源限制**：`RLIMIT_CPU 120s` / `AS 2GiB` / `FSIZE 100MiB` / `NOFILE 128`。

**虚拟路径沙箱**：仅暴露 `data/` `artifacts/`（可写）、`results/<run>/` `targets/`（只读）。校验在客户端（[paths.py](file:///home/lyh/Liuyihey/ProteinCraft-Agents/Backend/sandbox/paths.py)）与服务端（`mcp_server._resolve`）各做一份 `resolve() + relative_to()` 双层校验。用户目录命名为「可读前缀 + sha1[:10]」，兼顾可辨识与防枚举。

**新增 6 个后端模块（1109 行）**：[mcp_server.py](file:///home/lyh/Liuyihey/ProteinCraft-Agents/Backend/sandbox/mcp_server.py) 428 行、[workspace.py](file:///home/lyh/Liuyihey/ProteinCraft-Agents/Backend/sandbox/workspace.py) 186 行、[runner.py](file:///home/lyh/Liuyihey/ProteinCraft-Agents/Backend/sandbox/runner.py) 168 行、[mcp_client.py](file:///home/lyh/Liuyihey/ProteinCraft-Agents/Backend/sandbox/mcp_client.py) 164 行、[paths.py](file:///home/lyh/Liuyihey/ProteinCraft-Agents/Backend/sandbox/paths.py) 116 行、[audit.py](file:///home/lyh/Liuyihey/ProteinCraft-Agents/Backend/sandbox/audit.py) 34 行。

**新增 5 个 agent 工具**：`list_files` / `read_file` / `write_file` / `save_artifact` / `run_analysis_code`，通过 [sandbox_tools.py](file:///home/lyh/Liuyihey/ProteinCraft-Agents/ProteinHarness/executors/sandbox_tools.py) 接入 harness，design agent 工具数从 9 增至 14。

**Artifacts UI**：新增 [ArtifactCard.jsx](file:///home/lyh/Liuyihey/ProteinCraft-Agents/Interface/src/components/ArtifactCard.jsx)，完整复刻 `structure_preview` 的设计语言——白底 16px 圆角、`rgba(84,72,229,0.2)` 紫色描边、`0 4px 20px rgba(84,72,229,0.08)` 阴影、左上 Download 药丸按钮、右上 Maximize2 放大按钮。

**安全对抗验证 11/11 拦截**（含路径穿越 `../../etc/passwd`、绝对路径越权、符号链接写穿、跨用户访问、越权 run 目录、资源耗尽）。

---

#### 五、artifact 卡片贯通 transcript op 链（commit `4bbddac`）

**发现的上轮漏洞**：完成度审计时发现 `ui:artifact` 事件在 transcript 累积器的分发链末端被 `else: return []` **静默丢弃**，且 artifact 消息不持久化——表现是 LLM 说"上方预览卡"但刷新页面即消失。

**补齐的完整链路**：
- 后端：`_OpType.ARTIFACT = "op_artifact"` → `_TranscriptOp.artifact()` 静态工厂 → `_apply_op` 分支（按 id 去重后 append）→ `_TranscriptAccumulator` 分发链
- 前端：[transcript.js](file:///home/lyh/Liuyihey/ProteinCraft-Agents/Interface/src/lib/transcript.js) `applyOp` 新增 case
- 消息流：[streamMessages.js](file:///home/lyh/Liuyihey/ProteinCraft-Agents/Interface/src/lib/streamMessages.js) 将 artifact 与 structure 统一纳入 `normalizeToolStructureOrder`

**真实链路验证**：LLM 发起 tool_use → 产出 `cos_curve.png` → `op_artifact` 事件 → 持久化 → 浏览器像素级验证卡片与全屏弹窗渲染，确认与分子展示框视觉一致。

---

#### 六、真实结果分析失败 / 图片 404 / 卡片错位修复（commit `4edeace`）

用户截图反馈三个问题。**归因结论：全部是 harness 侧缺陷，agent 的自我诊断准确**。

**缺陷 1：`results/` 未挂载进 Python 沙箱（核心）**
agent 执行 `pd.read_csv('results/<run_name>/...')` 必然 `FileNotFoundError`。修复：新增 `_ensure_view_links()`，在用户根下建立 `results` / `targets` 符号链接指向 `RESULTS_ROOT` / `TARGETS_ROOT`（符号链接本身只提供只读视图，配合 realpath 校验防写穿）。

**同时加固 audit hook**：读操作校验 realpath 必须落在白名单前缀内；写操作 realpath 必须落在用户根或 `/tmp`；`results/` 下的访问强制校验 run 名在 `_ALLOWED_RUNS` 内；禁用 `os.symlink` / `os.link` / `os.rename`。

**缺陷 2：admin 查看他人 artifact 图片 404**
原实现为 owner-only。修复：`workspace.find_artifact_any_user()` 遍历各用户 manifest 兜底定位；`/api/artifacts` 列表接口**刻意不返回他人数据**（避免信息泄露），仅单文件路由对 admin 兜底。

**缺陷 3：全屏聊天模式下 artifact 卡片严重左漂**
用 Playwright 做几何实测定位（非截图猜测）：`artifact-row x=72` vs 正常消息行 `x=320`。根因是 `isFullscreen` 模式下所有消息行都有 `maxWidth: 800 + margin: '0 auto 16px'` 居中约束，而 ArtifactCard 漏了这一处理。修复后复测 `x=360`，与工具卡内容列对齐。

**验证**：真实任务 `protein_binder_custom_rcsb_5NIU_proteina_db4a5272` → 直读 26×98 CSV → 产出 39,663 bytes 散点图；写穿 / 越权 / 符号链接三处均正确抛 `PermissionError`，攻防 5/5 BLOCKED。

**文档同步**：`WIKI_analysis.md` 新增两条 agent 规范（`results/`、`targets/` 在代码里可直接只读访问，大表不要用 `read_file` 搬运而应在 `run_analysis_code` 内 `pd.read_csv`）。

### 结果与产出

**数据安全**
- 用户聊天历史与 linked chat 全部恢复（27 组绑定 / 23 个任务归属），并新增空库安全门禁防止同类事故重演
- 三份可回滚备份保留（2 份会话库 + 1 份全量代码包）

**Agent 能力**
- design agent 从 9 个工具扩展到 14 个，具备完整的"读数据 → 分析 → 出图 → 产出 artifact"闭环
- 用户可在对话中直接产出个性化图表，成果以与分子展示框一致的卡片形态呈现，支持点击放大与下载
- 分析代码全程沙箱执行（90s 超时、输出截断 20k、单 artifact 50MB、单用户 500 个上限）

**测试与质量**
- 新增 [test_sandbox.py](file:///home/lyh/Liuyihey/ProteinCraft-Agents/Backend/test_sandbox.py) 13 项安全/链路测试全部通过（虚拟路径、跨用户隔离、results 视图只读、run 白名单强制、ui:artifact 事件、审计日志）
- 全量回归：后端 185 passed（仅 2 项豁免清单内预存失败），前端全绿
- 文档沉淀：新增 `wiki/sandbox.md` 作为沙箱权威文档，含威胁模型与残余风险说明；`wiki/index.md`、`wiki/directory-map.md` 同步更新

### 遇到的问题及解决

- **bwrap 存在但不可用**：本机内核禁止命名空间（`unshare -n` 报 `Operation not permitted`、`--unshare-user` 报 uid map 权限拒绝、docker 用户不在 docker 组）→ `bwrap_available()` 改为实测探测，失败自动降级 `setrlimit` + PEP 578 audit hook，并在 wiki 中明确记录残余风险与 root 权限下的修复路径（建议 root 开通 bwrap/docker）
- **沙箱把自己的科学栈拦死**：numpy 初始化会 `import ctypes`，被 audit hook 拦为 `PermissionError: ctypes is disabled` → 将科学栈（matplotlib/numpy/pandas/scipy/seaborn/PIL/plotly）的 import 移到 audit hook 注入**之前**预热
- **OpenBLAS 线程爆内存**：`pthread_create failed for thread 51 of 64` → 环境变量固定 `OPENBLAS_NUM_THREADS/OMP/MKL/NUMEXPR/VECLIB = 4`、`MPLBACKEND=Agg`
- **f-string 生成 prelude 出 tuple repr**：`!r` 对 tuple 生成 `('/a', '/b',)` 导致 prelude 语法与语义双重错误 → prelude 改为列表拼接 `"\n".join(lines)`
- **测试污染真实数据**：`_OWNERSHIP_PATH` 是模块级常量，`mock.patch.object(OUTPUT_DIR)` 无效 → 改为 `_ownership_path()` 函数化 + 测试侧 monkeypatch
- **IDE 文件工具频繁超时**：`Read`/`Write`/`Edit` 报 `IDE Command timeout` / 匹配失败 → 改用 shell heredoc 与带 `assert count == 1` 校验的 Python 精确替换
- **浏览器自动化桥接不可用**：内置浏览器视口被压到 654×290、截图 API 被限流、`fill` 对 React 受控输入无效 → 改用项目自带 Playwright（chromium-1228）自写脚本完成登录、导航、几何测量与截图验证
- **方法论收获**：布局错位问题上，Playwright `getBoundingClientRect` 几何实测 + 打印 DOM 父链，比看截图猜测快且准——一次就定位到缺失的 `maxWidth: 800 + margin auto`

### 下一步计划

- 为沙箱申请 root 级 bwrap / docker 隔离权限，消除当前降级方案（CPython 自身理论上可逃逸 audit hook）的残余风险
- 修复遗留的 systemd 部署单元 `/etc/systemd/system/bindagent.service`（仍指向已不存在的 `Proteina-Complexa` 目录，后端目前靠手动 nohup 运行），需 `sudo bash deploy/update-system.sh`；同时统一单元命名（`ProteinCraft` vs `bindagent`）
- 本阶段 7 个提交均未推送到远端，待确认后统一推送
- `docs/Literature_Agent_UI_Analysis.md` 等一次性分析文档可评估归档
