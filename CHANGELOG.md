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
