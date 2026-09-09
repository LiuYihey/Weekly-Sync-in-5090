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
