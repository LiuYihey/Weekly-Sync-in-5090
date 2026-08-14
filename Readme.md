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
