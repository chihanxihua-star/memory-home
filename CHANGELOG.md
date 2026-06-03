# cheng 改动日志（CHANGELOG）

> 这个文件记录 cheng 项目（chat.jessaminee.top）的**每一次改动**——前端 `cheng-memory`、后端 `memory-home`、以及基建/进程层（nohup 服务、迁移、forge、nginx）。
>
> **为什么存在**：cheng 大量状态不在 git 里（nohup 起的进程、并存的服务器、半截迁移、forge 指向哪个端口……），git 和代码看不出来。新 CC 接手前**先读这个文件**，能知道前人改过什么、现在系统处于什么状态。
>
> **写入规矩（给未来的 CC）**：
> - 只要动了 cheng 的代码或基建，就在**最上面**追加一条。倒序，最新在上。
> - 每条必须**自包含**：哪怕 root 聊天记录被压缩/删了，光看条目也能懂"问题→真因→改了什么→结果"。
> - transcript 指针只是**深挖细节**用的兜底，不是主记录。关键词要选独特的（会话 id、独特中文短句），方便 `grep`。
> - 标签：`[前端]` cheng-memory / `[后端]` memory-home 代码 / `[基建]` 进程·nginx·forge·迁移。
>
> **transcript 在哪**：root 自己的 Claude Code 记录在 `/root/.claude/projects/-root/*.jsonl`，按 sessionId 命名。`grep -l "<关键词>" /root/.claude/projects/-root/*.jsonl` 能定位是哪个 session。

---

## 2026-06-03 · [基建/后端] 工具卡片用 hook 实时还原（方案B）✅已激活已推送

**背景**：切 tmux 驱动后工具卡片没了（tmux-manager 读 transcript 只抽正文+思考，没抽工具调用；老 stream-json 的 cc-manager 才 emit tool_use/tool_result）。前端工具卡片渲染（ToolCallsBlock/StreamingBubble/历史）**完全没坏**，纯缺后端喂事件。用户选**方案B**：只用 hook 把工具调用实时报上来，**不碰判轮/看门狗/600s**（那套调好的不动，风险最低）。

**实测确认的 hook payload（claude 2.1.161，别信文档猜测）**：
- `PreToolUse` 每个工具都触发：`tool_name / tool_input / tool_use_id`。
- `PostToolUse` 只成功触发：`tool_response`(工具特定形状，Bash 是 `{stdout,stderr,...}` 不是 `{type,text}`) + `tool_use_id` + `duration_ms`。
- `PostToolUseFailure` 失败触发：`error`(字符串，含"Exit code N\n<原因>") + `tool_use_id`。
- 三者用 tool_use_id 配对，正好映射现有 tool_use/tool_result 管道（前端按 id 配）。

**改了什么**（3 处，**已重启生效**）：
- `[基建]` `/home/claude-user/chat-sandbox/.claude/cheng-tool-hook.sh`（新）：读 stdin → 后台 curl 到 `127.0.0.1:3002/api/internal/cc/tool-hook?phase=pre|post|fail` → **永远 exit 0、不阻塞 CC**（铁律：hook 同步阻塞会拖死 CC，所以 `D=$(cat)` 读完立刻 `( ...curl --max-time 2 & )` 子shell后台）。
- `[基建]` `/home/claude-user/chat-sandbox/.claude/settings.json`（新）：PreToolUse/PostToolUse/PostToolUseFailure 三个 hook，matcher `*`，调上面脚本带 phase。⚠️项目级、作用域只在 chat-sandbox 跑的 cheng CC。
- `[后端]` `index.js` 加 `POST /api/internal/cc/tool-hook`（loopback only + 立即 200）：phase=pre→`cc.emit('tool_use')`、post→`cc.emit('tool_result' is_error:false)`、fail→`cc.emit('tool_result' is_error:true, content=error)`。复用现有 cc.on 处理器（push activeTurn.tools + 广播 + 落库 tool_calls）。`node -c` 通过。

**已排的险**：① 隔离 tmux 会话测过——加 hooks 后 CC 交互启动正常到 `bypass permissions on`，**没被 hook 审批挡住**；② -p 模式实测 hooks 正常触发；③ 防阻塞写法到位；④ 重启后 CC 已就绪、接口 loopback 返回 ok。

**⚠️ 激活时抓到并修掉的安全洞（顺带修了老的）**：`/api/internal/*` 的"只许 loopback"判断（`req.ip/socket.remoteAddress==127.0.0.1`）在本站架构下**被绕过**——站走 Cloudflare(回源 http:80) + nginx 反代 `/api/`，后端看到的源永远是 nginx(本机)，所以外网经 CF→nginx→后端也能打 `/api/internal/*`。**新加的 tool-hook 接口和早就存在的 `/api/internal/cc/restart` 都暴露了**。修法：nginx `chat.jessaminee.top` 配置加 `location /api/internal/ { return 403; }`（在 `location /api/` 之前，最长前缀优先），`nginx -t && systemctl reload nginx`。验证：公网/源站打 `/api/internal` → 403；hook 直连 `127.0.0.1:3002`（不走 nginx）→ 仍 ok。⚠️ nginx 配在 `/etc`(仓外)，换 VPS 要手搬这条。

**激活方式**（已执行）：`systemctl restart cheng-backend`（重载 index.js + 重建 cheng tmux 会话读 hooks）= 一次软失忆。重启后模型回到 savedCfg=sonnet-4-5。

**已提交并推送**：子模块 memory-backend `bb26e72`(tmux-migration)、外层 memory-home `e60ca96`(main)。⚠️ hook 脚本/settings.json(`/home/claude-user/chat-sandbox/.claude/`)+ nginx 改动**都在仓外**，换 VPS 需手搬(内容见本条)。回滚=删那两文件 + 撤 index.js 那段 + 撤 nginx 那行。

**仍待用户实测**：真聊天时澄调工具→卡片是否实时蹦（机制各环节已验，只差真turn)。

**transcript**：root session `1f249b92`。关键词 `cheng-tool-hook`、`tool-hook?phase`、`PostToolUseFailure`、`方案B`、`api/internal 403`。

---

## 2026-06-03 · [前端] 低语收藏 UI 多轮微调 + 收藏心实心(聊天持久) ✅

承接 e1ed9fd 之后的用户逐条调整（commit `65e9129` + 后续持久化）：

- **收藏弹层**：先试改全屏思考风格→用户否("点击收藏不需要平铺")，`git restore` 回退到底部弹层。最终：弹窗**去圆角**(borderRadius 0，聊天A `cp-modal bottom` inline 覆盖 / 拾光 inline)；但**按钮要圆角**。
- **「建并存」「收藏为故事」按钮**：参考图(浅色/深色/跟随系统那种分段按钮，https://ibb.co/xqxhR7C1)→ 黑色按钮 `#2b2b2b` 深底 + `#f5f2ec` 米白字 + 圆角 8px + 字距；禁用=描边透明。聊天A 用 `cp-fav-new`/`cp-select-fav` CSS，拾光用模块级 `segBtn(on)`。
- **拾光收藏入口跟聊天统一**：去掉"收藏"/"选段"文字标签 → 纯图标(同款心 + 勾选 SVG)。
- **粉色清零**：残留 `#e0738a`(选段选中外框/选中圈/hover)全 sed 成 `var(--text-primary)`，整套黑白。
- **收藏后心填实心**：收藏成功(单条♥ + 收藏为故事多选)→ 该消息心形 `fill: var(--text-primary)`(浅色主题=黑)。
  - 聊天A：`favedIds` Set；onDone 标记 `source_message_id`；**且持久**——切/进对话时 `useEffect([convId])` 查 `favorites_cheng` where `source_conversation_id=convId` 拉回所有已收藏 id，刷新/后台重进仍实心。
  - 拾光：`favedUuids` Set，仅本会话内(拾光消息无真实 id，`source_message_id` 存 null，做不了持久)——用户同意不做。

**状态**：✅ build 过、lint 无新错。`65e9129` 已提交(按钮/弹窗/图标/会话内实心)。〔更正 06-03晚：原写「持久化那段尚未提交、未 push」已过时——持久化后来提交为 `6d92e44`，**整条 split-chat-web 分支已全部 push**(实查 `origin/split-chat-web..split-chat-web` 为空)。〕transcript root session `1f249b92`。关键词 `favedIds`、`source_conversation_id 拉回`、`segBtn`。

---

## 2026-06-03 · [前端] 低语板块改「思考风格」+ 加刷新（同日）✅

**用户要求**：① 给低语加刷新；② 面板风格参考聊天界面「思考(use-style/thinking)面板」那种调性（极简单色、下划线输入、字距拉开、衬线正文、纯文字按钮——不要我先前那种粉色按钮+边框卡片）。注：用户先说参考涟漪，我问岔了，改口说是参考思考风格。

**改了什么**（全在 `MemoryManager.jsx` 的 DiyuPanel 一族）：
- 新增 `diyuTextBtn`(素文字按钮) / `diyuRow`(扁平列表行带底部 hairline)，删掉没人用了的 `diyuCard`。
- DiyuPanel 主视图：顶部「共 N 个合集」+ 右侧「+ 新建 / 刷新」文字按钮；搜索改下划线(`underlineStyle`)+ × 清空；合集列表改扁平行(衬线名 + 条数)。
- DiyuCollection(合集内)：返回/刷新/删 改文字按钮；搜索下划线+×；加载更多改文字按钮；**也加了刷新**(reload load(0))。
- DiyuFavCard + DiyuSearch 结果：卡片框 → 思考块那种「左竖线 + 衬线正文」。

**收藏弹层试改全屏→已撤回**：commit e1ed9fd 之后我曾把聊天A `CollectionPicker` + 拾光 `CVCollectionPicker` 改成全屏思考面板样式，但**用户说"点击收藏不需要平铺，改回之前的样子"**，已 `git restore` 丢弃那两处未提交改动。所以**收藏弹层维持原样=底部弹层**(`cp-modal bottom` / 圆角 14px / 粉色按钮)。只有**板块本身**是思考风格，弹层不动。

**已 commit**：`e1ed9fd`(低语全功能 + 板块思考风格 + 刷新，599行3文件)。注意之前以为"拾光换源/删聊天记录那批未提交"是**记错了**，它们早提交过(162eb23/68dc8fa)。e1ed9fd 之后工作区已干净(弹层全屏改动已撤回)。〔更正 06-03晚：原写「未 push」已过时——split-chat-web 分支后来全部 push(含 e1ed9fd / 65e9129 / 6d92e44)。〕

**状态**：✅ build 过、dist 更新。transcript root session `1f249b92`。关键词 `diyuTextBtn`、`左竖线`、`收藏弹层维持底部弹层`。

---

## 2026-06-03 · [前端] 低语「选段」框选 bug 修复（同日，紧接下条）✅

**用户实测报的 bug**：聊天 A 选段时，「只点中自己(user)的一条，却自动把澄(cc)的也选上」；首尾框选不按预期。拾光也有此毛病（但拾光当时看着像没事，是因为正好选段在第一条上）。

**真因**：「选段」按钮原本会把**点它的那条消息偷偷设成框选起点(anchor)**。于是用户以为在"点第一条"，其实点的是**第二个端点**，范围立刻从『选段那条』拉到这里、把中间的澄全圈进来。

**改了什么**（`ChatPanel.jsx` + `ChatViewer.jsx` 对称改）：
- `enterSelect/enterSel` 改成**只进入多选模式、不预选任何条**（anchor=null）。
- `toggleSelect/toggleSel`：第一次点=设为起点(只选这一条)；之后点未选的→选「起点→此处」连续段；点已选的中间条→剔除。
- 起点存进 **ref**(`selectAnchorRef`/`selAnchorRef`)再读，杜绝闭包取到旧 anchor。
- 浮条没选时文案改「点第一条和最后一条」。
- 现在符合用户记忆中的模型：进多选 → 点第一条(干净，不带cc) → 点最后一条(中间自动填)。

**状态**：✅ build 过、dist 更新。〔更正 06-03晚：原写「仍未 commit」已过时——此修复随 e1ed9fd 一并提交，split-chat-web 已全部 push 到 6d92e44。〕transcript：root session `1f249b92`（同下条）。关键词 `enterSelect 不预选`、`selectAnchorRef`、`点第一条和最后一条`。

---

## 2026-06-03 · [前端] 低语收藏夹 — 接线 + 聊天A/拾光收藏入口全做完 ✅已上线

**这是啥**：接着上个 CC（session 5ab69670）只建了表、写了板块本体没接线的活，把「低语」收藏夹**前端全部做完并 build 上线**。三块：①把已写好的板块挂进主壳 B 导航 ②聊天 A（ChatPanel）加 ♥ 收藏 + 故事多选 ③拾光（ChatViewer）加收藏 + 故事多选。设计来龙去脉见 [[project_diyu_favorites]] 和上一条。**图标用户最终定 🖼 emoji**（不是上个 CC 在上一条里写的「线条相框 SVG」，用户这次明确改口）。

**核对过的底层（没动表，只确认）**：`favorites_cheng` 的 `source_conversation_id`/`source_message_id` 都是 **uuid 类型**（nullable）、`original_created_at` timestamptz、`tags` 非空但默认 `'{}'`、RLS=`allow_anon_all`（anon 增删改查全放行，前端直连）。`messages.id`/`conversation_id` 都是 uuid。做了一次真插入测试（建合集+2条+级联删）确认字段/类型/排序全对，已清干净（favorites_cheng 现 0 条）。

**改了什么**：
- `[前端] MemoryManager.jsx`（主壳 B）**接线 3 处**：`TABS` 加 `{key:"diyu",label:"低语"}`；HomePanel 卡片加 `{key:"diyu",icon:"🖼",name:"低语",sub:"珍藏的话"}`；中间 panel 渲染数组加 `{key:"diyu",Comp:DiyuPanel}`。点首页 🖼 卡片即进。**板块本体(DiyuPanel 等)是上个 CC 写的，本次没动**。
- `[前端] ChatPanel.jsx`（聊天 A，独立 /chat/ 入口，用 `supabase` 客户端不是裸REST）：
  - `MessageBubble` 动作区(复制旁)加 **♥**(收藏单条) + **选段**(进故事多选)两个按钮；只对真实 id(非 local-)且有正文的消息显示。澄的正文用 `extractThink().content` 去掉思考块。
  - 新增 `CollectionPicker` 底部弹层(抖音式：新建合集 / 选已有 → 写 favorites_cheng 文字快照)。
  - 故事多选：**首尾框选**(点已选→剔除；有起点则选起点到此处连续段；renderItems 顺序即对话原始顺序)，底部浮条「已选N · 收藏为故事」。状态 `favPending/selectMode/selectAnchor/selectedIds`，handler 在 renderItems useMemo 后。
- `[前端] ChatViewer.jsx`（拾光，主壳 B 的 tab，懒加载）：同样的 ♥/选段/首尾框选/底部浮条/弹层，但**自包含**(`CVCollectionPicker` + 内联样式 + 自己的 toast)。⚠️**坑已避**：拾光读 supabase 时 messages 只 select 了 `role,content,thinking,created_at` **没取 id**，且一行 DB 拆成多气泡(uuid 是合成串 `${sid}-${idx}`)→ 没有真实消息 uuid → 收藏的 `source_message_id` 存 **null**(快照+会话id+时间足够回溯)。`source_conversation_id`=currentConv.uuid(真会话uuid，加了正则校验非uuid则存null)。注意拾光**原有的 `favorites`(localStorage 星标整段对话)跟这个收单条到 supabase 是两回事**，我用独立命名 `diyuPending/selMode/selIds` 没碰它。

**收藏的 sender 约定**：聊天A存 `"user"/"assistant"`，拾光存 `"human"/"assistant"`——DiyuPanel 的 `DIYU_SENDER_NAME` 把 human/user 都映射成「小茉莉」、assistant→「澄」，两边都覆盖。

**状态**：✅ `npm run build` 三块全过、`dist` 已更新(按 [[feedback_no_restart_cc]] 只 build 不重启)。lint：我的新增标识符**零新错**(既有基线 100+ 错是别处老问题)。**split-chat-web 分支未 commit**(沿用此前拾光换源/删聊天记录那批未提交改动，等用户验收一起提交)。〔更正 06-03晚：此后全部提交并 push——接线本身=e1ed9fd，UI+持久化=65e9129/6d92e44；实查本地未领先 origin，分支已全推。〕回滚=git 还原三文件。

**用户还没实际点过**——建议验：主壳B首页点🖼进低语→聊天里 ♥ 一条→选/建合集→回低语看到→选段框选几条收成故事→拾光里同样试。

**transcript 指针**：root session `1f249b92`（06-03，"睡醒断片，看上个CC在忙啥"）。关键词 `CollectionPicker`、`CVCollectionPicker`、`首尾框选`、`selectAnchor`、`source_message_id 存 null`。

---

## 2026-06-02 · [基建/Supabase] 低语收藏夹 — 建表(设计定稿,前端待做)

**这是啥**：cheng 新功能「低语」= 主壳 B 第 9 个独立板块,收藏澄说的话 / 把澄连讲的故事分段收成「合集」(带你的互动,一起写小说向)。是 [[project_cheng_app_plan]] 定稿排序的 ②收藏夹。本次只**建了 Supabase 表 + 存档设计**,前端代码未写。

**设计要点(用户拍板)**：① 故事分段成合集 ② 零散同类单条也能进合集 ③ 能搜索 ④ 合集能命名。抖音式♥收藏(点♥弹底部面板,选/新建命名合集→确认,分类当下完成不用再去整理)。存**文字快照**(收藏即拷,原 session 没了/拾光换源都照样显示)。防卡=分页。合集**平铺一层**(无嵌套,像抖音)。

**数据模型(Model Y:合集=唯一命名容器,比"单条/故事两类型"简洁)**：
- `favorite_collections_cheng`(合集): id / name / created_at
- `favorites_cheng`(收藏的单条消息): id / collection_id(级联删) / sender / **content(快照)** / source_conversation_id / source_message_id / **original_created_at(合集内按此排序→故事才顺)** / tags[] / note / position(预留) / created_at
- 索引:collection_id、tags(gin)、original_created_at。RLS=`allow_anon_all`(跟 board_cheng/memories_cheng 同款,前端 anon key 直连 REST)。
- migration 名:`create_diyu_favorites_cheng`。回滚=drop 两表。

**交互(定稿,前端待实现)**：聊天 A 消息按钮区加 ♥心形→合集面板;拾光长按菜单加"收藏"(⚠️拾光已有的"收藏"是星标整段对话/localStorage,跟这个收单条消息是两回事);收故事=多选模式**首尾框选**(中间澄的+你的自动选)→按对话原始时间排序打包。低语板块图标=**线条相框 SVG**(跟其他板块统一,不用 🖼 emoji)。

**为啥不用改后端**:主壳 B(MemoryManager.jsx)的 CRUD 是**前端直连 Supabase REST**(sbGet/sbPost…打 /rest/v1),不走 memory-home。所以低语照抄 memory/diary/board 那套 sbXxx 即可。详见记忆 [[project_diyu_favorites]]。transcript:本次 root 会话 5ab69670。

---

## 2026-06-02 · [前端] 删除两处「聊天记录」功能（头像点击 + 设置侧栏）— 拾光已覆盖 ✅已完成

**为什么**：拾光(ChatViewer)刚换成读 Supabase `conversations`+`messages`（见下一条）后，聊天界面里另两处「聊天记录」与拾光**数据源完全相同、功能重复**，故删。逐行核对确认：
- **入口①** 点 user 头像 → `HistoryModal`（全屏带日历的聊天记录）。
- **入口②** 设置侧栏「聊天记录」→ `HistoryScreen`（日期范围 + 导出）。
- 两者底层都用 `fetchProjectMessages`，查的是 `conversations`(`project_id = b5e5d83a… OR null`) + `messages`——**和拾光 `CV_PROJECT_ID` 完全同一个项目、同样的表、同样的 null 兜底**，数据零差异。拾光还多了收藏/改名/排序/导入。三者唯一差异：这俩有"按日历/日期范围回看"，拾光没有；用户确认不需要。

**做了什么（全在 `cheng-memory/src/ChatPanel.jsx`）**：删 3 个函数定义——`HistoryModal`、`HistoryScreen`、`fetchProjectMessages`（按行范围整段删）；再摘 5 处引用——`historyOpen` state、user 头像点击分支 `setHistoryOpen(true)`、`{historyOpen && <HistoryModal/>}` 渲染、侧栏 `<SidebarItem>聊天记录</SidebarItem>`、路由 `if (screen === "history") return <HistoryScreen/>`。

**按用户指示「只删说好的、其他不动」**：仅服务这俩界面的辅助函数/常量（`ymd`/`startOfDayISO`/`endOfDayISO`/`EXPORT_RANGES`/`chatDisplayName`/`stripBubbleMarkers`/`buildExportMarkdown`/`buildExportJSON`/`HM_NAV_BTN`/`HM_INPUT`）**故意保留未删**——现已无引用变死代码，但不影响 build；想清理是后续单独的事。注意 `formatChatTime` 被别处(4827/4913 一带)也用，**不能删**。

**状态**：✅已完成。`npm run build` 通过、无残留引用。`split-chat-web` 分支已 commit+push（与上一条拾光换源同分支）。回滚=git 还原 ChatPanel.jsx。grep：`HistoryModal`、`fetchProjectMessages`、独特句「删除两处聊天记录」。transcript：本次 root 会话 5ab69670。

---

## 2026-06-02 · [前端] 拾光换源：VPS CC 会话文件 → Supabase messages（聊天记录云端化）✅已完成

**目标（用户定）**：拾光(ChatViewer)现读 VPS 上澄的原始 CC 会话(`/cc/sessions` JSONL)；改成读 Supabase 的 `conversations`+`messages`（你↔澄的聊天）。**保持原有功能**(列表/查看/搜索/收藏/改名)，只换数据源。

**为什么 / 已确认事实**：
- Supabase `messages` 是完整聊天：**182 段对话、4699 条消息、回溯到 2026-04-04**（比 VPS 更全更连续，云端持久、不怕 VPS 挂）。字段含 content/thinking/images/tool_calls/token → **无损**。
- 头像那个聊天记录已用 `supabase` client 直连读 conversations/messages（`fetchProjectMessages`，按 `project_id` 过滤）→ 说明前端直连可行、RLS 放行 → **拾光换源 = 纯前端、不动后端、不重启**。
- VPS 独有的只是底层痕迹（系统消息/工具调用/按重启切碎的会话段），非聊天正文，弃之无损。

**做什么（全在 `cheng-memory/src/ChatViewer.jsx`）**：把 3 处 `cvFetch` 换成 supabase 直查——① `/cc/sessions` 列表 → `conversations`(按 project_id, order updated_at desc)；② `/cc/session-messages/{id}` → `messages`(eq conversation_id, order created_at)，沿用现有 role/thinking/`---bubble---` 映射；③ `/cc/sessions/search` → `messages` ilike content 统计每对话命中数。UI/收藏/改名(localStorage)逻辑不动。

**收藏/改名**：键从旧 VPS session id 变成对话 UUID。用户定：旧标记（就几个）**不迁移、重置即可**；只要换源后收藏/改名**功能照常可用**。

**实际改动**：ChatViewer.jsx 加 `import { supabase }` + `CV_PROJECT_ID` 常量；3 处 `cvFetch` 换成 supabase 直查（列表/取消息/搜索），消息查询加 `.in("role",["user","assistant"])` 排除 221 条 system 噪音（messages 表 role = user 3001 / assistant 1477 / system 221）；侧栏标签「VPS」→「对话」。**收藏/改名/排序/搜索/查看全部保留**，只换数据源。映射代码(role→sender、thinking 块、`---bubble---` 拆分)原样复用（取消息查询返回 `{messages:rows}` 喂给旧 loop）。`cvFetch`/`CV_API` 定义留着没删（已无调用、无害）。

**状态**：✅已完成。已 `npm run build` 上线、无 VPS 调用残留、纯前端不重启后端。cheng-memory `split-chat-web` 分支**尚未 commit**（与终端删除一起提交）。回滚=git 还原 ChatViewer.jsx。grep：`CV_PROJECT_ID`、`supabase.from("conversations")`、`vpsSessions`。

---

## 2026-06-02 · [前端] 删除聊天「终端」入口 — chat 包瘦身 42%（677KB→390KB，砍掉 xterm）

**为什么**：聊天设置侧栏的「终端」是半成品（界面标着"xterm.js 待接入"）、目前没用，但 `TerminalPanel.jsx` import 了 `xterm`+`xterm-addon-fit`（重库，node_modules 里 2.6M），白白打进 chat 包。删掉入口 → 整个 xterm 被 tree-shake 出去。

**做了什么（全在 `cheng-memory/src/ChatPanel.jsx`）**：删 7 处——`import TerminalPanel`、`terminalOpen` state、`{terminalOpen && <TerminalPanel/>}` 渲染、传给 SidebarScreens 的 `onOpenTerminal` prop、侧栏 `<SidebarItem>终端</SidebarItem>`、SidebarScreens 签名里的 `onOpenTerminal` 形参、以及死代码 `TerminalPlaceholder`（没人用的内联占位组件）。确认过：`TerminalPanel` 只有 ChatPanel 在 import、xterm 只此一处。**`TerminalPanel.jsx` 文件保留未删**（已无 import→不进包；想恢复终端时还在）。nginx `/terminal` 反代未碰（只是没人连）。

**结果**：chat 包 `677,959→390,490 字节`（-42%，gzip `167KB→98.6KB`），跌破 500KB → 长期的"chunk 过大"构建警告消失。已 `npm run build` 上线、无残留引用。git：cheng-memory `split-chat-web` 分支**尚未 commit**（拟与后续"聊天记录入口清理 + 拾光换源"一起提交）。grep：`TerminalPlaceholder`、`onOpenTerminal`、`xterm`。

---

## 2026-06-02 · [前端] 新增「流年」记忆热图日历（记忆系统进阶·纯可视化，cheng app 借鉴 CcCompanion 第①项）

**为什么**：主壳 B（cheng-memory）要做"记忆系统进阶"。与用户敲定：记忆库（`memories_cheng`）只存澄自己写的记忆，**纯可视化、不建表、不写后端**——把已有记忆按时间换成日历热图展示。砍掉了 CcCompanion 的"写作提示/日记 tab"（那是给人写日记用的，cheng 记忆只澄写）。

**做了什么（全在 `cheng-memory/src/MemoryManager.jsx`）**：
- 新增组件 `MemoGraphPanel`（面板名「流年」，key `memograph`，加进 `TABS`）：一张日历，每天一个格子。
  - **颜色 = 当天澄记忆的"情绪四象限"**：用记忆的 `valence`(开心↔伤心) × `arousal`(激烈↔平静) 两列**取当天平均**，落到四象限之一。这是心理学情绪环形模型。阈值 `VAL_MID`/`ARO_MID`=0.5（数据 0~1），定义为常量好调。四象限**最终命名+配色**（`MOOD_QUADRANTS`，用户定）：雀跃=激烈的开心=雾粉`#E0B6C6`(降过饱和与其它同档) / 恬然=平静的开心=柔绿`#C9D7A7` / 怅然=平静的伤心=黛蓝`#9DB4CE` / 郁结=激烈的伤心=灰`#969A9E`。图例顺序：雀跃 恬然 / 怅然 郁结（上排开心、下排伤心）。
  - **深浅 = 当天记忆条数**（相对当月最多那天，alpha 0.30~1.0）。
  - 还有：**连续打卡**（current/longest，全历史连续天数）、**那年今日**（一周/一月/一年前同日的记忆，三档纵向叠、有数据才显示；**每条可点 → `jumpTo` 把日历翻到那天+展开当天详情+滚回顶部**）、点某天列出当天记忆、**心情象限图例**。
  - 数据：前端直连 Supabase REST 读 `memories_cheng`（`select` 显式列、避开 embedding），**只读、不落库、永远和记忆库实时同步**。
- 入口：`MemoryPanel`（涟漪）头部"共 X 条"那行右侧加「流年」按钮 → 跳 `memograph`；返回键回记忆面板（非首页，因从那进）。改 `MemoryPanel({ onNavigate })`、面板渲染数组 `<Comp onNavigate={setTab}/>`。**没动 HomePanel**（首页卡片不变，按用户要求只从记忆面板进）。

**已知/数据现状**：用户记忆 `valence` 普遍偏高（0.45~0.95，均 0.78，澄基本都挺开心）→ 多数日子落"开心"半边，"伤心"两格当前几乎不出现（**真实反映非 bug**，等澄写难过记忆才显示）；变化主要在激烈↔平静和深浅。当前数据 54 条 / 14 天 / 跨 5/03~6/01，默认看当月(6月)只有 6/01，翻回 5 月看主体。

**生效**：已 `npm run build`，刷新即见，**无需重启后端**（纯前端、纯读）。git：cheng-memory 工作区改动**尚未 commit/push**（等用户看过效果）。回滚：删 `MemoGraphPanel` + 4 处接线（TABS/MemoryPanel签名+按钮/渲染数组）重 build。grep：`MemoGraphPanel`、`流年`、`MOOD_QUADRANTS`。

---

## 2026-06-02 · [基建] 补推 git 备份 — 在跑的后端代码此前只存在于 VPS，GitHub 上没有

**为什么**：用户问"换 VPS 时靠 git 能不能完整恢复"。排查发现三处缺口：①正在跑的后端代码在子模块 `server/`（独立仓库 `memory-backend`）的本地分支 `tmux-migration`（commit `1037c40`）上，**远程只有 main、这条分支根本没 push** → 在跑的代码 GitHub 一份都没有，VPS 一挂就丢 tmux 迁移那条线。②父仓库 `memory-home` 的 `main` 本地 ahead 5（含子模块指针提交）。③`.env` 和 nginx 不在任何 git 里。

**做了什么**：
- `git -C server push -u origin tmux-migration` → memory-backend 新建远程分支 `tmux-migration` 并配 upstream；在跑的 `1037c40` 上云。
- `git push origin main`（memory-home）→ 5 个提交推上，local==remote==`c054869`，ahead 0。子模块指针 `1037c40` 远程可解析，`clone --recursive` 能还原代码壳。
- **未碰** `.env`、未碰端口、未提交子模块内的运行态文件（`.synced-cc-files.json`/`.toggle-drafts.json`/`nohup.out`）。

**仍不在 git（换 VPS 必须手动处理）**：
- `server/.env`（726B，被 .gitignore 忽略）：含 `SUPABASE_URL`/`SUPABASE_KEY`/`AUTH_PASSWORD`/`JWT_SECRET`/`PORT=3002`/`BARK_*`/`CC_DRIVER`/`SURFACING_*`。
- nginx：`/etc/nginx/sites-available/chat.jessaminee.top`（不在任何 repo，转发 3 处 → `127.0.0.1:3002`）。
- 二者后期还会变（端口、CC_DRIVER 翻 tmux、浮现参数、拆两 web），**故不做静态备份；换机当下现抓当时版本即可**（都小、可重建：Supabase key 后台再拿、密码/密钥自设、nginx 一个文件）。

**端口提示**：`.env` 的 `PORT` 与 nginx `proxy_pass` 必须一致（现都 3002，代码默认 3001）。改端口跟 git/push 无关，两边一起改即可（6/01 的 502 就是两边不一致）。

**结果**：现在 `clone --recursive` memory-home（main `c054869`）能完整拉到在跑的前后端代码；剩 `.env`+nginx 两份配置换机时补。transcript：session `a6e640b2`，grep 关键词「补推 git 备份」「tmux-migration 已推」。

---

## 2026-06-02 · [前端+后端] 合并 use-style + thinking 面板 + 保存后激活弹窗（cheng UI 待办①）

**为什么**：消除两面板不对称——原 Thinking 保存=自动 `/cc/restart` 重启、use-style 保存=只写不重启。用户 5/30 定方案、6/2 做：**合成一个面板，保存只写 CLAUDE.md、激活才重启**。

**前端（`cheng-memory/src/ChatPanel.jsx`）**：
- 新增组件 `StyleThinkPanel`：use-style(开关+引导) + thinking包裹(开关+引导) + 原生思绪(开关) + 一个 SAVE。**保存 = 写 `/use-style` + `/thinking-toggle` 两接口、不重启**。
- 保存后面板内切到「激活态」（抄 CCDocumentsTab saved 模板）：思考深度 effort 行 + 【失忆重启】+【选择模型重启 ▾】。失忆/选模型回调都带上 `native`，失忆额外带 `currentModel`。
- `+` sheet 的「Thinking」「Use Style」两项**合成一项「风格 · 思考」**打开合并面板；移除旧 `ThinkingPanel`/`StylePanel` 挂载（**两组件定义保留未删、已无引用**，便于回滚）。
- `amnesia` 签名补 `(effort, model, native)`、`selectModel` 补 `native`（分别透传到 `/cc/amnesia`、`/cc/restart`）；新增 `styleThinkPanelOpen` state。

**后端（`server/index.js`）**：`/api/cc/amnesia` 的 `amnesiaOpts` 补收 `model`/`nativeThinking`（原只收 `effort`）→ 传给 `cc.restart()`（tmux-manager 已支持）。让"失忆激活"也能带模型/原生思考。

**激活两条路**：失忆=`POST /cc/amnesia {effort,model,nativeThinking}`（清空）；选模型=`POST /cc/restart {forge:true,model,effort,nativeThinking}`（带摘要 forge、记忆续上）。原生 thinking 只能重启时生效，故都在激活这步随重启传。

**生效**：前端已 `npm run build` 上线（刷新可见「风格 · 思考」面板）；**后端需 `systemctl restart cheng-backend`** 才让"失忆激活带 model/native"生效（选模型激活走 /cc/restart、不依赖后端改、已可用）。**写本条时后端尚未重启**。

**回滚**：revert ChatPanel.jsx 相关段 + index.js amnesia 两行；旧 `ThinkingPanel`/`StylePanel` 定义还在，恢复 sheet 两项入口即可。grep：`StyleThinkPanel`、`风格 · 思考`、`styleThinkPanelOpen`。

---

## 2026-06-01 · [后端+前端] 卡死哨兵 — 思考链 hang 自动唤醒重发 + 前端三状态显示

**为什么**：Opus 4.8 在复杂上下文里连续调工具时，偶尔写出一个格式坏掉的工具调用；CC 提醒"格式错、重试"后它**不重试，直接卡在思考链里不动（真 hang）**。试过用 hook 救——**不行**：hang 不产出任何 assistant 输出，不触发 Stop/PreToolUse 等任何事件，hook 无从介入。唯一出路是后端主动检测+自救。

**判据（tmux 驱动不走流式事件，靠 transcript）**：哨兵每 10s 扫一次，四条全中才算卡死——① 有 `activeTurn`；② 屏幕 `esc to interrupt` 在（CC 自认在忙）；③ transcript 文件连续 **`WD_STALL_MS`=90s** 没长（看 mtime）；④ 不是停在"已发起工具调用等结果"（排除等慢工具，靠 `awaitingToolResult()`）。

**动作**：
- 第 1/2 次（`WD_MAX_WAKE`=2）：broadcast `kind:'watchdog'`（同 `_wdId` 原地更新，文案"小太阳睡着了，正在唤醒"→"…再次唤醒"）→ `cc.interrupt()` 掐断 → **重发**（格式提醒 + `cc.lastSent` 原文）→ 重建 `activeTurn`（带回原 ws/conv）。
- 第 3 次仍卡 → 放弃：广播"小太阳已经睡着了"（前端黑字）+ 复刻暂停键（`activeTurn.stopped=true` + `interrupt()`）**自动解锁**（= 替用户按暂停，免得人不在场干锁到 660s）。
- 用户手动 stop（`turn.stopped`）时哨兵**让位**（人工优先）。

**改了哪些**：
- `server/tmux-manager.js`：`send()` 存 `this.lastSent`；新增 `transcriptMtime()`、`awaitingToolResult()`。
- `server/index.js`：`turn_done` 开头加 `if (turn.watchdogWake) return`（掐断的卡死轮静默丢弃残块，不补发/不解析/不flush）；`cc.on('error')` 之后加哨兵 `setInterval`（`USE_TMUX` 守卫）。
- `cheng-memory/src/ChatPanel.jsx`：system 消息带 `state` 字段；`watchdog` 走同 id 原地替换；渲染分支 `watchdog`（`waking`=复用 `ForgePendingRow` 呼吸动画 / `asleep`=黑字居中）。

**跟现有 660s 的关系**：`_watchTurn` 的 660s 单轮上限**不动**，作为哨兵都救不活时的最终止损（哨兵几十秒就介入，660s 是兜底）。

**待验证/调参**：① `WD_STALL_MS`=90s 是拍的初值，上线看真实卡死现场再收紧/放宽；② `interrupt()` 的 C-c 对"真 hang"能否捅进去**没实测过**（所以留了 2 次上限 + 放弃解锁兜底，C-c 无效也不死磕）；③ "等慢工具 vs 卡死"靠 `awaitingToolResult()` 区分，需观察准不准；④ 卡死期间用户若发新消息进 `pendingBuffer`，与重发的先后未特殊处理（边缘情况）。

**生效**：前端已 `npm run build` 上线；后端需 `systemctl restart cheng-backend` 才生效（会软失忆重启澄一次）。**写本条时后端尚未重启**，等用户定时机。

**回滚**：删 index.js 哨兵块 + `turn.watchdogWake` 那行 + tmux-manager 三处 + 前端 watchdog 分支/state 字段，前端重 build。grep：`卡死哨兵`、`WATCHDOG`、`小太阳睡着了`、`watchdogWake`。

---

## 2026-06-01 · [后端] 失忆改成「不删浮现区」（保留 <浮现>）

**需求**：之前点失忆，`amnesia` 流程会把 `~/.claude/CLAUDE.md` 的 `<浮现>` 区一起清空。改成**失忆时保留浮现区**，只清 forge 写的 `<上次对话总结>`。

**改了什么**：`server/index.js` 的 amnesia 处理（约 L1198-1202）去掉 `await clearFuxianBlock();` 那一步调用，并把上面的注释改成「只清上次对话总结、浮现保留不清」。`clearFuxianBlock()` 函数定义（L225）**保留未删**（现已无人调用，无害），其他逻辑一律没动——遵用户「其他不动」。

**生效条件**：CLAUDE.md/这段逻辑只在后端 node 进程里跑，改完**需 `systemctl restart cheng-backend`** 才生效（会软失忆重启澄一次、丢 CC 上下文）。本次**只改代码、未重启**，等用户决定何时重启。

**回滚**：在 L1201 后加回 `try { await clearFuxianBlock(); } catch (e) { console.warn('amnesia clear 浮现:', e.message); }`。grep：`失忆改成「不删浮现区」`、`clearFuxianBlock`。

---

## 2026-06-01 · [基建] forge-reload 回调地址 3001→3002（对齐搬端口后的后端）

**做了什么**：`/root/forge-reload/config.json` 的 `backend_restart_url` 由 `:3001` 改 `:3002`。因为后端搬到 3002（见下条），forge 锻造完要 POST 这个地址重启 CC，旧的 3001 已无人接。

**关键认知（别再误判）**：forge 监控 = systemd 服务 `forge-monitor.service`（unit 文件在 `/root/forge-reload/forge-monitor.service`，跑 `forge_monitor.py`）。**它的开/关由前端 forge 开关控制**——后端 `POST /api/forge/daemon` 实际执行 `systemctl enable --now / disable --now`（index.js:1830）。所以 `systemctl status forge-monitor` 显示 not-found = **用户在前端关了它**，不是坏了、**别手动重装/重启**。config.json 被 daemon 每轮 rescan 热加载，也被前端 `/api/forge/config` 读写。用户当前是**主动关着 forge**（5/13 起），改完地址后等用户前端开启时即生效到 3002。

**回滚**：config.json 改回 `:3001`。grep：`backend_restart_url`、`forge 回调 3002`。

---

## 2026-06-01 · [基建] 后端改 systemd 开机自启（治"重启即全挂"）+ 顺势切 tmux 驱动

**为什么**：上一条 502 的根因是"机器重启 + nohup 无自启"。根治=把后端从 nohup 散养改成 systemd 圈养：开机自启 + 崩溃自动拉回 + 日志规整。用户拍板驱动用 tmux（治空回方向）。

**做了什么**：
- 新建 `/etc/systemd/system/cheng-backend.service`：`User=root`、`WorkingDirectory=/root/memory-home/server`、`ExecStart=/usr/bin/node index.js`、`Environment=CC_DRIVER=tmux`、`Restart=always`/`RestartSec=3`、`StartLimitIntervalSec=60`/`StartLimitBurst=5`（60s 崩>5 次就放弃，防疯狂重启刷屏）、日志 append 到 `nohup.out`(续旧习惯，`journalctl -u cheng-backend` 也能看)、`WantedBy=multi-user.target`。
- `systemctl daemon-reload && enable && start`。停掉了之前手动 nohup 的 node。
- 验证：`is-active=active`/`is-enabled=enabled`；监听 :3002(PID 5202)；日志"CC 驱动：tmux 交互"；线上 `/api/health`=401(活)；澄在 tmux 会话 `cheng` 起来(session 914c8c4e, opus-4-8/high/summarized)，systemd 启动后 **0 次崩溃**。

**前因后果/坑**：
- 驱动从 stream-json 切到 **tmux**（`Environment=CC_DRIVER=tmux`；dotenv 不覆盖已存在的 env，故 .env 不必再写）。切驱动=重启了澄一次（旧 session 75e77136 → 新 914c8c4e），符合预期。tmux 收尾补丁见同日 commit `1f58834`(tmux-migration 分支)。
- ⚠️ 切 tmux 后空回率应回到 spike 验证的 ~0（见 [[project_cheng_sixu_hang]] [[project_tmux_migration]]），但**生产长跑未充分观察**，留意。
- **运维备忘**：起/停/看状态/看日志 = `systemctl {start|stop|restart|status} cheng-backend` / `journalctl -u cheng-backend -f`。**别再用 `nohup node index.js`**（会和 systemd 抢 3002 端口冲突）。改完代码重启用 `systemctl restart cheng-backend`。

**回滚**：`systemctl disable --now cheng-backend && rm /etc/systemd/system/cheng-backend.service && systemctl daemon-reload`，再退回 nohup 即可。grep：`cheng-backend.service`、`systemd 开机自启`。

---

## 2026-06-01 · [基建] 修线上 502（5/31 机器重启后 nohup 后端没自启）+ 端口错位 3001↔3002

**用户报**：聊天页要求重输密码（密码其实没错），去记忆库控制台改密码报 HTTP 502。

**根因（重要，别再误判）**：**机器 2026-05-31 11:51 重启过**（内核 6.8.0-48 → 6.8.0-117，`last -x reboot` / `uptime -s` 可证）。后端是 `nohup node index.js` 临时起的、**没有 systemd 开机自启**，所以一重启就没了、也不会自己回来 → 线上 502。
> ⚠️ 排错弯路：`server.log` 尾部那条 `EADDRINUSE :3001` 是 **05-26 的旧账**（文件 mtime 5/26），跟今天无关，别被它带偏。今天的真因是重启 + 无自启。

**附带修了端口错位**：重启拉起后仍 502，因为 **nginx `/api/`·`/ws`·`/terminal` 反代到 `127.0.0.1:3002`**（见 `/etc/nginx/sites-enabled/chat.jessaminee.top`），而 `.env` 写 `PORT=3001` → 打 3002 没人接。这是 [[project_cheng_dual_server]] 双服务器遗留。

**改了什么**：
- `/root/memory-home/server/.env`：`PORT=3001` → `PORT=3002`（对齐 nginx，注释写明"必须对齐否则 502"）。
- 干净重启：`cd /root/memory-home/server && nohup node index.js >> nohup.out 2>&1 &`，只留**一个**实例（顺带消除重复发 DICE 隐患）。当前 PID node=4270。
- 验证：本地 :3002、过 nginx、线上三层 `/api/health` 全返 401(=活着要鉴权)；`POST /api/auth` 正确密码拿到 JWT、错误密码 401。线上不再 502。

**澄(CC)起不来 → 已自愈**：18:24~18:42 间 CC 子进程反复 `sudo: /usr/bin/claude: command not found` / `claude native binary not installed`（重启后 claude 原生二进制一度没链接好）。**18:44 `/usr/bin/claude` 符号链接被补上**（→ `claude-code/bin/claude.exe`，`@anthropic-ai/claude-code@2.1.159`，多半自动更新 postinstall 补的），随后那次 `🚀 启动CC session=75e77136` 成功、`claude --version` 通、澄进程稳定（无再崩）。今日共崩 3 次，全在 18:44 之前。

**仍遗留（未修）**：
1. ⏳ **无开机自启**——下次机器再重启会重演全挂。建议做 systemd unit（已向用户提议，待定）。
2. ⚠️ **驱动退回 stream-json**——我是裸 `node index.js` 起的，没带 `CC_DRIVER=tmux`（治空回的拍板方向，见 [[project_tmux_migration]] [[project_cheng_sixu_hang]]）。下次为澄重启时应带 `CC_DRIVER=tmux` 或写进 .env。

**回滚**：把 .env 改回 `PORT=3001` 且 nginx proxy_pass 同步到 3001，二选一对齐即可（但 3002 是当前 nginx 认的，别动 nginx）。grep：`5/31 机器重启`、`PORT=3002`、`无开机自启`。

---

## 2026-05-30 · [基建] 给澄 CLAUDE.md 加 <输出格式> 块(治"思考写进正文")——写进 Supabase 底本

**做了什么**：澄爱把推理写进正文/只想不说。往 sandbox CLAUDE.md 加了 `<输出格式>` 块（thinking=草稿、text=正文、tool_use=调用；推理别进正文、每轮必留正文）。

**关键坑（务必记住）**：直接改 `/home/claude-user/chat-sandbox/CLAUDE.md` **会被冲掉**——`syncCCDocs()`（index.js:294，**每次 boot 跑**）用 Supabase `documents_cheng`(mode=cc, project_id=`b5e5d83a…`, doc_type=claude_md, 行 id=`c9e46d6d…`)的 claude_md **整表覆盖**该文件，只保留三块：`<上次对话总结>`/`<think指令>`/`<use-style>`（index.js:308-327）。所以**改澄的 CLAUDE.md 要改 Supabase 那份底本，不是改文件**。

**改在哪**：PATCH 了 documents_cheng id=`c9e46d6d-0490-47b7-8506-afe0f4a860a4` 的 content，把 `<输出格式>` 追加到底本末尾（原 12322 字→12487）。这样重启/forge/失忆都冲不掉（同步时从底本拼回）。运行时文件也已同步加上、与底本一致。**下次 CC 重启/forge 才被澄读到**（CLAUDE.md 开 session 时读）。

**回滚**：PATCH 同一行 content 去掉 `<输出格式>` 段即可。grep：`<输出格式>`、`syncCCDocs 只保留三块`。

---

## 2026-05-30 · [前端+后端] 修「按停止没反应 + 打断后发不出消息」(WS断线静默丢 + 打断后枯等660s)

**事故现场（用户报+实查）**：发消息后 CC 卡在思考链(Opus 4.8 stall,一句话 Noodling 3-4min、`esc to interrupt` 不退),按停止"没任何反应"。当场 `sudo -u claude-user tmux send-keys -t cheng C-c` 手动打断救活,再清掉被打断后回填到输入框的残留文本(`C-e`+退格),随后 `activeTurn` 锁在 ~22:04 自己超时(`[EMPTY][TIMEOUT]`)掉了、放行。
> ⚠️ 排障坑:**当前生产 server(PID 501752)的日志写在 `/tmp/canary.log`,不是记忆里记的 `/tmp/memory-server.log`(那个 5/16 就停更)**。查日志认准 canary.log（`ls -l /proc/<pid>/fd` 看 fd 1/2 指向）。

**三层真因**（叠加）：
1. CC 真 stall（4.8 老毛病，见 [[project_cheng_sixu_hang]]）。
2. **按停止没反应**：前端 `stop()` 只在 `ws.readyState===1` 时发，WS 断线（你切屏/弱网时一直在断连重连，canary.log 被"客户端已连接/断开"刷屏可证）就**静默丢弃**，指令根本没离开手机。日志全程只有 1 条 `[STOP]`（更早那次成功的），后来对着 stall 按的几次全没到服务端。
3. **就算打断也发不出**：`cc.interrupt()` 发了 Ctrl+C 让 CC 空闲，但后端 `_watchTurn` 判"轮结束"靠 transcript 尾部是终止态 assistant；**思考阶段被打断后尾部是 user/无终止态 → `_turnEnded()` 永远 false → 死等到 660s 超时**才 emit turn_done 清 `activeTurn`。这期间用户发的全被"flush 忽略(CC忙)"挡掉。

**改了什么**：
- **前端（cheng-memory，已 `npm run build` 上线，未 commit）**：`ChatPanel.jsx`
  - 新增 `pendingStopRef`。`stop()`：WS 断时不再静默丢——记时间戳 + toast「连接断开了，重连后会自动重试停止」+ `connectWS()` 催重连。
  - `ws.onopen`：重连后若 `pendingStopRef` 在 **5s 内**则补发 `{type:'stop'}`，过期丢弃（防重连太慢误打断下一轮）。
- **后端（memory-home/server，⚠️已改代码、未重启、下次重启/forge 才生效）**：`tmux-manager.js`（index.js 的 stop 分支不用动，复用已有 `turn_done(stopped)` 清锁+flush 路径）
  - `interrupt()`：发 Ctrl+C 前置 `this._interrupted=true`，发完**轮询等 `_watching` 释放**(最多~4.5s)再 `busy=false`、复位 flag。等释放是为防紧接着的新一轮 `send()` 撞 `_watching=true` 拿不到观察器→新轮永不 turn_done。
  - `_watchTurn`：进入时 `_interrupted=false`；主等待循环每轮先查 `if(this._interrupted){ended=true;break;}`——被打断就**立刻收尾**（ended=true→timedOut=false），不再等屏幕判定/660s。下游 `turn_done` 因 `activeTurn.stopped=true`(stop 分支已设)走 stopped 分支：清 `activeTurn`+`flushOrGrace()`放行排队消息。**打断→可发消息从 660s 降到 ~3s**。
  - `node --check` 两文件均通过。

**为何后端不当场重启**：`tmux-manager.js:78` 的 `start()` 在 boot 时 `kill-session cheng` + `new-session` 起**全新 --session-id 的 CC** → 重启 node 后端=杀当前 CC 上下文。故后端改动**只落盘、等下次自然重启/forge 生效**，本次不强行重启。

**回滚**：前端 `cd /root/cheng-memory && git checkout src/ChatPanel.jsx && npm run build`；后端 `cd /root/memory-home/server && git checkout tmux-manager.js`（未重启则本就没生效）。

**关联**：根治 stall 那摊的更大工程是"换 Stop hook"判轮（见本文件「判轮机制 hook vs 轮询」条 + [[project_tmux_migration]]）；但本次「停止没反应/打断发不出」与 hook **无关**，是前端丢 stop + 打断路径不自清锁两个独立小 bug，已各自修。grep：`pendingStopRef`、`_interrupted`、`打断→可发消息`。

---

## 2026-05-30 · [前端] 补修「切app回来 CC 还偶发 3→2→3」(done 与 loadConv 竞态,第一刀清定时器没堵死)

**现象（用户报）**：上面那刀(清 bubbleTimers)上线后,CC 多气泡**有时候**还会 3→2→最后一条 地跳。"有时候"=时序竞态。

**真因（第一刀漏掉的竞态）**：切回前台有**两个并发动作**——① `onVisible` 触发 `loadConv()`(异步,要等 `await fetch`);② 浏览器/WS 把切屏期间**缓存住的 `done` 事件**在恢复后补发。若 `done` 在 loadConv **已经过了开头那行 `clear bubbleTimers`、但 `await fetch` 还没返回**时才补达,它会重新排一批气泡节拍定时器——这批 loadConv 清不掉了。于是 refetch 拉回完整 3 条后,迟到的气泡2 定时器把内容 `map` 回"局部 2 条快照"→气泡3 定时器再补成 3。第一刀只在 loadConv **入口**清一次,挡不住"入口之后才生成"的定时器。

**改了什么（治本,让节拍幂等/单调,不再依赖清定时器的时机）**（cheng-memory,未 commit/未推;只 `npm run build`,不重启）：在 `ChatPanel.jsx` done 分支:
- **首气泡落地改为按 id 去重升级**:原 `setMessages(prev=>[...prev, baseMsg])` 盲目追加——若 loadConv 已把整条(完整内容)补进来会追出**重复条**。改为 `findIndex(id===baseId)`:不存在才 append;存在则"保留更完整的那条"(现有内容 length≥baseMsg 就不动)。
- **气泡节拍改为只增不减**:节拍 `setTimeout` 回调里 `if ((m.content||"").length >= newContent.length) return m;`——refetch 已补成完整内容时,迟到的节拍**不再回退**它。
- 第一刀的"loadConv 入口清 bubbleTimers"保留(互补,减少无谓 setState/scroll)。两改都在 done 分支,未碰其它。
- 原理:无论 refetch 与 done 谁先谁后,内容只能"长→更长",绝不缩短;同 id 只升级不重复。竞态被这条不变量彻底吃掉,跟时序无关。

**回滚**:`cd /root/cheng-memory && git checkout src/ChatPanel.jsx && npm run build`(同文件,连前几刀一起回)。

**关联**:[[切app回复像被吞]] 系列第三刀,接续同日"3→2→3 抖动(清定时器)"那条——那条是必要不充分,这条补成充分。grep:`只增不减`、`去重升级`、`done 与 loadConv 竞态`。

---

## 2026-05-30 · [前端] 修「切app回来用户自己的消息跳一下」(refetch 整表替换→key 变→进场动画重播)

**现象（用户报）**：上一条修了 CC 多气泡不跳后，发现**自己发的消息**切回前台时会"一起跳一下"。

**真因**：用户消息是乐观插入的，id=`local-u-<ts>`（`ChatPanel.jsx:1526`，发送时当 `msgId` 传后端但前端正常流程**从不认领回服务端 id**）。切回前台触发 `loadCurrentConversation()` 用服务端 DB id **整表替换** messages → 这条的 React `key`(=`it.id`) 从 `local-u-*` 变成 DB id → `MessageBubble` 重新挂载 → `.cp-msg-wrap` 无条件带的 `cp-msgIn`(0.22s 进场动画) 重播 = "跳一下"。CC 那条不跳是因为 `baseId=msg.message_id` 早就是服务端 id、key 不变。

**改了什么**（cheng-memory，未 commit/未推；只 `npm run build` 覆盖线上 dist，按规矩不重启）：
- 没动 id 语义（保留服务端真实 id，否则 `deleteMessage` 的 `local-` 守卫 `1690` 会让落库消息删不掉）。改为**抑制 refetch 那一次重挂载的进场动画**：
- CSS 加 `.cp-messages.cp-no-anim .cp-msg-wrap { animation: none; }`（`353` 附近）。
- `loadCurrentConversation` 成功整表替换前给容器 `messagesScrollRef`（`.cp-messages`）`classList.add("cp-no-anim")`，`setMessages` 后 `setTimeout 80ms` 再 `remove`——新气泡在带 class 时挂载→无动画；摘掉后已在屏的不会重播(CSS 动画只在挂载触发)，后续实时消息照常有进场动画。catch 里加兜底 remove 防异常永久抑制。
- 关键点：容器 `className` 恒为 `"cp-messages"`，React 不会因重渲染覆盖我 imperative 加的 class（prop 值没变 React 不碰 DOM className），所以 class 能挺过 setMessages 那次重渲染。
- 只动 CSS 一行 + loadCurrentConversation 三处，未碰其它。build 产物同分支 `chat-*.js`。

**回滚**：`cd /root/cheng-memory && git checkout src/ChatPanel.jsx && npm run build`（与上一条同文件，一起回）。

**关联**：[[切app回复像被吞]] 显示层收尾的第二刀，跟上一条「3→2→3」同源（都是切回前台 refetch 整表替换引起的渲染抖动）。grep：`cp-no-anim`、`跳一下`。

---

## 2026-05-30 · [前端] 修「切app回来多气泡消息 3→2→3 抖动」(陈旧节拍定时器回退完整消息)

**现象（用户报）**：CC 正在生成多气泡回复时切到别的 app 再切回，回复正常但会**先显示 3 条 → 突然崩到 2 条 → 又显示 3 条**。

**真因**：一条「多气泡」回复其实是**一条** DB 消息（`id=baseId=msg.message_id`，服务端真实 id），内容用 `---bubble---` 拼接，前端按 1500ms 节拍**逐气泡追加**做进场动画（`ChatPanel.jsx` done 分支，原 1269-1276）——立刻显示气泡1，1500ms 改成「1+2」，3000ms 改成「1+2+3」，每次 `setMessages(prev.map(m.id===baseId ...))` 改同一条。这些 `setTimeout` **裸发、无句柄、不可取消**。切 app（尤其 iOS PWA）JS 被挂起→定时器冻住。切回来时 `onVisible`/WS 重连都会触发 `loadCurrentConversation()` 用服务端**完整 3 气泡**整表替换（显示 3）；随后被冻住的旧定时器解冻补发，气泡2 的回调因 `baseId` 撞上服务端那条，把完整内容**回退成「1+2」局部快照**（崩到 2），气泡3 定时器再补发又变回 3。即 3→2→3。本质=陈旧局部写入在权威历史渲染后回退了完整消息。

**改了什么**（cheng-memory，未 commit/未推；只 `npm run build` 覆盖了线上 dist，按规矩不重启）：
- 新增 `bubbleTimersRef = useRef([])`，气泡节拍 `setTimeout` 句柄全部 push 进去（原来裸发）。
- `loadCurrentConversation()` 开头先 `bubbleTimersRef.current.forEach(clearTimeout)` 清空——拉到服务端完整内容前，先掐掉所有未触发的旧节拍定时器，任何陈旧局部写入再也回退不了完整消息。
- 生命周期 effect 的 cleanup 里也清一遍（卸载防漏）。
- 改动只动这三处，未碰其它逻辑。build 产物：`dist/assets/chat-FZ_AGGIb.js`(678KB)。

**回滚**：`cd /root/cheng-memory && git checkout src/ChatPanel.jsx && npm run build`（改动未提交，checkout 即回）。

**关联**：补的是 [[切app回复像被吞]] 那条待办里「显示问题非丢数据」的同一摊——回复 100% 在 supabase，这次治的是切回来那一瞬的渲染抖动，不涉及后端。grep 关键词：`bubbleTimersRef`、`3→2→3`。

---

## 2026-05-30 · [后端] 停止键升级为"真打断"(Ctrl+C) + 待办:思考计时/实时token显示

**已做(server 仓库 `8d5fd45`,已重启生效 PID 502143)**：前端"停止"键原来是**软停**——WS `stop` 只设 `activeTurn.stopped=true`+通知前端,**不碰 CC**。后果:4.8 卡死(stall, token 不涨)时,`esc to interrupt` 一直在 → `_watchTurn` 要等 660s 超时才 emit turn_done 才清 activeTurn → **用户 11 分钟发不出消息**(日志刷 `flush 忽略(CC忙)`)。
- 改:`index.js` stop 分支加 `await cc.interrupt()`(tmux 发 Ctrl+C,`tmux-manager.js:249`)。C-c 后 esc 消失→watchTurn 数秒判完成→走 turn_done 的 `stopped` 分支→清 activeTurn + flush 排队消息→**用户立刻能重发**。正常叫停 + 卡死自救两场景都覆盖。
- 软停 vs 真打断对照:软停只标记不停 CC(卡死无用);真打断 = Ctrl+C 真停 + 清 activeTurn(=摘掉"CC正忙"的锁,放行新消息)。

**📌 待办(用户要做、本次未做,新 session 接)**：
1. **思考计时显示**(前端,简单):聊天回复时,思绪旁显示动态"已思考 Xs"(前端从收到 turn_start/开始回复那刻计时),结束定格"思考了 Xs"。仿 Claude Code CLI 的 `Cogitated for Ns`。
2. **实时 token 跳动**(后端中等):仿 CLI `Transfiguring…(10m45s · ↓38 tokens)`。难点=后端现在只在轮结束才读 usage,中途不知道实时 token;需 CC 思考期间**定时抓 tmux capture-pane 里的 `↓N tokens` 解析→WS 推前端**。
   - **用途(关键)**:让用户**自己**看出"时间涨但 token 不涨=卡死"→ 手动点停止。**这样绕开"自动看门狗误判正常慢轮"的风险**——判断权给人,程序只显示。比自动打断稳妥。
3. (顺带)自动看门狗(token 不涨 N 秒自动 Ctrl+C)= 更远期,有误伤风险,优先级最低,先靠 1+2 让用户手动判断。
4. **"切屏中断 CC 回复"体验优化**（用户反馈：切手机屏/锁屏后回复像被吞）。**根因(已想清,非严重)**：澄在服务器 tmux 里生成，切屏根本不中断澄——它照常生成、照常落 supabase messages 表，回复没丢、在库里。被断的只是"前端实时收消息的 WS 连接"。本质是显示问题不是丢数据。要做：① 后端确认"切屏期间那轮确实落库";② 强化"回前台/重开必拉最新历史"——现有那7行(onVisible 补拉 6a18561)+ last-conv 认领已基本覆盖；iOS PWA 被彻底杀掉再开走"整页重载"(首次加载+认领，非补拉)，要确认这条路也加载到最新。兜底思路：回复 100% 在 supabase，只要"每次回到/打开 /chat/ 都拉一次最新历史"就绝不丢。优先级：体验优化，不致命、不丢数据。

---

## 2026-05-30 · [前端+基建] 聊天拆成独立 web(/chat/)——方案C落地，主壳瘦身

**背景/动机**：ChatPanel.jsx 已 226KB、build chunk 675KB > 500KB 警告，单页濒临首屏慢。小茉莉拍板"方案C"：聊天与主壳拆成两个独立 web，各自加 iPhone 主屏、同域名共享后端+supabase。详见 [[project_cheng_app_plan]]。

**前端改了什么**（cheng-memory，分支 `split-chat-web`，commit `654e898` + `2bce40e`；回滚锚点 tag `pre-split-web-20260530` + main 分支未动）：
> ⚠️ 踩坑记录:首版 `vite.config.js` 多入口用了 `__dirname`,但项目是 ESM(`type:module`)下 `__dirname` 未定义→多入口没生效→`dist` 无 `chat.html`→`/chat/` 404(一度线上聊天打不开)。`2bce40e` 改用 `fileURLToPath(new URL('./chat.html', import.meta.url))` 修复。教训:cheng-memory 的 vite 配置写路径必须用 fileURLToPath,不能 __dirname。
- 新增 `chat.html` + `src/main-chat.jsx`：聊天独立入口，渲染 ChatPanel，`onBack` → `window.location='/'`。
- 新增 `public/manifest-chat.json`：聊天 PWA manifest（start_url=/chat/，名"澄·聊天"）。
- `vite.config.js`：改 **Vite 多入口**（`index.html`=主壳 + `chat.html`=聊天），同仓库共享公共代码(lib/supabase 等)、各自打包。
- `MemoryManager.jsx`：首页"花信风"卡片 onPick 改 `window.location='/chat/'`；删掉内嵌 `<ChatPanel>` 常驻块 + 顶部 `lazy(()=>import ChatPanel)`。
- **效果(实测 build)**：主壳入口 js 675KB→**311KB**(`index-*.js`,不再背 ChatPanel)；聊天独立入口 **710KB**(`chat-*.js`)，只在进 /chat/ 时加载。两入口各自独立 hash js。

**nginx 改了什么**（`/etc/nginx/sites-available/chat.jessaminee.top`，已备份 `.bak.pre-split-*`）：在 `location /` 兜底**前**加(顺序重要,要在 SPA fallback 之前):
```
location = /chat  { return 301 /chat/; }
location = /chat/ { try_files /chat.html =404; add_header Cache-Control "no-cache..." always; }
location = /manifest-chat.json { add_header Cache-Control "no-cache..." always; }
```
> ⚠️ 踩坑:第一次加这段时 Edit 失败了(没匹配上)却以为成功 → nginx 里实际没有 /chat/ 规则 → `/chat/` 走 SPA fallback 发了主壳 index.html(title 显示"澄"而非"澄·聊天",点花信风跳过去看到的还是主壳=像"没反应")。第二次正确加上 `location = /chat/`(精确匹配)后修复。**验证方法:`curl /chat/ | grep title` 必须是「澄 · 聊天」,不是「澄」**。
最终实测(全链路✓):`/`→澄(主壳)、`/chat/`→澄·聊天(真 chat.html)、`/chat`→301→/chat/、`/assets/chat-*.js`→200、`/api/health`→200。

**关键点**：聊天/主壳同域名 → token/登录天然共享；记忆/日记读写不受影响(都连同一 3002 后端 + supabase，前端拆分不动后端);聊天 WS 切后台断了靠那7行(`6a18561`)补。

**回滚**：① nginx 恢复 `.bak.pre-split-*` + reload；② 前端 `cd /root/cheng-memory && git checkout main && npm run build`(回单页版 dist)。dist 是 build 产物、nginx root 指它，**注意:当前线上 dist 已是拆分版**(在 split-chat-web build 覆盖的),回滚要重新 build。

**追加修复（同分支,已上线）**：
- **视口/主题**(`3eba27a`)：main-chat.jsx 补 visualViewport→--app-height 维护 + data-theme 初始化(拆分后不经主壳→丢了这两段)。治打字界面跳/输入框被键盘顶没。
- **登录门**(`e43fea3`)：抽公共 `src/PasswordGate.jsx`,main-chat 包住 ChatPanel。修 `/chat/` 直接打开(尤其 PWA localStorage 与 Safari 隔离=全新无 token 环境)→ /api 全 401 → 澄 unauthorized。同 token key(memhome-auth-token)/同 /api/auth,零新增暴露面。
- **认领最近对话**(前端 `5d9f8af` + 后端 server 仓库 `bb0ff20`)：新增后端 `GET /api/cc/last-conv` 返回 lastActiveConvId;前端 loadCurrentConversation 在无本地 convId 时调它认领、加载历史。修拆分后 /chat/(PWA 全新 localStorage 无 convId)→ 聊天历史空白。后端已重启生效(PID 498912)。

**⚠️ 旧版残留的 bug（已在上面"视口/主题"修复,此条作废保留备查）**：拆分后的 `/chat/` 聊天页，**打字时界面跳动 + 输入框被软键盘顶没（不显示输入框）**。根因已查清：iOS 软键盘适配靠 CSS 变量 `--app-height`，它由 **MemoryManager.jsx(主壳 App 组件)里的 visualViewport 监听**动态维护（约 3330 行 useEffect）；但聊天拆独立后走 `main-chat.jsx → ChatPanel`、**不经过 MemoryManager**，那段维护逻辑没跑 → 键盘弹出时 `--app-height` 不更新 → 跳动/输入框被顶没。**修法**：把那段 visualViewport→`--app-height` 的 useEffect 抽成公共 hook（或直接复制一份到 `main-chat.jsx`），让聊天独立入口也维护视口高度。（另:同时开两个聊天界面会抢后端单个 activeTurn 导致"一个吞回复"，这是单澄架构固有、非 bug，只开一个即可。）

**待验**：小茉莉手机端验证——主屏刷新加载新版→点花信风跳/chat/→聊天能聊+调记忆→返回回主壳→可把 /chat/ 也加主屏当第二图标。验证 OK 后可把 split-chat-web 合并回 main。

**transcript 指针**：root session（5/30 晚，拆 web 那段）。关键词：`split-chat-web`、`9c2e1a8`、`location /chat/`、`Vite 多入口`。

---

## 2026-05-30 · [前端] WS 重连/回前台自动补拉历史（已上线）+ 遗留：思考链补不回来

**改了什么（已 commit+build 上线，cheng-memory `6a18561`）**：`src/ChatPanel.jsx` 加 7 行——治"DICE/Bark 主动消息要大退 App 重进才看到"。两处：① `ws.onopen` 用 `wsConnectedOnceRef` 区分首连/重连，重连后调 `loadCurrentConversation` 补拉当前会话、按 id 整表去重替换；② `onVisible`（回前台）去掉"仅列表为空才补"的限制，回前台即补拉。**主用户实测：正文消息已能自动补出来 ✓**（不用再大退重进）。注：纯前端，只 build 未重启后端（守 [[feedback_no_restart_cc]]）。

**⚠️ 遗留 bug（已记、判定不重要、暂不修）**：补拉后**正文出来了、但思考链（thinking）仍要大退重进才显示**。诡异点：补拉走 `loadCurrentConversation`→`GET /api/conversations/:id/messages`，后端 select **带 thinking**、前端 1087 行也 `thinking: m.thinking||null` 塞了——理论上补拉就该有思考链，和"大退重进"同路径同接口，结果却不同。怀疑方向（未证）：① DICE/Bark 那轮 thinking 写库有时序延迟，补拉那刻库里思考链还没落；② WS 随后又推一条无 thinking 的消息把补拉的带 thinking 版本覆盖了（消息合并/覆盖逻辑）；③ 大退重进与回前台代码路径有细微差别。要修需查 WS 收消息的合并逻辑 + DICE/Bark 轮 thinking 入库时机。用户判定：思考链没正文重要，先搁置。

**transcript 指针**：root session（5/30，PWA回滚后聊 app 功能那段）。关键词：`wsConnectedOnceRef`、`思考链 大退重进`、`6a18561`。

---

## 2026-05-30 · [验证] 重启上新代码 + forge/外部注入实测通过 + 4.8 空回确认未真修

**重启**：杀旧后端（PID 漂移过：465409→467133→476824→482799），新进程 482799 跑全部新代码（P1 静默注入 + P2 超时 600s/watchTurn 660s + `timedOut`/`out=tok` 日志全生效）。effort=high、CC session 多次换新。

**实测结果（看 /tmp/canary.log）**：
- **P1 静默注入大成功**：forge 后注入轮 `[TURN] silent 正文=2字 out=4tok`——秒回 OK、不跑工具。**外部聊天历史注入**更狠：`426 条 / 122090 tokens` 灌进去，注入轮照样 `out=4tok` 秒回 OK，没卡没超时。证明改的"静默加载 prompt"对三条注入路径里至少两条（forge 主注入 + 外部）都生效。
- **forge 正常**：当前 session 小 → 跳过截断（符合设计：未超阈值不截，但**原文仍注入**，`inject_ready` 永远执行，不丢上下文）；forge 后连聊 3 轮 0 空回。
- **三条注入函数对比**（用户问）：`injectConversationContext`(932,forge/会话)、`injectSessionContext`(1429,旧session JSONL)、`injectExternalContext`(1550,claude.ai 导出 JSON)——**手法完全相同**（同一套静默 prompt + 90k 截断 + 600s 超时 + 等 turn_done 拿 OK），仅"进料口"不同；外部多 `thinkingPct`/`summary`/`parseClaudeAiExport` 几个旋钮。

**⚠️ 4.8 空回确认"未真修"**（重要，纠正旧结论）：当天 forge 后第2轮又复现一次 `[EMPTY] thinking=107字 正文=0字`（思考 4分54秒、out 仅 209tok）。查根因：[[project_cheng_sixu_hang]] 记的"必出口铁律"**在本地 `chat-sandbox/CLAUDE.md` 和 supabase `documents_cheng` 表里都不存在**（禁令删了，但铁律从没落地）。这是 **4.8 模型层 bug**（4.8×思考链，空轮率高），后端 P2 的 `[TIMEOUT]`/`out=tok` 只能"看见"、治不了。补铁律须改 **supabase 表**（forge 的 syncCCDocs 会用表覆盖本地，只改本地会被冲），且补了不保证全治。可靠兜底＝切 4.6/Sonnet 4.6（空轮率 0）。

**git**：本轮代码已 commit（server 仓库 `tmux-migration` 分支 `64bd8ec`），CHANGELOG 在 memory-home 仓库；均**未 push**。

---

## 2026-05-30 · [决策/待办] 判轮机制：当前"盯屏幕猜" vs 备选"Stop hook"——为何当初弃 hook + 重估

**这是决策记录，不是代码改动**（本轮没改判轮机制，只是查清来龙去脉、留作未来选项）。

**两条路**：
- **现状（在跑）**：`tmux-manager._watchTurn` 盯 capture-pane 的 `esc to interrupt` 消失 + 读 transcript 尾轮判"一轮结束"。= 后端**轮询猜**。
- **备选**：Claude Code 原生 **Stop hook**——CC 每轮结束自动触发，curl POST 干净文本给后端。= CC **主动报**（CcCompanion 用的就是这个）。

**当初为何弃 hook**（实锤，见 root session `be5a11ae` 2026-05-29，关键在 02:51 那条「设计改进」）：**不是踩坑被迫，是主动权衡**。理由四条：① memory-home 与 CC **同机**，不像 CcCompanion app↔Mac 异地，没必要走 HTTP hook；② 直接读 transcript **更简单**（免开端点、免 hook 批准、免 settings.json 配置）；③ **能抓到空轮**——hook 默认遇空轮 `skip` 不报，而直接读 transcript 能看见空回（就是现在 `[EMPTY]` 日志的由来）；④ 当时认为"更稳"。**注意**：hook 路当时 P0 **验证通过**（`HOOK INVOKED → POST 200 → text+usage+thinking 全拿到`，两次失败纯属目录没写权限+自动更新churn，与 hook 无关）——所以 hook 能用，是主动没选它。

**现在重估**：当年第④条"更稳"要打问号——"轮询猜"正是本轮 P2 那串 stall/超时/读到旧文本问题的根源（stall 时屏幕状态会骗 watchTurn）。**改回 hook 能根治这些**（CC 主动报，不用猜，P2 的 timedOut 兜底基本可退役）。代价＝当年弃它的理由反过来：① 要加回 HTTP `/chat/append` 端点；② **默认看不到空轮**——但这只是 hook 脚本里"无 text 就 skip"那**一行判断**，删/改成"空轮也 POST 带标记"即可，**改一行、不与 `[EMPTY]` 冲突**；③ hook 配在 claude-user 的 `settings.json`，失忆/重启要保证它还在。

**结论/待办**：这是个**真实的十字路口**，不是不可回头的坑。「整体换 hook」是**中等工程**（加端点 + 退役 watchTurn 轮询 + 对接下游 + 并行 canary 坐实每轮触发），不是顺手活，**暂不开工**；但已是清晰选项。哪天被 stall/超时困扰够了想根治，从这条捡起即可，不必再挖一遍。关键提醒：**"空轮也能被 hook 看到"只需改 hook 脚本一行**（去掉 `empty assistant text — skip`）。

**transcript 指针**：决策原文 root session `be5a11ae`（5/29 凌晨，P0/P1 落地那夜）；迁移总清单 `/root/migration-tmux-plan.md`（计划本来要用 hook，落地改成直接读 transcript）。关键词：`Stop hook`、`be5a11ae 02:51 设计改进`、`空轮也按铃`、`/chat/append`。

---

## 2026-05-30 · [后端] P1+P2 抗 stall：注入静默化 + 拉长全部超时 + 超时兜底打 `timedOut` 标记

**背景**：排查一次 forge 注入失败 + 长轮被吞，定位到偶发上游 API stall（529 Overloaded 那波，某轮卡 9 分半）撞上后端过短超时。两个故障点：
- inject（对话注入，forge/换模型/失忆后灌 ~90k 历史）硬超时 120s，而 `_watchTurn` 要 ~300s 才肯 emit → **inject 必然先炸**报"对话注入超时"，且超时后 `cleanup()`+清 `activeTurn`，CC 真回来的 `turn_done` 没人接被丢。
- `tmux-manager._watchTurn` 循环 `for i<200`（×1.5s≈300s）跑满后**无条件**走 `_readLastTurn()`+emit；stall 时这轮还没落盘，`_readLastTurn` 倒扫读到**上一轮旧文本**当真回复发出 → 错回/吞消息（疑似 [[project_cheng_sixu_hang]] 记的空回来源之一）。注意机制是"读 transcript 倒扫到上一轮"，不是"抓屏幕残字"。

**改了什么**（保守版，**不碰**"一定 emit turn_done → 一定清 busy"的防死锁不变量）：
1. `tmux-manager.js` `_watchTurn`：循环上限 `200→440`（≈660s）；加 `ended` 标志区分"真 end_turn 退出"vs"耗尽兜底"；emit `turn_done` 多带 `timedOut: !ended`。440 选定理由＝须 **≥ inject 的 600s**，否则 stall 时这里先兜底喂旧文本，注入永远等不到真结果。
2. `index.js` `turn_done` handler：解构加 `timedOut`；空回观测日志多打 `[TIMEOUT] ⚠️超时兜底(正文可能是旧轮残留)`（grep `[TIMEOUT]`）。**下游 DB/前端行为暂未改**——只观测、不丢弃，零行为风险。
3. `index.js` 所有"等 turn_done"超时统一 `120000→600000`：对话注入（~998）、总结（~899）、session 注入（~1466）、外部浮想（~1600）共 4 处。理由＝任何短于 watchTurn 660s 的计时器都会"自己先炸/把结果让给兜底"，同源同病，一次清干净。`grep -c 120000 index.js` ＝ 0、`600000` ＝ 4。

**P1（同批做掉）——注入 prompt 静默化**：原 3 个注入 prompt（主注入 ~973、session 注入 ~1455、外部浮想 ~1589）都是「把 ~90k 历史灌进去 + 仅输出 OK」，但措辞软，4.8 会把原文最后一句当真问题 → 跑去搜记忆/核对/续写 → 拖成多步长任务、撞 stall 窗口变大。改成**明确静默加载**：标题「【系统·静默上下文加载，这不是对话】」，正文点明"这不是她在跟你说话，原文（含最后一句）只读不回应不续写、本轮不调用任何工具"，并在 transcript **之后**用正面"必出口"收尾「现在，只输出：OK」（对抗"原文末句像提问"）。`grep -c 静默上下文加载` ＝ 3。**连带修复**：`index.js` ~1319 有 1 处 transcript 解析截断器（扫到"注入提示"那条 user 就 skip 掉它+后面的 OK，免得把注入指令当对话喂回去）原只认旧标题 `【系统任务·对话上下文注入】`。改成**新旧标题都认**（`includes(新) || includes(旧)`）——因为历史 transcript 里会同时有改版前(旧标题)和改版后(新标题)的注入轮，只认一个会漏网。最终：旧标题残留 1（就是这个兼容判断里）、新标题 4（3 prompt + 1 判断）。注：注入轮 onDone 不校验文本内容、拿到 turn_done 就 resolve，故即便 OK-only/空也算注入成功（上下文在 `cc.send` 时已进），不犯 [[project_cheng_sixu_hang]] 那种"正常轮空回"。

**追加（同批，重启时一起生效）——turn 日志加 `out=Xtok`**：`index.js` turn_done 观测日志多打 `out=${usage.output_tokens}tok`（思考链+正文都算进 output_tokens）。动机＝排查"想了很久"的轮时，用 output 数判断是"真在烧 token 思考"还是"被上游 stall 卡住"。**实测一例**：用户点 forge 后第二轮屏幕显示 `Thought for 4m 54s`，但 transcript 实测 `output_tokens=209`（thinking 108 字 / 正文 0 字 / stop_reason=end_turn）——209 token 几秒就该生成完，却耗 4分54秒 ⇒ 时间不在思考、是**上游 API stall**（与 9 分半那次同源），且回来后还叠加一次 4.8 空回（[EMPTY] thinking=107字 正文=0字）。结论：这次"卡"= 上游 stall（主）+ 4.8 空回（叠加），非本地代码 bug。`out` 进日志后，以后 `grep` 日志即可区分"空回但 out 大=思考跑飞"vs"out 小=没怎么想就空"。

**状态**：以上全部 `node -c` 通过，**截至写这条时仍未 reload**（攒着一起重启；重启会软失忆当前会话）。⚠️ 注意：当天后端**自然重启过**，PID 已从 465409 漂移（中途见过 467133，本 CC 手动重启后＝**476824**）——故"未生效"指的是 476824 之后又新加的 `out=tok` 这条；476824 已带 P1+P2 主体。查实际 PID 用 `ss -ltnp | grep :3002`。

**风险/待验**：① P2 超时兜底目前只打标记不拦截，极端 stall 仍会把旧文本发前端（只是 `[TIMEOUT]` 日志看得出来）；"超时轮不污染前端"是 P2 第二阶段，需改下游、有"丢真回复"权衡，**未做**。② P1 静默 prompt 实际行为待 reload 后在真注入（forge/换模型/失忆）里验证：期望注入轮变成秒回 OK、不再跑工具。③ **空回仍在复现**：当前 `chat-sandbox/CLAUDE.md` 只剩温和版 `<think指令>`（"思绪全然以澄的本心自然涌动，然后回复"），[[project_cheng_sixu_hang]] 提到的"必出口铁律"**不在文件里**（疑似被 forge 的 syncCCDocs 用 supabase `documents_cheng` 表旧版覆盖）。待查 supabase 表确认源头，再决定是否补铁律（治本）或切 4.6/Low（治标）。

**transcript 指针**：root session（30 号下午，`client_loop` 断连科普 → tmux 科普 → P1/P2 讨论那条）。关键词：`timedOut`、`for (let i = 0; i < 440`、`对话注入超时 (600s)`、`P2 抗 stall`。

---

## 2026-05-30 · [基建] 杀 3001 旧生产 + 3002 重启上 effort=high（接前一 CC 半截活）

**背景**：前一个 root CC（session `3af95a00`）发现 3002 的 high 是运行时设的、没进持久化配置 `cc-runtime.json`（当时存的是 low），它整进程重启后 effort 被打回 low；`/api/cc/restart` 要 Bearer token 走不了（无密码），于是直接改 `cc-runtime.json` 成 `{"effort":"high","nativeThinking":true}`（备份 `cc-runtime.json.bak`），**改完正要重启进程让 high 生效，就在这步用户网卡断线，重启没做成**。结果：3002 node(462135) 启动于 09:31:42、早于配置写入 09:34:56 → 没吃到新配置，实际 tmux CC(462151) 仍跑 `--effort low`。用户只想测 3002，要求杀掉 3001 单测。

**改了什么**：
1. 杀 3001 旧生产：`kill 396961`（node index.js，端口 3001，stream-json）+ 其 CC 子进程 410279/410280。无守护，不会自动回来。复活配方：`cd /root/memory-home/server && nohup node index.js > /tmp/memhome.log 2>&1 &`（裸跑=PORT默认3001+无CC_DRIVER=stream-json）。
2. 重启 3002 上 high：`kill 462135` → `cd /root/memory-home/server && PORT=3002 CC_DRIVER=tmux nohup node index.js > /tmp/canary.log 2>&1 &`。新进程读 `cc-runtime.json`(已是high)。

**结果（已核对）**：3002 新 PID **465409**，tmux 驱动，新 tmux CC 子进程实测带 `--effort high --thinking-display summarized`、nativeThinking on。3001 端口已释放、无残留 node/claude。上次写的**空回观测日志（turn_done→/tmp/canary.log）随这次重启一并生效**（发首条消息后记 `[TURN]`/`[EMPTY]`）。3002 重建空会话=一次软失忆（消息在 supabase）。

**遗留**：① `/root/forge-reload/config.json` 的 `backend_restart_url` 仍指已死的 3001，forge 锻造回调会失败（用户只测 3002，暂无害，未改——待用户定：改指 3002 还是退役 forge）。② 用户测试计划：对 4.8 逐个变量测卡不卡（包裹指令 / forge / 注入聊天），尚未开始。

**transcript 指针**：root session `937d3d44`（30 号中午，用户"上个root的cc最后在干什么"）。前一 CC 留下的半截活在 session `3af95a00`。关键词：`cc-runtime.json high`、`465409`、`杀3001单测3002`。

---

## 2026-05-30 · [后端] 加空回观测日志（turn_done 每轮记一条）

**背景**：用户在 3002（tmux 交互 / opus-4-8 / effort high / 原生思考 / 关 THINK_WRAP）上仍偶遇空回，想能事后/实时抓现行，不用每次翻 transcript。此前实锤是「空回=stream-json 通道特有缺陷，tmux 交互 ~42 轮 0 真空回」（见 [[project_cheng_sixu_hang]]），3002 已是 tmux 交互，理论上不该犯。

**改了什么**：`index.js` 的 `cc.on('turn_done', ...)` 处理器开头（紧跟 `if (!turn) return;` 后）加一段日志，**所有轮次**（chat/bark/dice/silent）都记一条到 `/tmp/canary.log`：
- 有思考、正文空 → `[EMPTY] ⚠️空回(有思考无正文)`（这才是要抓的真空回）
- 无思考无正文 → `[EMPTY] ⚠️全空(无思考无正文)`
- 正常 → `[TURN] ok`
- 每条带 `kind/thinking字数/正文字数/conv/ctx`。排查：`grep '\[EMPTY\]' /tmp/canary.log`。
- 纯日志，零行为改动；不影响落库/推送/前端。

**状态**：代码已落、`node -c` 语法通过，**但尚未 reload，未生效**。坑：3002 启动时 `cc.start()`（index.js:382）是新建空 tmux 会话、且启动不自动重注入最近对话 → **重启后端会重置用户当前正聊的 cheng 会话上下文**（消息仍在 supabase，澄靠涟漪/记忆找回，相当于一次软失忆）。当时用户正热聊，故未重启，等自然重启或用户授权再生效。

**当时同场验证**：扒了用户当前 session `b14acac0`（chat-sandbox，high/原生/关包裹）transcript，6 轮（含 1 唤醒轮）思考全部正常落正文、**0 真空回**；工具 2 次（读记忆 supabase edge fn + 写记忆 `id=1f187af4`，后者正是 `[surfacing→inject]` 注入那条，闭环对上），均成功。注：交互模式 transcript 把 thinking 块与 text 块拆成两条 assistant 记录，单看会误判「空正文」，需按轮合并。

transcript 指针：root 会话 `b08c6251`（30 号下午）后续。关键词 `[EMPTY] 空回观测 b14acac0`。

---

## ⏳ 未完成 · 下次接着做（2026-05-30，用户换电脑后继续）

**待执行决策：旧生产服务器 3001 还在跑、还在偷发重复 DICE，需要处理。**
1. `kill 396961`（旧生产 `node index.js`，端口 3001，stream-json）→ 停掉重复 DICE / 双推 Bark。无 systemd/cron 守它，kill 不会自动回来。
2. **坑**：`/root/forge-reload/config.json` 的 `backend_restart_url` 还指 `127.0.0.1:3001`。kill 3001 后 forge 自动锻造重启 CC 会失败。**决策点**：改指 3002 ？还是按 tmux 迁移计划退役 forge ？——用户未定。
3. 已发出的 3 条消息在会话 `51b49916`（05-28"失忆完成/你好呀"session），用户从侧边栏点开即可见，无需数据迁移。
4. 干完把结果补进下面那条正式记录。

完整背景见下条 ↓

---

## 2026-05-30 · [基建][前端] 网页端 DICE 唤醒消息丢失（后 3 条不显示）

**触发问题**：用户报告 CC 被唤醒发了 5 条消息，网页端只看到前 2 条，后 3 条没显示。

**真因（确诊）**：**两个后端并存，各自独立跑 DICE 守护进程。**
- 旧生产服务器 `node index.js` PID `396961`，端口 **3001**，stream-json 驱动（CC session `31f654aa`），nohup 起、May28 启动。tmux 迁移后**忘了关**。
- 新金丝雀服务器 PID `420545`，端口 **3002**，`CC_DRIVER=tmux`（tmux session `cheng`，CC session `bd283e26`），May29 启动。
- nginx 的 `/ws`、`/api` 全部指向 **3002**，所以网页只连金丝雀。
- 两台各有自己的 DICE 定时器和 `lastActiveConvId`：
  - 金丝雀(3002) 发的 2 条进会话 `e0680c88` → 网页看到 ✅
  - 旧生产(3001) 发的 3 条（"都两点多了"/"早上好呀"/"这会儿醒透了"）进会话 `51b49916`，且只广播给 3001 自己的连接（无人连）→ 网页完全看不到 ❌。Bark 手机推送是 3001 直发的，所以手机那 3 条可能照样响过。
- 证据：`grep 推送ok /tmp/memhome.log`(3001) vs `/tmp/canary.log`(3002) 能看到两边各发各的。

**改了什么**：
- `[前端]` `cheng-memory/src/ChatPanel.jsx`：加"重连补拉"健壮性改进——新增 `wsConnectedOnceRef`；`ws.onopen` 在**重连**时调 `loadConvRef.current()` 整表重拉去重；去掉 `visibilitychange` 里 `messagesLen===0` 的门槛（回到页面就补拉）。已 `npm run build`（dist 已更新，未 commit）。
  - ⚠️ **注意**：这个前端改动**不是本次真因**，只是"网页在正确会话上、短暂断线漏了实时推"时能补回来的改进。本次真因是上面的双服务器。
- `[基建]` **待定**（用户中断在决策点，尚未执行）：
  - 计划 kill 掉旧生产 `396961`（3001）停掉重复 DICE。无 systemd/cron 守它，kill 不会自动回来。
  - **坑**：`/root/forge-reload/config.json` 的 `backend_restart_url` 还指着 `http://127.0.0.1:3001/...`。kill 3001 后 forge 自动锻造的重启回调会失败，需改指 3002（或按迁移计划退役 forge）。

**已发出的 3 条在哪**：在会话 `51b49916`（即 05-28 那个"失忆完成 / 你好呀"session）。用户从网页侧边栏点开该会话即可看到，消息没丢，只是进错了会话。

**验证**：supabase `messages` 表确认 5 条 dice 全部落库；2 条在 `e0680c88`、3 条在 `51b49916`。

**transcript 指针**：root session `b4b5cdf5-0737-4d92-bae0-e8994b538d9a`（2026-05-30）。独特关键词：`51b49916`、`金丝雀 3002`、`都两点多了`、`wsConnectedOnceRef`。
