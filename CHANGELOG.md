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
