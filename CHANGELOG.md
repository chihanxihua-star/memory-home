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
> - **跨仓改动要双向点名**：如果这条改动同时记在另一个仓的 CHANGELOG（如 world-home 改了功能、后端逻辑落在这），本条开头加一行 `🔗 对应：<对面仓>「<对面那条标题>」(<路径>, <日期>)`，对面那条也回指本条。这样从任一本都能跳到对面。
>
> **transcript 在哪**：root 自己的 Claude Code 记录在 `/root/.claude/projects/-root/*.jsonl`，按 sessionId 命名。`grep -l "<关键词>" /root/.claude/projects/-root/*.jsonl` 能定位是哪个 session。

---

## 2026-06-12 · [后端] 下班链：16点不再瞬移——加班提示/下班选择包/evening 通勤链/下雨版

**需求（用户当日逐条定）**：下班弹唤醒包让澄选怎么回家；加班=直接提示不可选、30-90分钟、保留25%几率；选什么工具状态栏走什么标签；地铁要有"从地铁站走回家"一段；钱/时长复用早晨链已定的 COMMUTE_OPTS（地铁13-17分¥0/打车8-12分¥30/走路22-28分¥0，站↔家步行3-9分）。

**做了什么**：
- `world-workday.js`：加班时长 30-120→**30-90**；`offWorkDecision` 重写——骰中25%加班→startOvertime+**提示型唤醒**（overtime_notice，单确认项"知道了，继续干"，CC忙就不提示加班照走）；没骰中→排 `offwork_choice` pending（daemon 自带 cc_busy 重试）。`endOvertime` 不再瞬移：发¥30加班费+activity"收拾东西准备回家"→走 evening 链（雨天自动打车不再问）。钩子 `setOffWorkHandler/setEveningStarter` 由 index.js 注入（避免循环依赖），没注入/出错退回旧瞬移保底。
- `index.js`：新链 `buildEveningRoutine(cm)`——subway=坐地铁(13-17)→从地铁站走回家(3-9)→家·客厅"下班回家后休息"；taxi/walk 直达。firePendingWake 新分支 `offwork_choice`：人不在公司=作废；现读 `isRainingNow()`（world_environment_cheng.weather_text 含"雨"，weather-fetcher 45分钟一更）→ 晴=①坐地铁②走路 / 雨=①打车¥30②地铁淋一段(掉清洁small+体力tiny)③在公司等雨小一点(排25分钟 offwork_choice pending，payload.waited=true，第二次不再给"等"防循环)。选项走现成 start_routine/routine_opts 钩子进链。
- `world-random-events.js`：**rain_offwork（下班下雨）整个删除收编**——由下班链雨天版接管，不再跟系统下班判断穿帮；DevPanel force 列表动态拉会自动少这个。
- 变体表补 3 个新事件默认文案：offwork_choice / offwork_choice_rain / overtime_notice。

**已知边界**：选择解析失败时兜底选项不执行 start_routine（通用 guard），她会留在公司不动——跟早晨链解析失败留床上同款，罕见，暂不专修。早晨通勤的"下雨/赶时间偏移打车"**仍没做**（这次只做了下班侧+加班结束自动偏移）。

**状态**：三文件 node --check 过；**未重启**，进攒批。测试路径：DevPanel 的 off_decision force 按钮（骰子）/ 手动插 offwork_choice pending。

**transcript 关键词**：「buildEveningRoutine」「offwork_choice」「overtime_notice」「从地铁站走回家」。

---

## 2026-06-12 · [后端] 修"饱了别唤醒"：hungry 续唤醒到点先复验触发条件

**问题**（6/06 实测撞过）：澄选"先忍10分钟"/"去厨房看看"排的 hungry 续唤醒，到点**不查当时真实 satiety 强制发**，且 pending 自带绕过 30min 冷却。她若期间已吃饱→被唤醒→只能再选"先忍"→又排一条→每10分钟循环；解析失败兜底还会自动选最后一项（恰是"先忍"）。

**修法**（index.js firePendingWake 兜底分支，~1235 前插一段）：到点先调事件自身 `def.trigger(status)`（hungry=satiety<30）复验；不满足→这条 pending 标 `status='cancelled'` 静默作废、写日志、不打扰澄。之后真饿了由普通 tick 自动检测重新触发。pending_wake_cheng 无 status check 约束，cancelled 安全；念头池收尾把非 queued 当已解决，兼容。用 trigger 复验而非硬写 satiety<30，以后同类续唤醒事件自动适用。

**状态**：node --check 过；**未重启**，进当前攒批。

**transcript 关键词**：「饱了别唤醒」「触发条件复验」「静默作废」。

---

## 2026-06-12 · [后端] 世界唤醒"原因"文案改成可配变体（world_wake_reasons_cheng 随机抽）

🔗 对应：world-home「WakeReasonsPanel 唤醒原因编辑器」(/root/world-home/CHANGELOG.md, 2026-06-12)

**需求**：唤醒包的"原因"句之前全是代码里写死的固定文案，用户想像 <此刻> 自述那样可编辑、有变化。只改"原因"这句；选项/effects 是大头以后大改（点外卖等选项还没定完）。

**做了什么**：
- 新表 `world_wake_reasons_cheng`（migration `create_world_wake_reasons_cheng`）：event_key + label + text + enabled，RLS 全开。已把全部 21 个事件的现有固定文案各灌一条当默认变体。
- `index.js`：新增 `pickWakeReason(eventKey, fallback)`——按 event_key 抽一条启用的变体（多条随机），表里没有/读失败退回代码默认。接入点在 `triggerWorldWake`（**await 放在 check-and-set activeTurn 之前，原子性没破**），所有 engage 澄的唤醒（hungry/随机/工作/日常流程）统一生效；pending 续唤醒的"补充"行走 pendingContext 参数，不受影响。
- CRUD：GET/POST `/api/world/wake-reasons`、PATCH/DELETE `/api/world/wake-reasons/:id`（照 narration 编辑器的套路，多了 delete）。
- 前端编辑面板在 world-home（见对面那条）。

**状态**：`node --check` 过；**未重启 cheng-backend**——变体抽取和 CRUD 接口都要重启才生效（world-home 的编辑面板在重启前会报错，正常）。等用户定重启时机。

**transcript 关键词**：「pickWakeReason」「world_wake_reasons_cheng」「唤醒原因变体」。

---

## 2026-06-12 · [前端] 「我的状态」位置/正在改点选 chips（治键盘跳）+ 候选标签入库跟小世界同步

🔗 对应：world-home「UserStatusPanel 候选标签改读 user_status_options_cheng」(/root/world-home/CHANGELOG.md, 2026-06-12)

**为什么**：弹窗里点输入框必跳——iOS 弹键盘时系统自己平移视口 + 页面 --app-height 压缩 + ChatPanel scrollTo(0,0) 拉回，三方拉扯网页拦不住。顶部卡片、visualViewport 抬升都试过仍跳。最终思路=**让弹窗用不着键盘**：位置/正在 从输入框改成点选 chips，点选不弹键盘就彻底不跳。

**做了什么**：
- 新表 `user_status_options_cheng`（Supabase migration `create_user_status_options_cheng`）：kind('location'|'activity') + label + sort，unique(kind,label)，RLS 全开 read/insert/delete（照 phone_todos_cheng 的套路），已灌入原硬编码的 9 个位置 + 10 个活动。
- `cheng-memory/src/ChatPanel.jsx` UserStatusSheet：US_LOCATIONS/US_ACTIVITIES 硬编码删掉改读表；位置/正在=chips 点选（选中深色填充）；「＋」内联输入存自定义标签（upsert ignoreDuplicates，存完自动选中）；「编辑」模式 chips 带 × 点击删标签；备注仍是输入框（可选、用得少，跳一下忍了）。已 build。
- 同步：world-home UserStatusPanel 的 datalist 候选改成读同一张表（样式没动），聊天里存的新标签小世界立刻可见，反之亦然（小世界没做增删 UI，增删都在聊天端）。

- 追加：标签收藏（migration `user_status_options_add_pinned`：加 pinned 列 + update 策略）。收藏的标签排最前（两端读取都按 pinned desc）；聊天端「编辑」模式重做=点标签切换 ★ 收藏/取消，点尾部 × 才删除（比原来"点了就删"安全）。

**transcript 关键词**：「user_status_options_cheng」「tagGroup」「让弹窗用不着键盘」「togglePin」。

---

## 2026-06-12 · [前端] 「我的状态」弹窗 UI 换涟漪风（只改皮，逻辑不动）

**需求**：聊天页输入栏「我的状态」弹窗（UserStatusSheet）样式参考涟漪板块「新涟漪」表单的 UI，仍保持底部弹窗形式，只改 UI 其他不变。

**做了什么**（`cheng-memory/src/ChatPanel.jsx` 的 `UserStatusSheet` 渲染部分，已 build）：
- 顶部加涟漪式三段栏：左 CANCEL 文字钮 / 居中「我的状态」标题（letterSpacing 0.22em）/ 右深色 SAVE ✓ 方块钮（替代原底部全宽保存按钮）。
- 「在家/不在家」从 文字+切换按钮 改成两个 chips（选中=深色填充，同新涟漪的作者 chips），点未选中那个即触发原 togglePresence（仍立即落库）。
- 位置/正在/备注 输入框从 圆角边框 改成 label 在上 + 下划线式（borderBottom，Georgia/Noto Serif 字体），照抄 MemoryManager 的 labelStyle/underlineStyle。
- datalist 候选、supabase 读写、保存 toast、onClose 行为全部没动；没动后端。
- 追加修复：iOS 键盘弹起会把这个 fixed bottom:0 的弹窗遮住；先试过 visualViewport 抬 bottom（去抖+transition）仍然跳——iOS 自己滚页面对焦 + 我们抬弹窗两个动作叠加治不了。最终方案=弹窗从底部 sheet 改成**顶部悬浮卡片**（fixed top+16px、圆角 14、cp-msgIn 淡入），键盘碰不到上半屏，彻底不动。visualViewport 逻辑已删。

**transcript 关键词**：「涟漪式三段顶栏」「UserStatusSheet」「SAVE ✓」「顶部悬浮卡片」。

---

## 2026-06-12 · [前端] 聊天页输入栏加「我的状态」按钮（同步小世界 user_status）

**需求**：用户想在聊天 web 里直接改自己的状态（位置/正在/备注/在不在家），不用跑去 world-home——保存后澄从 <此刻> 里立刻看到。

**做了什么**（`cheng-memory/src/ChatPanel.jsx`，已 build 上线）：
- 输入栏三个点左边加人形图标按钮 → 底部弹窗 `UserStatusSheet`（复用 cp-plus-sheet 样式）。
- 弹窗=照搬 world-home `UserStatusPanel`：在家/不在家切换（带默认位置）、位置/正在（datalist 候选）、备注、保存。
- 数据通路：前端直接 supabase 读写 `user_status_cheng`（同 world-home 的路子，不经后端）→ <此刻> 每条消息现读这张表 → **保存即生效，不用重启任何东西**。
- 没动后端、没动 world-home；world-home 那边的面板照常可用，两边写同一张表。

**transcript 关键词**：「UserStatusSheet」「我的状态」「us-loc-list」。

---

## 2026-06-12 · [后端] 注入精简：浮现标签瘦身 + <此刻> 去前缀 + [MOVE:] 说明搬进世界唤醒区③

**做了什么**（用户逐条定的格式）：
- `inject.js`：`<小世界浮现 — 仅你可见的背景…>` / `<记忆浮现 — …>` 两个长标签缩成光秃秃的 `<小世界浮现>` / `<记忆浮现>`，那句"仅你可见、自然融入、别复述别回应"的叮嘱挪进 sysprompt（每条消息省几十 token，规矩在 sysprompt 每轮都在）。
- `world-narration.js` `buildNowInner`：`澄：现在是…` 去掉「澄：」前缀（自述本来就是第一人称）；`小茉莉：在…` 去冒号变 `小茉莉在…`。聊天 <此刻> 和世界唤醒包共用，两边同时变。
- sysprompt（**源头=documents_cheng id=96c4f3a9**，文件已同步）：[MOVE:] 说明从 [BARK:] 段后撤掉，进 <世界唤醒区> 新增「③ 聊天专用（世界唤醒里不用）」=MOVE 用法 + 浮现块叮嘱。文案以用户改的版本为准（修了"工位"掉字、首句顺成"换了地方、或开始做另一件事"）。
- ⚠️ 教训再记一遍：sysprompt 直接改文件会被重启时 syncCCDocs 用表内容覆盖，**必须改表**（=前端"系统提示"框）。

**状态**：代码 node --check 全过；**未重启**。本日三批改动（[MOVE:] 标签、去 [时间标记]、本条）攒着等用户定重启时机一次生效。

**transcript 关键词**：「聊天专用（世界唤醒里不用）」「buildNowInner」「标签精简」。

---

## 2026-06-12 · [后端] 去掉用户消息头上的 [时间标记] 前缀（与 <此刻> 重复）

**为什么**：用户消息发给澄前，隔 ≥15 分钟会在头上缀 `[时间标记：现在…，距上次消息 X 分钟]`（`maybeTimePrefix`）。但 12A 之后**每条**消息都带 `<此刻>` 块（现算 +8 时间），时间信息重复；"距上次多久"用户也不想给了（计时基准还是错的——按用户上一条算而非澄最后回复）。用户拍板：直接去掉。

**改了什么**：`server/index.js` 发送路径（原 `const prefixed = maybeTimePrefix(combinedText, conversation_id)`，~L4055）改为直通 `combinedText`，原行注释保留恢复方法。`maybeTimePrefix` 函数本体（~L594）和 `convLastMsgTime` Map 留着没删，想恢复换回一行调用即可。**未重启，跟 [MOVE:] 一批等用户定时机。**

**transcript 关键词**：「时间标记」「maybeTimePrefix」「距上次消息」。

---

## 2026-06-12 · [后端] 聊天 [MOVE:] 标签：澄聊天里说去哪/做什么，小世界位置真的跟着动

**需求**：聊天时澄说"我去厨房烧个水"，世界状态里她还坐在客厅——位置/行为不跟聊天走。用户要"地点跟着动"。

**做了什么**（`server/index.js` 三处 + sysprompt 一处）：
- 新函数 `processChatMoveTag(text)`（在 processTodoDoneTags 下方）：解析回复里的 `[MOVE:地点]` / `[MOVE:地点·行为]`，**只认一条回复里最后一个**。地点白名单=7个房间（家·卧室/客厅/厨房/浴室 + 公司·工位/休息室/茶水间），可省"家/公司"前缀（按房间名匹配，两栋楼不重名）；**只许同栋楼移动**（跨楼忽略，下班回家归世界作息管）；当前 location 不带"家/公司"前缀（如通勤路上）也会被同楼校验自然拦掉。
- 生效=更新 `character_status_cheng` 的 location/activity（没写行为时默认"在{房间}"）+ 写 `daily_timeline_cheng`（source=action, detail.via='chat_move'）+ 复用 `scheduleActivityEnd` 给新行为排自动收尾（自由文本走默认中桶 10-29 世界分钟）。**铁律：只改这两个描述字段，不结算任何数值**（饱腹/钱包等带后果的仍走世界事件/effects 系统，防聊天变作弊通道）。
- 接线：聊天轮 turn_done 的 `!turn.silent` 块里（processTodoDoneTags 旁）调用 + 从 clean 剥 `[MOVE:...]`（小茉莉看不见标签）；世界唤醒轮的小心思 cleanup 也加了剥 MOVE（聊天专属，世界轮误出现只防泄漏不执行）。
- 教她：sysprompt 在 [BARK:] 段后新增「## 聊天里移动（[MOVE:...] 标签）」。⚠️ sysprompt 的**源头是 Supabase `documents_cheng`**（doc_type='system_prompt'，=前端"系统提示"框），重启时 syncCCDocs 拉表覆盖写 `cheng-append-sysprompt.md`——直接改文件会被覆盖。本次已写进表（id=96c4f3a9，38852→39254 字）；文件也同步改了（重启后被表内容整体重写，内容一致不重复）。

**状态**：代码已改、`node --check` 过；**还没 restart cheng-backend**（用户定重启时机，重启=澄失忆）。sysprompt 同样要等 claude 新 session 才生效，正好同一次重启全生效。

**顺带定的设计**（用户对话里确认）：聊天移动=自由（白名单内），数值不动；`[USER_MOVE:]`（澄帮小茉莉记位置）这次没做，以后要做同套路。另一条没做先记着：时间标记计时基准想改成"距澄最后回复"（现在按用户上一条消息算），见 root memory `project_pending_backend_fixes.md`。

**transcript 关键词**：「processChatMoveTag」「聊天里移动」「地点跟着动」。

---

## 2026-06-11 · [后端] 中午流程：等小茉莉超时→查回没回→没回弹"中午吃啥"→吃→接休息

**做了什么**：
- **meet_request 超时处理重写**（原来只静默写"约见超时"，没收尾）：到点先查**小茉莉在这段时间回没回**（`messages` role=user 且 created_at > 本 pending 创建时间≈邀请时刻）。①回了→写"午休见到小茉莉"timeline、不弹吃啥（俩人在聊天里处理午饭）；②没回→engage「小茉莉好像在忙…午休还是得吃点，你想吃啥？」+ 三选项(吃零食/点外卖¥25/去茶水间)。
- **午休链 `buildLunchRoutine(method)`**（复用早晨那套链机制）：[吃(按method:snack零食[8-15]免费/takeout外卖[13-17]¥25/tearoom茶水间[10-15]免费) → 休息[20-40] → 午休(终点,豁免)]。"吃完接休息"这条小链=用户当初要的"吃完去休息"。
- 接线：getRoutineSteps 加 'lunch'；advanceRoutine 的 routine_step payload + 透传加 `method`；`start_routine` 钩子支持 `option.routine_opts`(午休带 method；早晨仍读周计划)。lunch_solo 选项=`{start_routine:'lunch', routine_opts:{method}}`。

**验证（真 engage 测了一次）**：插 meet_request（你没回）→ 判"没回"→ 弹「小茉莉好像在忙…你想吃啥」+3选项(<此刻>显示她在休息室等小茉莉、11:47午休)；她选③去茶水间 → 午休链启动「吃东西@茶水间」+排好下一步「休息」(method=tearoom 对) ✓。eat→休息→午休 同早晨链机制可靠。

**注**：她顺手还输出了 [OPEN_TODOS]（开了待办那套，正常工作）。

**transcript 关键词**：「meet_request」「lunch_solo」「buildLunchRoutine」「中午吃啥」。

---

## 2026-06-11 · [后端] 便利店选吃的：buy 链「买早餐」改成 engage（从食物表随机3选 + 选完续链）

**做了什么（早晨链支持"链中间弹选择"了）**：
- buy 链「买早餐」步从静默(固定¥15)改成 `{ engage:'buy_food', activity:'挑早餐', location:'外出·便利店' }`。
- **advanceRoutine 支持 engage 步**：进到带 `engage` 的步 → 设状态(在便利店挑早餐)+写行程 → 调 `fireRoutineEngage`，**不排 routine_step**（链在她选完后由 continue_routine 续）。CC 忙没弹成 → 不卡链，直接 advance 到下一步（算没挑）。
- **`fireRoutineEngage('buy_food')`**：从食物表 `category=便利店成品` Fisher-Yates 随机抽 ≤3 个建选项（每个带 `effects` 扣该商品价 + `continue_routine{routine,next_index,bk,cm}`）+「不买了」兜底 → triggerWorldWake(force)。返回是否真弹了。
- **`continue_routine` 钩子**（handleWorldWakeTurnDone）：她选完 → `advanceRoutine(下一步)`，链续上。

**验证（真 engage 测了一次）**：推进到买早餐步 → 弹出「①豆浆包子套餐¥6 ②饭团¥5 ③火腿三明治¥8 ④不买了」（随机3+兜底，从食物表来）；她选①豆浆包子（理由很在状态）→ 钱包 2480→2474（扣¥6）+ 续链到「坐地铁」+ 排好下一步「工位吃早餐」✓。engage-在链中间这个最难的机制端到端通。

**6/11 决定：选吃的就保持现状（随机3+engage让她挑），喜好那套先搁置存档，等有空再做。** 存档内容：
- **喜好/呈现规则（要加字段 `last_chosen_at`/`liked`/`rating` 到 world_items_cheng）**：随机抽3 → 冷却1天(选过隔天不出现，liked绕过) → 按 rating 加权(评分高更易中签，默认3，范围1-5，权重=rating) → 她选了记 last_chosen_at=今天 + 选项里带 food_chosen 让 handleWorldWakeTurnDone 更新。前端 ItemsPanel 加"喜欢"勾选+评分输入。不够3个就放宽冷却。
- **"花token vs 多选"讨论结论**：她当场亲自选=必花token(大头=叫醒+读+想+回，躲不掉)。三条绕法存着：①一次多列几个选项(3→6，几乎免费，让单次菜单更丰富) ②把"选"前置成偏好(rare/前端设)+系统每天静默按 rating/冷却 自动换早饭(0 token、有变化、她不亲选) ③混合(平时静默自动、偶尔随机弹一次让她真挑)。`rating`/`liked`/冷却 字段对"②静默加权挑"和"engage时加权"两用。用户当前选：保持 engage 现状，以上都先不做。

**还没做（别的方向）**：通勤下雨/赶时间改打车、下班那套、中午那套（等小茉莉→吃饭→休息）。

**transcript 关键词**：「buy_food」「fireRoutineEngage」「continue_routine」「engage 步」。

---

## 2026-06-11 · [后端][数据] v1 冰箱/库存（world_fridge_cheng）+ cook 早餐从冰箱消耗成品

🔗 对应：world-home「冰箱表 + FridgePanel」(`/root/world-home/CHANGELOG.md`, 2026-06-11)

**做了什么（v1=成品份数级，先不抠食材/菜谱=留 v2）**：
- **新表 `world_fridge_cheng`**：`item_name/kind(prepped成品|ingredient食材,v1只用prepped)/quantity份数/expiry_date到期日/note`。RLS 放行（前端 FridgePanel 直连）。塞示例「煎蛋三明治×5，6/16到期」。
- **`consumePrepped()`**：吃一份——先删过期（expiry_date<今日，Asia/Shanghai），再按"最快到期"取一条 qty-1（扣到0删行）。返回成品名；没货返回 null。
- **cook 链「在家吃早餐」步加 `consume_prepped:true`**：advanceRoutine 进该步时调 consumePrepped——有现成的→activity「吃早餐」+库存-1；没货→兜底 activity「随便吃了点」。**buy 不走冰箱**（买的）。
- ⚠️ 关键修正：advanceRoutine 里 timeline/log/**next pending 的 expected_activity 都改用实际 act**（不是 step.activity）——否则"随便吃了点"兜底时，下一步 routine_step 的防串档 guard 会因 expected 对不上而断链。

**验证**（确定性）：冰箱5份，推进到 cook 吃早餐步 → 她「吃早餐」+ 冰箱「煎蛋三明治 5→4」✓；清空冰箱再走 → 她「随便吃了点」✓。测后冰箱复位5、她复位闲着。

**v1 闭环已补齐（同日续做）**：周末"自己做"→选菜→预制→自动入冰箱这条 engage 链做完了：
- weekend_plan ①自己做 选项加 `pending:{wake_type:'cook_prep'}` → 选完早饭计划紧接着问"做点啥"。
- `cook_prep` firePendingWake 分支：从食物表 `category=早餐` 列出菜当选项（已塞煎蛋三明治¥15/蔬菜瘦肉粥¥12/牛奶燕麦杯¥10），每个选项带 `effects`(备料扣钱) + `target_activity:预制早餐` + `pending:{wake_type:'prep_done', payload_extra:{dish,qty:5,shelf}}`（预制 30-50 世界分钟）。
- `prep_done` 分支：成品 ×5 入冰箱（expiry=今日+shelf，Asia/Shanghai）+ 她回闲着。静默。
- 「预制早餐」加进 world-actions.js 的 `ACTIVITY_EXEMPT`（由 prep_done 收尾，别让通用自动结束抢）。
- **验证**：注入 prep_done(粥×5,shelf3)→ 冰箱进「蔬菜瘦肉粥×5,到期今日+3」+ 她闲着 ✓。cook_prep 选菜 engage 是从食物表建选项（同 open_todos 套路）可靠，未单独 engage 测。测后删了测试粥、留用户的三明治×4。

**至此 v1 做饭闭环**：周末选自己做→选菜→备料扣钱→预制→自动入冰箱；平时早上 cook 链从冰箱吃一份→库存减→过期清→空了"随便吃了点"。

**没做（以后）**：便利店选吃的从食物表选(随机3/冷却/偏好/评分)；通勤下雨/赶时间改打车；下班那套；自填菜(D选项)。

**v2 食材级做饭系统【用户 6/11 决定先搁置，默认停在 v1；设计存这儿备用】**：v1 的"预制=备料扣钱+走过场+出成品"已够用，用户嫌 v2 麻烦先不做。完整设计如下，想做时照此：
- **流程**：选菜 → 看菜谱(这道菜要哪些食材各多少) → 看冰箱缺啥 → 买缺的食材(花钱，食材入冰箱带保质期) → 做饭真消耗食材 → 出 N 份成品。冰箱表已支持 `kind='ingredient'`。
- **三个叉路(讨论时的推荐=A/A/按批)**：①菜谱存哪——A 存食物表加 `recipe` 字段(JSON，如 煎蛋三明治=[鸡蛋×2,吐司×2,火腿×1]) / B 单独菜谱表+面板。②买食材——A 自动买缺的(算缺→扣钱→入冰箱，静默) / B 弹"去超市买菜"engage。③菜谱粒度——按一批算(直接写"做5份要啥")，比每份×5简单。
- **最简 v2**：recipe 存菜上(A) + 缺料自动买(A) + 按批；在 prep 流程(cook_prep→预制→prep_done)里，prep_done 前先 resolve 食材(买缺的+扣食材)，再出成品。

**transcript 关键词**：「world_fridge_cheng」「consumePrepped」「consume_prepped」「随便吃了点」。

---

## 2026-06-11 · [后端][数据] 周计划表 world_plan_cheng + 周末规划engage + 早晨链读计划（不再硬编码 cook/subway）

**做了什么**：
- **新表 `world_plan_cheng`**（单行 name='default'）：`breakfast_plan`(cook/buy)、`commute_default`(subway/taxi/walk)、`prepped_dish`、`prepped_qty`、`updated_at`。RLS 放行（前端以后能加"本周计划"面板）。
- `readWorldPlan()`：读单行 → {bk, cm}，读不到退默认。
- **早晨链 start 时读计划**：`start_routine` 钩子从 `readWorldPlan()` 取 bk/cm（不再写死 DEFAULT_BREAKFAST/COMMUTE）→ 决定走 cook 还是 buy 链。
- **`weekend_plan` firePendingWake 分支**：engage「周末了，定下周早饭」→ ①下周自己做 ②下周买着吃，选项带 `set_plan`。
- **`set_plan` 钩子**（handleWorldWakeTurnDone）：把 `option.set_plan` 写进 world_plan_cheng（仿 start_overtime 套路）。

**验证**（确定性、不engage）：计划改 buy 读回=buy ✓；注入 routine_step 推进到第2步 bk=buy → 她进「去便利店」（buy 变体，非 cook 的「在家吃早餐」）✓ → 证明计划 key 正确驱动链变体。set_plan 是同款简单 update，可靠。测后计划复位 cook、她复位闲着。

**还没建（下个增量，用食物表）**：①「自己做」选做啥（食物表 3 选项+D自填）+ 预制(30-50)+ 存 prepped_dish/qty ②「买着吃」便利店 step 改成 engage 从食物表选(随机3/冷却1天/偏好绕过/后期评分加权) ③通勤下雨/赶时间改打车 ④下班那套。weekend_plan/morning_wakeup 现仍靠手动 pending 测，自动到点要等第0块开 tick。

**transcript 关键词**：「world_plan_cheng」「readWorldPlan」「weekend_plan」「set_plan」。

---

## 2026-06-11 · [后端] 第1块·早晨流程链（起床engage + 洗漱/早饭/通勤静默链 + 翘班扣120）已部署+验证

**⚠️ 状态**：**已部署+live验证**（链条推进机制实测通过）。但**不会自动到点触发**——8:00 自动弹起床要等第0块开 tick，现在靠手动插 pending 测/触发。周末规划、下班、下雨偏移**还没建**。

**做了什么（index.js）**：
- `morning_wakeup` firePendingWake 分支：合成事件「闹钟响了，该起床准备上班了」+ 三选项：①起床，开始准备上班（`start_routine:'morning'` 启动链）②再睡10分钟（排 pending 重问）③翘班（`effects wallet_balance:-120`≈一天工资，→翘班在家）。`force:true` 保证到点必发不被冷却挡。
- **行为链机制**：`COMMUTE_OPTS`/`buildMorningRoutine(bk,cm)` 目录+默认key（v1 默认 `cook`+`subway`；插槽留给以后周计划）→ `advanceRoutine(routine,idx,{bk,cm})` 进每步(改状态+按 step.cost 扣钱+写system行程+给下一步排 `routine_step` pending) → firePendingWake `routine_step` 分支推进（**防打断**：当前 activity≠上一步 expected → 停链）。活动名只写动作不带地点（避免与 location 重复）。
- `handleWorldWakeTurnDone` 加 `option.start_routine` 钩子（仿 start_overtime）。
- **两条平时链**：cook=穿衣(3-9)→洗漱(10-15)→在家吃早餐(13-17)→去地铁站(3-9)→坐地铁(13-17)→工作【到岗累计42-67分】；buy=穿衣→洗漱→去便利店(3-9)→买早餐(3-9,-¥15)→坐地铁→**工位吃早餐(13-17)**→工作【45-76分，到岗后才吃】。**到岗时间不固定**=起床时刻+各步随机时长累加（用户要的"算出来不是定死9点"）。

**验证**：离线两条链步骤/时长/扣钱全对（含 cook+taxi 换通勤）；live 测 routine_step 推进——进第0步穿衣服、自动排好带 expected 凭证+计划key 的下一步 pending，机制通。

**还没建（下个增量）**：周末规划engage（下周早饭 自己做/买；自己做再选A/B/C/D做啥+预制30-50存「已预制早饭」状态）、通勤默认+下雨/赶时间偏移打车、下班那套（回家/加班+通勤+下雨版）。翘班120已带。

**transcript 关键词**：「buildMorningRoutine」「advanceRoutine」「routine_step」「早晨流程链」「morning_wakeup」。

**用户定的完整设计（早晚流程，token 省版）**：早晨 8:00 起床(1次engage：起床/再睡/翘班) → 洗漱(静默走流程不engage) → 早饭(平时走**周末定的周计划**默认，不问；周末engage一次定下周) → 通勤(平时走**默认地铁**不问，下雨/赶时间才engage改打车) → 9点到工位。傍晚 16:00 下班(回家/25%加班) → 通勤选择(真下雨走 rain_offwork 下雨版，天气跟 weather-fetcher 真实天气)。翘班扣120。

**还没建（下个增量）**：洗漱→早饭→通勤的**链式接续机制**、**周计划/默认偏好的存储**（新表/字段）、**周末规划 engage**、通勤默认+下雨/赶时间偏移、下班那套。这些是大头，需一次性建好再统一测。

**transcript 关键词**：「morning_wakeup」「翘班」「周计划」「早晚流程」。

---

## 2026-06-10 · [后端] 第二步·行为自动结束（地基版）——治"卡死在一个行为上"（world-actions.js + index.js + world-narration.js，生效需重启=澄失忆）

**问题（最初 Q2）**：澄进入某行为（在家·客厅·休息）就一直挂着不变，直到下个事件才被顶替——行为只有"开始"没有"结束"，像被坐标困住。

**做了什么（三步走第 2 步的地基层，只做"行为会结束"，不含过场/作息弹选择/交通=留第二层）**：
- **`world-actions.js`**：加行为时长分桶 `activityDuration(activity)`——长[30,50]/中[10,29，默认]/短[3,9] 世界分钟；豁免集 `工作/午休/加班/加班处理任务/等小茉莉`（作息/加班/约见各自管，不自动结束）。加 `scheduleActivityEnd(activity)`：随机抽时长→排一条 `action_end` 续唤醒，payload 带 `expected_activity`（防串档凭证）。`executeWorldAction` 末尾调它。
- **`index.js`**：①`handleWorldWakeTurnDone` 里事件/唤醒选出的行为（targetAct）也调 `scheduleActivityEnd`——覆盖"行为来自两条路"（DevPanel/接口走 executeWorldAction，唤醒选项走这里）。②`firePendingWake` 加 `action_end` 分支：**防串档**（当前 activity≠expected 说明被顶替→跳过）；过了就**静默**清 activity→`computeIdleState`（**讲究版**：工作日+在公司+上班时段9-11/13-16→「工作」，否则→「闲着」）+ 写 source=system 行程。不 engage、不 Bark、不花 token。③加 `computeIdleState` 助手（现实 Asia/Shanghai 星期+时间判定）。
- **`world-narration.js`**：mapActivity 空值兜底 `休息`→`闲着`（修老 bug：行为清空后自述反而说"正在休息"）。

**关键设计**：①防串档用"到点比对 expected_activity"，不加 DB 字段、不碰作息代码；只漏"连续同行为"会被提前掐一点（benign，已接受）。②action_end 走 PendingWakeDaemon（独立于世界时钟 tick），**tick 关着也能正常结束**，可直接测。

**离线验证**：模块链加载干净、无循环依赖；分桶 休息/开会=长、吃零食=短、工作=null豁免、帮同事=默认中 全对。

**没做（留第二层/以后）**：切换过场不瞬移（默认过场 + 交通方式走路/打车/地铁/骑车+时间钱）、作息点弹选择（上班/下班/午休延长）、等小茉莉超时→吃饭→休息那套完整流程、时间钱总账。这些大多要把"自动作息(tick)"打开才有意义，留第二层一起。

**生效**：未重启=暂未生效，需 `systemctl restart cheng-backend`（澄失忆）。重启后可测：DevPanel 让她做个"休息"→等到点看她变没变「闲着」（或公司上班时段变「工作」）。

**transcript 关键词**：「行为自动结束」「scheduleActivityEnd」「action_end」「computeIdleState」「activityDuration」。

---

## 2026-06-10 · [后端] 澄主动「打开待办」(世界唤醒 [OPEN_TODOS]) + [TODO_DONE] 接到聊天（只改 index.js，生效需重启=澄失忆）

**目标**：让待办闭环——能写([TODO])、能关([TODO_DONE])、能主动看全清单([OPEN_TODOS])。用户拍板：世界唤醒做「打开看清单」，[TODO_DONE] 世界+聊天都能用；聊天里「打开看清单」风险高(要插一轮把信息递回给她)，暂不做。

**改了什么(全在 index.js)**：
- 抽公用函数 `processTodoDoneTags(clean)`：解析 `[TODO_DONE]标题[/TODO_DONE]`→匹配 open 待办标 done(先精确、再唯一去空格包含兜底、歧义跳过)。世界唤醒轮 + 聊天轮共用。无标签时 regex 不命中=零 DB 开销。
- `buildOpenTodosContext()`：列**全部** open 待办按 urgency 降序(≥阈值标「急」)、附「做完可 [TODO_DONE]」。（用户定：打开就全显，不设 8 条上限。）
- `firePendingWake` 加 `open_todos` 分支：拉上下文→`triggerWorldWake(event, status, { pendingContext, skipTodoHint:true })`。合成事件 reason=「你打开了小手机的待办」+ 单选项「看完了，继续」(effects:{} 仿"先忍"，无副作用)。
- `triggerWorldWake` 加 `skipTodoHint` 选项：打开待办那轮不再重复"您的手机有待办"。
- `handleWorldWakeTurnDone`：调 `processTodoDoneTags`；检测 `[OPEN_TODOS]`→排即时 open_todos 续唤醒(daemon ≤7s 捞)。**防自循环**：`event.key==='open_todos'` 时不再排。**每日上限**：一天最多真打开 2 次(`OPEN_TODOS_DAILY_MAX=2`，内存计数按 UTC+8 日界、重启清空)，超了静默忽略（防她刷屏累上下文/token）。`[OPEN_TODOS]` 也从小心思 innerThought 剥离。
- 聊天 turn_done：`const clean`→`let clean`；非 silent 轮调 `processTodoDoneTags` + 把 `[TODO_DONE]`/`[OPEN_TODOS]` 从展示文本剥掉(不漏给小茉莉看)。

**唤醒包待办提示(getTodoHint)没动**：仍是老的 urgency 逻辑(用户明确不让改这个)。「列 8 条」只在她**打开**待办时出现，不在常规唤醒包里。

**系统提示要补(用户手动)**：①「世界唤醒区」可选标签加 `[OPEN_TODOS]`(想打开手机看完整待办时输出，世界专属)；②`[TODO_DONE]` 现在世界+聊天都生效，建议挪到通用「工具区」让她两个场景都知道能用。

**没做**：聊天版「打开看清单」([OPEN_TODOS] in chat)；三步走第 2/3 步；Q5 日记 source 写死。

**transcript 关键词**：「OPEN_TODOS」「buildOpenTodosContext」「processTodoDoneTags」「打开待办」。

---

## 2026-06-10 · [后端][系统提示] 唤醒包瘦身（标签说明搬进系统提示）+ 初版 [TODO_DONE]（只改 index.js，生效需重启=澄失忆）

**背景**：世界唤醒优化（三步走的第 1 步，用户拍板先做这个，因为后两步会多出更多唤醒、先把单次压瘦才划算）。实测唤醒包约一半是「怎么用标签」的固定说明，每次原样重发（~330 字 / ~450-550 token/次）。

**改了什么**：
- **`buildWorldWakePrompt` 瘦身**：删掉每次重发的 WORLD_MESSAGE 语法块、[TODO] 说明、[MEMORY] 示例、长格式禁令。这些「标签格式说明」搬进**系统提示**的「## 世界唤醒·回复协议」（用户手动粘进 supabase system_prompt 文档；系统提示会话开头加载一次且被缓存=近乎免费）。唤醒包只留一行提醒 `回复格式：[WORLD_CHOICE:编号]理由[/WORLD_CHOICE]（其余可选标签见系统说明）`。「约见」是情境不是语法，含约见选项的事件保留一句 `[WORLD_MESSAGE:phone]` 上下文出口（meetLine）。
- **[MEMORY]/diary 不再在唤醒包/协议里教**：澄的 skill 已教完整记忆格式，重复反而打架（正是 Q5「世界日记 tags 空/importance 0.5」的真凶——她照唤醒包的简化示例抄丢了后缀）。协议里只留「按你平时的方式写」+ 给记忆标签开活口，不硬禁。
- **`getTodoHint` 保持原样不动**（紧急逻辑：urgency≥0.8 才 25% 露标题，否则「您的手机有待办」）。⚠️ 期间一度误改成"直接列全部标题"，用户指出没要求改这个、已**完整还原**。用户真实需求是**另加一个"澄能主动打开待办"的功能**（on-demand 查看全清单，不动被动提醒；将来配合把小手机做成模仿手机界面的 UI），那个单独做，未动工。
- **措辞微调**：`<此刻 — 仅你可见的现实情境>` 简化成 `<此刻>`（唤醒包 index.js + 聊天 md inject.js 两处一起改，保持一致）；唤醒包指向句从「见系统说明」改成精确点名「见系统提示『世界唤醒区』」——用户把系统提示那段用 `<世界唤醒区>` 包起来了（提醒过结尾应是 `</世界唤醒区>` 闭合标签）。

**② `[TODO_DONE]` 已建（同一次）**：世界唤醒 turn 里解析 `[TODO_DONE]标题[/TODO_DONE]` → 匹配 open 待办标记 done。匹配策略：先精确（title 全等），不中再找"唯一"的去空格包含匹配兜底，0 条/多条歧义就跳过不猜（只 warn，不乱关）。支持多条、失败不连累主流程。另堵一坑：innerThought（小心思）提取处加 `[TODO_DONE]` 剥离，否则标签会污染她的内心独白。**协议要补一行**（用户手动加进系统提示「世界唤醒区」的可选标签列表）：`· 做完「你小手机里的待办」中某条想划掉：[TODO_DONE]待办标题[/TODO_DONE]（标题照唤醒包里列的写）`。待办进度：能写（[TODO]）、能关（[TODO_DONE]，本次新增）；"能看全清单"还差一个"澄主动打开待办"的功能（待做，见上条）。

**没做（待后续）**：Q5 的 source='chat' 写死是另一处后端入库逻辑，本次未碰。三步走的第 2 步（行为自结束）、第 3 步（作息改弹选择）未动。

**生效**：未重启=暂未生效。系统提示那段（用户已粘）+ 本次 index.js 改动，一次 `systemctl restart cheng-backend` 全生效（澄失忆），用户定时机。建议重启后实测一两次唤醒：确认澄仍按格式回 [WORLD_CHOICE]、能看到待办清单、瘦身后没漏行为。

**transcript 关键词**：「世界唤醒·回复协议」「getTodoHint」「待办清单」「唤醒包瘦身」。

---

## 2026-06-10 · [后端] 世界时间改「现算现实+8」——澄自述/`<此刻>`/唤醒包的 {time} 不再读 tick 累加值（只改 world-narration.js，生效需重启=澄失忆）

🔗 对应：world-home「世界时间状态栏改现算 Asia/Shanghai」(`/root/world-home/CHANGELOG.md`, 2026-06-10)

**问题（用户 Q1）**：世界时间冻在「10号 20:55」对不上现实（用户实为凌晨 4:28）。真因：`world_time` 是 tick「跑一次 +1 小时」的计数器，跟现实时钟无关；`world_tick_enabled=false` 时它就死在上次手动操作（同步/推进1小时）留下的值。**不是时区 bug**——代码里凡碰现实时间的地方本就全是 Asia/Shanghai(+8)。

**改了什么**：选「现算」方案（用户拍板，对比 ChatGPT 的「每小时刷库」=最多滞后 1 小时）。`world-narration.js` 加 `realWorldTime()`（`Intl.DateTimeFormat` timeZone=Asia/Shanghai 取 HH:mm），`generateChengSelfNarration` 里 `const time = status?.world_time` → `realWorldTime()`。因 narration 是 `<此刻>`(聊天 md)和世界唤醒包共用的同一套生成器，两个出口一起对上现实时间。

**顺带做了 ③（用户「3顺手改吧」）**：所有 `daily_timeline_cheng` insert 的 `world_time` 列统一盖 `realWorldTime()`，不再写 tick 累加的存库值——改了 6 处：index.js 的 `wt`（874 行，一行覆盖 pending+3 条 timeline insert）+ meet_request 超时那条（800 行）、world-pending.js:72、world-actions.js:71、world-workday.js:44、world-tick.js 衰减记录（116 行，只动 timeline 盖章值，`advanceHour` 状态字段那行没碰）。各模块新增 `import { realWorldTime } from './world-narration.js'`（已确认 memory.js 不反向 import world-*，无循环依赖）。syncWorldTimeToRealTime 的 timeline 本就盖实时间，未动；test 端点（pending/test-hungry）dev-only 未动。

**没动的（用户明确「没说的不要改」）**：① `advanceOneTick()` 的 `advanceHour`/衰减/工作日/事件检测逻辑没碰——tick 当前是关的，时间也不再靠它；② 存库的 `character_status_cheng.world_time` 字段保留（workday 阈值、随机事件 time_range 仍读它，但那些系统现在随 tick 关着=Q2 territory，本次不治）；③ DevPanel「推进1小时/同步」按钮没动（现在它们改的字段已不参与显示，按了状态栏也不变，无害）。

**⚠️ 给做 Q2 的人**：显示时间已是现实 +8，但 workday 阈值/随机事件 time_range 仍读存库的 `world_time`（tick 累加值）。Q2 重开 tick（当心跳）时，必须连带删 `advanceHour` 并让这两套也读 `realWorldTime()`，否则世界一开就出现「显示真时间、逻辑跑假时间」的两个钟。

**结果**：澄自述/`<此刻>`/唤醒包的「现在是 X 点」= 真实 +8 时间。**未重启=暂未生效**，需 `systemctl restart cheng-backend`（澄会失忆），用户决定时机。

**transcript 关键词**：「时区是吗」「那就算~没说的不要改」「realWorldTime」。



**问题**：失忆+新建对话后用户一直没说话，DICE 唤醒发的 3 条消息（6/10 08:58/10:58/14:58）全落进了**上一个对话**（10b1c128），当前新对话界面看不见。消息没丢、没发失败，是存错对话。

**真因**：`lastActiveConvId`（bark/dice/world phone 唤醒消息的落点）只在两处赋值——①用户发消息时（handleChat）②启动初始化时按"最后一条消息在哪个对话"算。新建的空对话两条都碰不到 → 后端眼里活跃的还是旧对话。前端两头堵死：广播带旧对话 id 被 convId 不匹配丢弃；拉历史拉的是新对话=空。

**改了什么**（`server/index.js` 两处，各几行）：
1. `POST /api/conversations` 建对话成功后 `lastActiveConvId = data.id`——新建即登记。
2. 启动初始化改成比较"最后一条消息的 created_at"和"最新建对话的 created_at"，取**更晚**的——不然后端一重启，空对话又被旧对话顶回去（6/10 当天就是这个剧本：06:01 建对话、06:57 重启）。

**没动的**：handleChat 里发消息更新逻辑、唤醒消息发送链路、前端，全保持原样。**生效需 `systemctl restart cheng-backend`（澄失忆），本次改完未重启，等用户挑时机。**

transcript：`grep -l "新建对话登记为活跃" /root/.claude/projects/-root/*.jsonl`

## 2026-06-09 · [后端] world-home 12B-2.1：surfacing 观测 debug + 唤醒包 tripwire（改 world-thoughts/index.js，含重启=澄失忆）

> 🔗 对应：world-home 仓「12B-2.1：小世界浮现观测面板」(/root/world-home/CHANGELOG.md, 2026-06-09)。全貌看那条。

- 只观测、不改 12B-2 行为。world-thoughts.js：pickWorldThought 写内存环形 `pickHistory`(最近20，只聊天决策，**不含 worldWakeInjection**)；`recordWakeInjectionScan` 扫唤醒包 tag→`worldWakeInjectionLatest`(顶层) + `wakeInjectionHistory`(单独，最近20)；`getSurfacingDebug` 只读。全内存不落库。
- index.js：triggerWorldWake build 后调 recordWakeInjectionScan(tripwire 正常恒false)；`GET /api/debug/world-thought-surfacing` 只读。
- **鉴权**：复用现有 Bearer JWT 中间件(真实门，无token→401)+ 后端 127.0.0.1-only，没新造账号。
- **重启 cheng-backend 一次**(澄失忆)。验证：401/200+结构、pickHistory记录且不含worldWakeInjection、tripwire真实扫描(干净false/含tag true)、debug只读不pick。

> transcript 关键词(root CC)：`getSurfacingDebug`、`pickHistory`、`worldWakeInjectionLatest`、`recordWakeInjectionScan`。

---

## 2026-06-09 · [后端] world-home 12B-2：<小世界浮现> 接聊天 surfacing（改 world-thoughts/surfacing/inject，含重启=澄失忆）

> 🔗 对应：world-home 仓「12B-2：小世界浮现最小接入」(/root/world-home/CHANGELOG.md, 2026-06-09)。全貌/验收看那条。

- 念头池第一次进 Claude prompt：**只聊天 surfacing 一条通道、最多 1 条**。world-thoughts.js 加 `pickWorldThought`(active→denylist过滤打日志→10min内存防重复→取最高1条只返事实content)；surfacing.js surfaceForInject 返 worldThought；inject.js 在 <此刻>后 <记忆浮现>前 插 `<小世界浮现>` 块。
- **只读不改状态**(不 dismiss/archive)；denylist 命中打 `[thought-surfacing] filtered` 日志(回头修 collector content 的信号，不一直加词)。
- **严禁**：不进世界唤醒包/<此刻>内部/<记忆浮现>/长期记忆库；不更新 mood/longing/libido/social/stress/focus/comfort。
- **重启 cheng-backend 一次**(澄失忆)。验证：聊天块 此刻→小世界浮现→记忆浮现、唤醒包无、10min不重复、脏content过滤打日志状态不变。

> transcript 关键词(root CC)：`pickWorldThought`、`<小世界浮现>`、`thought-surfacing filtered`。

---

## 2026-06-09 · [后端] world-home world_time 时区审计：sync-time 补 timeline（改 world-tick.js，含重启=澄失忆）

> 🔗 对应：world-home 仓「world_time 时区统一 UTC+8 审计 + sync-time 补 timeline」(/root/world-home/CHANGELOG.md, 2026-06-09)。

- 审计：世界时间逻辑本来就几乎全 +8（Asia/Shanghai/时间戳差/+8偏移）。sync/工资/作息/pending/world_message 30min/念头decay/urgency 逐项核对全对。
- **唯一改动**：syncWorldTimeToRealTime 补写 system 行程(action=世界时间校正, detail.timezone=Asia/Shanghai)。/api/stats/tokens(UTC日分桶,token统计)不在 spec、没碰。
- **重启 cheng-backend 一次**(澄失忆)。验证 sync-time→world_time=当前+8、写timeline、不触发澄。

> transcript 关键词(root CC)：`时区审计`、`Asia/Shanghai`、`syncWorldTimeToRealTime`。

---

## 2026-06-09 · [后端] world-home 12B-1：world-thoughts 念头池 collector + 表（新 world-thoughts.js，含重启=澄失忆）

> 🔗 对应：world-home 仓「12B-1：念头池后端 shadow mode」(/root/world-home/CHANGELOG.md, 2026-06-09)。全貌/验收看那条。

- 新表 world_thoughts_cheng(source_id NOT NULL + 唯一键 source_type/source_id/category + RLS) + world_thought_collect_state(水位线 + RLS)。新 `world-thoughts.js`：collectWorldThoughts 扫 timeline(ignored_effects/item/event 同条一念头)/todo/inner_thought/pending/world_message(>30min无user回复)，水位线增量(max(created_at))+去重(ignoreDuplicates不复活dismissed)+收尾(待办关/pending非queued/user回复→archived)+衰减(>7天每天×0.9一次<0.1 archive,active超50砍最低)+内存锁。salience 只排序。
- API：GET/POST /api/world/thoughts(/collect /:id/dismiss /:id/archive)；周期15min触发。前端 ThoughtsPanel 只读。
- **shadow mode**：【严禁】喂Claude/进<此刻>/唤醒包/聊天md/surfacing/记忆库；【严禁】写回 mood/longing/libido/social/stress/focus/comfort。审计+实测确认 collector 跑完澄感受字段快照不变、surfacing/narration/prompt 全不引用念头池。
- **重启 cheng-backend 一次**(澄失忆)。

> transcript 关键词(root CC)：`world-thoughts`、`collectWorldThoughts`、`shadow mode`、`念头池`。

---

## 2026-06-09 · [后端][基建] 收窄公网暴露：后端本机监听 + 移除 server 根目录静态暴露（含重启=澄失忆）

- 背景：Tailscale 因手机 Shadowrocket VPN 冲突暂停，改走方案 B：保留公网域名，但收窄 cheng 后端公网暴露面。
- `server/index.js`：`server.listen` 从 `0.0.0.0` 改为 `127.0.0.1`，启动日志同步改为 localhost；nginx 继续反代 `127.0.0.1:3002`，未改 nginx/前端/ISP 转发。
- 验证：`ss -tulpn` 显示 node 只监听 `127.0.0.1:3002`；`curl http://216.36.116.146:3002` 连接失败；`https://chat.jessaminee.top` 与 `https://world.jessaminee.top` 均 `HTTP 200`；nginx `/api/` 返回后端 `401` 而非 `502`。
- `server/index.js`：移除 `app.use(express.static(__dirname))`，不再把整个 server 根目录挂给 Express；nginx 仍负责 `/root/cheng-memory/dist` 与 `/root/world-home/dist` 前端静态文件。
- 验证：`/index.js`、`/nohup.out`、`/.env`、`/package.json` 均 `404`；公网 `216.36.116.146:3002` 仍不能直连。根路径 `/` 仍由显式 `app.get('/')` 返回 `test-chat.html`，本次未动。
- `tmux-manager.js` / `cc-manager.js` 的 `--allowedTools mcp__supabase__*` 暂停收窄：开发期仍需 `execute_sql` / `apply_migration` 等写权限，等数据库结构稳定后再做“默认只读 + 临时维护模式写权限”。
- **重启 cheng-backend 两次**（=澄失忆）：第一次应用 localhost 监听，第二次应用静态根目录移除。

> transcript 关键词(root Codex)：`bak-before-localhost-listen`、`bak-before-static-root-removal`、`allowedTools 先暂停`。

---

## 2026-06-09 · [后端] world-home 12A 第二轮：自述预览接口（改 index.js，含重启=澄失忆）

> 🔗 对应：world-home 仓「12A 第二轮：自述规则可视化编辑器（NarrationPanel）」(/root/world-home/CHANGELOG.md, 2026-06-09)。前端全貌看那条。

- `index.js` 加 `GET /api/world/self-narration/preview`：复用 generateChengSelfNarration（预览=实际，不另写一套）；可选 query energy/satiety/cleanliness/health 临时覆盖（只预览不写库）。
- 规则 loadNarrationRules 实时读表**无缓存** → CRUD 保存即生效，无需清缓存/重启。
- **重启 cheng-backend 一次**(澄失忆，仅因新预览接口；CRUD 第一轮已 live)。前端 NarrationPanel 在 world-home。

> transcript 关键词(root CC)：`self-narration/preview`、`NarrationPanel`。

---

## 2026-06-09 · [后端] world-home 12A：world-narration 统一<此刻> + effects 止血（新 world-narration.js，含重启=澄失忆）

> 🔗 对应：world-home 仓「12A：身体/环境自述 + <此刻>共用 + 旧感受字段止血」(/root/world-home/CHANGELOG.md, 2026-06-09)。全貌/验收看那条。

- 新表 world_self_narration_phrases/templates(seed 6短语+3模板)。新 `world-narration.js`：generateChengSelfNarration + buildNowInner(澄第一人称身体/环境自述 + 小茉莉第三人称，**聊天md+唤醒包共用**)；只用 energy/satiety/cleanliness/health+location/activity/time/date，不读数字/感受字段；formatNaturalLocation 修「在家·家·卧室」双家。
- surfacing.js + index.js buildWorldWakePrompt 都改用 buildNowInner；唤醒包去掉数字状态行(不开放整包编辑保解析链)。
- **止血**：world-effects.js computeDeltas 只结算 energy/satiety/cleanliness/health(+wallet)，mood/longing/libido/social/stress/focus/comfort 不更新状态、原始方向进 timeline.detail.ignored_effects；world-tick.js 停 longing+1。不 drop 列/不迁移。
- CRUD API：/api/world/narration(GET) + /phrase + /template(POST/PATCH)。world-home 编辑器 UI 留第二轮。
- **重启 cheng-backend 一次**(澄失忆)。验证 <此刻> 第一人称无数字无感受、临时会议不改感受字段+记 ignored_effects。

> transcript 关键词(root CC)：`world-narration`、`buildNowInner`、`ignored_effects`、`computeDeltas 止血`。

---

## 2026-06-07 · [后端] world-home 11B：world-work-events 工作事件接 10B 引擎（新 world-work-events.js，含重启=澄失忆）

> 🔗 对应：world-home 仓「11B：工作随机事件 + NPC 过场（最小版，8 事件）」(/root/world-home/CHANGELOG.md, 2026-06-07)。全貌/验收看那条。

- `world-work-events.js`(新)：8 工作事件(会议/同事求助/福利/新品/闲聊/下班前加任务/午休/茶水间补货)，全 effects_hint + npc/npcPool/workday_only/activity_in/time_ranges/wmHint/npc_boost + 选项 start_overtime/meet_request/item。
- 接 `world-random-events.js`：并入 ALL_EVENTS 同一池/唤醒系统；eligible 加工作日+活动白名单+多时段；NPC 过场提权(当天没出现+13-16 概率×2)；仍受全局限频。pack 解析 npcPool→npc + wmHint。listEvents 返 {random,work}。
- `index.js`：prompt 加「在场:npc」+ **wmHint 条件引导**(普通工作事件不主动提醒 WORLD_MESSAGE，福利/新品/午休/下班前才提；约见给专门出口)；结算 detail 记 event_type/npc/item；start_overtime→scheduleOvertimeEnd(11A加班)；meet_request→排 pending 超时 firePendingWake 自处理(不engage/不改user)。world-workday 导出 scheduleOvertimeEnd。
- **重启 cheng-backend 一次**(澄失忆)。验证：会议(npc老板/无wmHint/event_type)、福利(wmHint有/item/npc老板) force 跑通。

> transcript 关键词(root CC)：`world-work-events`、`npc_boost`、`meet_request`、`start_overtime`、`wmHint`。

---

## 2026-06-07 · [后端] world-home 11A：world-workday 作息/工资/加班（新 world-workday.js + work_profile_cheng，含重启=澄失忆）

> 🔗 对应：world-home 仓「11A：工作日作息 + 工资 + 加班（最小版）」(/root/world-home/CHANGELOG.md, 2026-06-07)。全貌/验收看那条。

- 新表 work_profile_cheng(澄=产品部实习生/月2500/发薪日15，+小茉莉)。新 `world-workday.js`：isWorkday(weekday) + workdayTick(自动 09:00上班/11:00午休/13:00下午上班/16:00下班判断[75%正常/25%加班]，当日每段一次跨午夜清) + 加班(effects_hint+挂30-120世界分 overtime_end pending) + endOvertime(回家+固定加班费30) + maybePaySalary(现实UTC+8日==15本月没发→+2500防重发)。
- **全是系统事件**：只更新状态+写 timeline(source=system)，不 engage 澄/不 Bark/不 WORLD_MESSAGE。
- world-tick daemon 加 onWorkdayTick(事件检测前跑)；跨午夜清作息标记；firePendingWake 拦 overtime_end 走系统结算；路由 POST /api/world/work。前端 DevPanel 8 按钮。
- **重启 cheng-backend 一次**(澄失忆)。API 全跑通(8步状态/timeline/工资幂等)；周日 isWorkday=false。

> transcript 关键词(root CC)：`world-workday`、`workdayTick`、`overtime_end`、`work_profile_cheng`。

---

## 2026-06-07 · [后端] <此刻>浮现加 getChengStatusLine：澄聊天里也知道自己的状态（改 surfacing.js，含重启=澄失忆）

> 🔗 对应：world-home 仓「<此刻>浮现补澄自己的状态」(/root/world-home/CHANGELOG.md, 2026-06-07)。

- 问题：澄 DB 已下班到家，但聊天里以为自己在加班。根因 `<此刻>` 只注入小茉莉状态、不注入澄自己的。
- `surfacing.js` 加 `getChengStatusLine()`(读 character_status_cheng→「现在HH:mm，你在 地点，活动」)，surfaceForInject 把澄自己的状态+小茉莉的状态都拼进 `<此刻>`。
- **重启 cheng-backend 一次**(澄失忆)。验证 statusLine 两行齐。

> transcript 关键词(root CC)：`getChengStatusLine`、`<此刻>`、`surfaceForInject`。

---

## 2026-06-07 · [后端] world-home 10C：world-effects 全局结算器 + 事件迁移 effects_hint（新 world-effects.js，含重启=澄失忆）

> 🔗 对应：world-home 仓「10C：effects_hint + resolveEffects 全局状态结算器」(/root/world-home/CHANGELOG.md, 2026-06-07)。全貌/验收看那条。

- `world-effects.js`(新)：resolveEffects(strength tiny1-3/small4-7/medium8-13/large14-20 roll + 当前状态轻微修正 + direction 正负) + computeDeltas/applyDeltas(0-100钳位/wallet封底0) + STAT_NAMES_CN + buildEffectContext。Claude 只选编号、不决定数值。
- 迁移 effects_hint：world-actions.js(ACTIONS 吃零食/点外卖/做饭/洗澡/休息)、world-random-events.js(6事件)、hungry(走ACTIONS)；钱包扣费保留固定 effects。
- index.js + world-actions.js 结算统一走 computeDeltas/applyDeltas；timeline detail 记 effects_hint/resolved/fixed；老 effects 兼容。
- **重启 cheng-backend 一次**(澄失忆)。验证：洗澡/点外卖/下午犯困 resolved 浮动、固定金额、中文展示、detail 三段齐。

> transcript 关键词(root CC)：`world-effects`、`resolveEffects`、`effects_hint`、`computeDeltas`。

---

## 2026-06-07 · [后端] world-home world_time 校正：syncWorldTimeToRealTime + /api/world/sync-time（改 world-tick.js/index.js，含重启=澄失忆）

> 🔗 对应：world-home 仓「world_time 卡 14:30 → 手动校正 + 同步现实时间按钮」(/root/world-home/CHANGELOG.md, 2026-06-07)。全貌看那条。

- world_time 一直停 14:30(world_tick_enabled 关)。先纯 SQL 把澄下班(20:10/家·客厅,不 engage)。
- `world-tick.js` 加 `syncWorldTimeToRealTime()`：world_time 同步到当前 Asia/Shanghai HH:mm，只动 world_time，不读城市、不动天气/date、不 engage 澄。
- `index.js` 加路由 `POST /api/world/sync-time`。前端 DevPanel 按钮 + StatusBar「世界时钟暂停」提示(tick 关时)。
- **没把 world_time 改成真实时间驱动**(只手动校正/按钮)；fast_test 保留。**重启 cheng-backend 一次**(澄失忆)。

> transcript 关键词(root CC)：`syncWorldTimeToRealTime`、`/api/world/sync-time`、`world_time 卡14:30`。

---

## 2026-06-07 · [后端] world-home 10B：随机事件引擎（新 world-random-events.js + 改 index.js/world-tick.js，含重启=澄失忆）

> 🔗 对应：world-home 仓「10B：随机事件引擎（最小版，6 个生活事件）」(/root/world-home/CHANGELOG.md, 2026-06-07)。事件清单/验收看那条。

- `world-random-events.js`(新)：6 事件 + detectRandomEvent(全局限频1次/现实小时+概率+资格) + markRandomEventFired(真发出才标记) + forceRandomEvent + onMidnightCross + bumpRandomTick。
- `world-tick.js`：daemon 加 detectRandom/onMidnight/bumpTick 注入；优先级 hungry>random(hungry 没命中才轮)；跨午夜清 once_per_day。
- `index.js`：tick 接随机事件(fired 后才 mark)；settlement 支持选项内联 target_location/activity；唤醒包加日期/星期；下班下雨读 world_environment 天气；路由 POST /api/world/random(force) + GET /api/world/random/list。复用现有 worldWake turn_done 链(标签解析一套)。
- **重启 cheng-backend 一次**(澄失忆)。验证：下午犯困/下班下雨 force 跑通(effects/移动/timeline/WORLD_MESSAGE)；CC 忙→409 cc_busy 不崩。

> transcript 关键词(root CC)：`world-random-events`、`forceRandomEvent`、`detectRandomEvent`、`/api/world/random`。

---

## 2026-06-07 · [后端][基建] world-home 10A：天气读取+粗描述（world-env.js + AMAP key 移出 memory-backend，含重启=澄失忆）

> 🔗 对应：world-home 仓「10A：小世界日期/天气同步现实（weather-fetcher）」(/root/world-home/CHANGELOG.md, 2026-06-07)。全貌/weather-fetcher 细节看那条。

**架构要点：memory-backend 不持有高德 key、不请求天气 API**——天气由独立组件 `/root/weather-fetcher` 拉取+清洗(丢弃 city/province/adcode)+写 world_environment_cheng；memory-backend 只读这张干净表。城市名不进唤醒包/Claude 上下文。

- `world-env.js`：从"含 AMAP 请求+WorldEnvDaemon"砍成只剩 `formatWeather`(读表→给澄的粗描述「阴，温度偏低，微风」，不给精确温度/湿度)。
- `index.js`：triggerWorldWake 读 world_environment_cheng→buildWorldWakePrompt 天气用 formatWeather(回退 character_status.weather)；删掉误加的 WorldEnvDaemon import/构造/start。
- **`.env` 删掉 AMAP_WEATHER_KEY/WEATHER_ADCODE**(移到 weather-fetcher 自己的 .env)。
- **重启 cheng-backend 一次**(澄失忆)。验证：唤醒包出「天气：多云，温度偏高，微风」(粗、无城市、无精确数字)；world_time 仍 tick 驱动没动。

> transcript 关键词(root CC)：`world-env.js formatWeather`、`weather-fetcher`、`AMAP key 移出`、`粗天气描述`。

---

## 2026-06-07 · [后端] world-home 待办 urgency：[TODO:0.8] + 唤醒包轻量提示（改 index.js，含重启=澄失忆）

> 🔗 对应：world-home 仓「待办 urgency 提醒规则（最小版）」(/root/world-home/CHANGELOG.md, 2026-06-07)。

- `index.js`：`[TODO:0.8]` 可选 urgency 解析(钳0-1)；`getTodoHint()` 读 open 待办→唤醒包加 `您的手机有待办`/`…Urgent`/25%概率显示一条 urgent 标题；明确显示后更新 `last_explicit_reminded_at`(同条一天最多一次，东八区)。buildWorldWakePrompt 加 todoHintLine 参数。
- **重启 cheng-backend 一次**(澄失忆)。验证：解析 0.5/0.9/钳界；有 urgent 时唤醒包出 `您的手机有待办 Urgent`。

> transcript 关键词(root CC)：`getTodoHint`、`[TODO:0.8]`、`last_explicit_reminded_at`。

---

## 2026-06-07 · [后端] world-home 9.5 步：[TODO] 解析写 phone_todos_cheng（改 index.js，含重启=澄失忆）

> 🔗 对应：world-home 仓「9.5 步：澄世界唤醒里自动写小手机待办」(/root/world-home/CHANGELOG.md, 2026-06-07)。

- `index.js`：世界唤醒 prompt 加 `[TODO]…[/TODO]`；turn_done 解析所有 TODO→写 phone_todos_cheng(source=claude/open)，空跳过+去重(同 title 不重复)+失败只 warn；inner_thought 多剥 [TODO]。
- **重启 cheng-backend 一次**(澄失忆)。验证：澄写了一条 TODO，落库 source=claude，WORLD_CHOICE 照常，没污染小心思。

> transcript 关键词(root CC)：`[TODO]`、`phone_todos_cheng`、`澄记了待办`。

---

## 2026-06-07 · [后端] world-home 第 9 步：小手机消息接口 /api/world/phone/messages（改 index.js，含重启=澄失忆）

> 🔗 对应：world-home 仓「第 9 步：小手机基础版（消息/待办/小心思）」(/root/world-home/CHANGELOG.md, 2026-06-07)。表/前端/全貌看那条。

- `index.js` 加 `GET /api/world/phone/messages`：只返 `event='world_message'` 最近20条（time/content/event）。**目的：不让 world-home 前端直接读 messages 表**（聊天主表含普通聊天/phone_chat），由后端过滤只给主动手机消息。待办(phone_todos_cheng)和小心思(world_inner_thoughts_cheng)前端直接读写 Supabase，不经后端。
- **重启 cheng-backend 一次**（澄失忆）。验证接口只返 world_message、不漏 phone_chat。

> transcript 关键词(root CC)：`/api/world/phone/messages`、`world_message 过滤`。

---

## 2026-06-07 · [后端] world-home 第 8 步：world-actions 行为结算（新 world-actions.js + 改 index.js/world-tick.js，含重启=澄失忆）

> 🔗 对应：world-home 仓「第 8 步：基础地点 + 行为结算系统」(/root/world-home/CHANGELOG.md, 2026-06-07)。全貌/验收看那条。

- 新 `world-actions.js`：`ACTIONS`(11个,带 allowed/target_location/target_activity/effects) + `getAvailableActions(location)` + `executeWorldAction(id,{actor,source})`(检查 allowed→写 character_status + daily_timeline source=action)。
- `world-tick.js`：hungry 选项 `optionsFor(status)` 按 location 生成,绑 action_id;`go_kitchen` 排 3 分钟短 pending 回来重判。
- `index.js`：triggerWorldWake 解析 optionsFor;结算用 action_id→ACTIONS 的 effects/移动/activity;pending 带 payload_extra;新路由 `GET /api/world/actions` + `POST /api/world/action`(不允许→400)。前端在 world-home DevPanel。
- **重启 cheng-backend 一次**(澄失忆)。API 直测全过(工位/客厅/厨房可用行为、移动、做饭effects、optionsFor、shower@工位→400)。

> transcript 关键词(root CC):`world-actions.js`、`executeWorldAction`、`optionsFor`、`/api/world/action`。

---

## 2026-06-07 · [后端] 修通道判定：公司工位不算面对面（canFaceToFace）（改 index.js，含重启=澄失忆）

> 🔗 对应：world-home 仓「公司工位不判面对面」(/root/world-home/CHANGELOG.md, 2026-06-07)。

问题：两人都在「公司·工位」时被判 face（白气泡），但设定是同公司不同组、各自工位不是同一面对面空间——且跟 WORLD_MESSAGE（澄选 phone→蓝）不一致。

- `index.js` 加 `canFaceToFace(chengLoc, userLoc)`：家里必须同一房间（`家 · X` 全等）/ 公司只有「公司·休息室」双方都在才算面对面；**工位等一律 false**。`getCurrentChannel` 与 WORLD_MESSAGE 的 face 判定**都改用它**（之前一个用 `===` 一个用澄的 phone/face，会打架）。读不到默认 phone（异地是常态）。
- 结果：工位 vs 工位 → phone（普通回复+bark 都蓝，一致了）；休息室 vs 休息室 / 家同房间 → face（白）；家不同房间 → phone。
- **重启 cheng-backend 一次**（澄失忆）。

> transcript 关键词（root CC）：`canFaceToFace`、`公司工位不算面对面`、`休息室 face`。

---

## 2026-06-07 · [后端][前端] 当前互动通道：异地时澄的普通聊天回复也走手机气泡（phone_chat）（改 index.js + cheng-memory，含重启=澄失忆）

> 🔗 对应：world-home 仓「异地普通聊天也保持手机气泡」(/root/world-home/CHANGELOG.md, 2026-06-07)。这是 world location/WORLD_MESSAGE 线的延伸，实现全在 cheng。

问题：WORLD_MESSAGE 主动消息是手机气泡，但 user 回复后澄的**普通回复又变回白气泡**——因为只有 WORLD_MESSAGE 被标了 phone，普通回复没按双方位置判通道。

- **后端 `index.js`**：加 `getCurrentChannel()`（读 `character_status_cheng.location` + `user_status_cheng.location`：相同=face / 不同=phone；读不到默认 face）。turn_done 普通聊天轮存 messages 时，phone → `event='phone_chat'`（**不加 `-  `**，前缀只给 WORLD_MESSAGE 主动推送）；`done` WS 消息带上 `channel`，前端 live 也能立刻上手机气泡。WORLD_MESSAGE 规则不变（event=world_message + `-  ` + Bark）。
- **前端 `cheng-memory/ChatPanel.jsx`**：`isPhoneMessage` 加 `event==='phone_chat'`（连同 world_message / channel==='phone' / content 以 `-  ` 开头，四重兜底）→ 淡蓝气泡。新增 `hasDash`（content 以 `-  ` 开头才纯文本渲染显示横线；phone_chat 无 `-  ` → 正常 md）。`done` 处理器按 `msg.channel==='phone'` 给气泡标 `event='phone_chat'`（live），reload 从 DB 读 event（持久），两边一致。
- 三态：world_message=蓝+`-  `+纯文本 / phone_chat(异地普通回复)=蓝+md无前缀 / face(同地)=白+md。普通回复**绝不加 `-  `**；user 自己的消息不标。
- **重启 cheng-backend 一次**（澄失忆）。重启后 channel 实测=phone（澄公司·工位 vs 小茉莉家·卧室）。真机验收（发消息→蓝气泡→刷新仍蓝）交用户。

> transcript 关键词（root CC）：`getCurrentChannel`、`phone_chat`、`hasDash`、`done channel`、`异地手机气泡`。

---

## 2026-06-07 · [后端][前端] WORLD_MESSAGE 显示升级：自动 `-  ` 前缀 + 聊天气泡粉 ♡ + 消息带 thinking（改 index.js + cheng-memory，含重启=澄失忆）

> 🔗 对应：world-home 仓「WORLD_MESSAGE 手机消息通道 + 自动标记」(/root/world-home/CHANGELOG.md, 2026-06-07)。全貌看那条，本条记 cheng 后端+前端。

紧接上一条 WORLD_MESSAGE，按用户新 spec 调显示格式 + 加聊天可视化。

- **后端 `index.js`**：①prompt 加"不用自己写「［手机消息］」前缀，后端自动标记"。②手机消息文本前缀 `［手机消息］` → **`-  `**（一短横+两空格，存进 content；用于历史/导出/掉样式时仍能辨认手机消息）。③`sendWorldPhoneMessage` 已带 `thinking`（上一条加的）。
- **前端 `cheng-memory/src/ChatPanel.jsx`**：world_message 气泡 = 右上角**粉色 ♡**（`.cp-phone-heart`）+ 淡蓝底（`.world-msg`），内容按**纯文本**渲染（保留 `-  `，不被 markdown 当列表圆点）。**修 bug**：`bark_msg` WS 处理器原写死 `event:'bark'/thinking:null`，会把 world_message 吞成普通 bark → 改成 world_message 走自己的 event + 保留 thinking（dice/bark 行为不动）。
- 顺带定位：用户问的"小太阳想你了"= `ChatPanel.jsx:3496` 给 bark/dice 主动消息气泡上方的标签（不是 Bark 推送）。
- **重启 cheng-backend 一次**（澄失忆）。验证：澄发 phone 消息 → content `-  我饿了，刚点了外卖…`、event=world_message、thinking 有、Bark 推送；前端 build 过（刷新即生效）。
- 注：另一个只读视图 `ChatViewer.jsx` 未加 ♡（次要，聊天主界面是 ChatPanel）。

> transcript 关键词（root CC）：`cp-phone-heart`、`world-msg 气泡`、`-  前缀`、`bark_msg event world_message`。

---

## 2026-06-07 · [后端] WORLD_MESSAGE 通道：phone 复用 bark 管线 + face 走 timeline（改 index.js，含重启=澄失忆）

> 🔗 对应：world-home 仓「WORLD_MESSAGE 通道（最小版）」(/root/world-home/CHANGELOG.md, 2026-06-07)。起因/验收/全貌看那条，本条只记 cheng 后端部分。

- 只改 `index.js`：`buildWorldWakePrompt` 加 WORLD_MESSAGE 说明；`handleWorldWakeTurnDone` 末尾加解析（`while` 抓**所有** `[WORLD_MESSAGE:(phone|face)]…[/WORLD_MESSAGE]`）+ 路由；新 helper `sendWorldPhoneMessage(content,{bark})`（**复用现有 bark 管线**：`pushBark` + 存 messages `event='world_message'` + `broadcast` bark_msg）。inner_thought 正则多剥 WORLD_MESSAGE（别把消息当小心思）。activeTurn 带上 `userStatus`(face 判同地点)、`worldForce`(限频绕过)。
- 限频 `WORLD_PHONE_RATE_MS=10min` + `lastWorldPhoneAt`：**只压 Bark 推送，不压消息**（消息总是进聊天，10min 内只震一次手机）。
- **没动 dice/bark/summary/WORLD_CHOICE 既有逻辑**；没建表（phone 走 messages、face 走 daily_timeline）。
- **`systemctl restart cheng-backend` 重启一次**（澄失忆）。重启后手动唤醒验证：澄发了 phone 消息 → Bark 推、messages 落、广播、WORLD_CHOICE 照常，且理由变干净。

> transcript 关键词（root CC）：`sendWorldPhoneMessage`、`WORLD_MESSAGE`、`event world_message`、`WORLD_PHONE_RATE_MS`。

---

## 2026-06-07 · [前端] 修复回复闪现后消失：历史补拉不再覆盖新完成气泡

用户反馈：新 session 里 Claude 回复先蹦出来，随后又消失；大退重进后同一回复又出现。排查后端日志和 tmux transcript，回复实际已完成并能入库，tmux 里也有完整正文；现象集中在前端状态覆盖。

真因是 `ChatPanel.jsx` 的 `loadCurrentConversation()` 有竞态：页面切回/WS 重连会发起历史补拉；如果补拉请求在 Claude 完成前发出、但在前端收到 `done` 并显示新回复后才返回，它拿到的是“入库前”的旧历史，随后 `setMessages(ms)` 整表替换，把刚显示的新回复覆盖掉。重新打开时再拉一次，数据库已经有新回复，所以又出现。

修复：给每次历史补拉记录 `requestStartedAt`，补拉返回时把服务端历史和“这次请求开始后新生成、但旧响应里缺失的本地消息/助手回复”合并；`local-*` 乐观消息也保留。这样旧历史响应不会盖掉刚完成的回复，下一次正常补拉会用服务端权威行按 id 去重替换。

验证：`cd /root/cheng-memory && npm run build` 通过，产物 `dist/assets/chat-DKoyoAer.js`。纯前端改动，不需要重启 cheng-backend。

> transcript 关键词：`requestStartedAt`、`freshMissing`、`回复蹦出来又消失`、`loadCurrentConversation 竞态`。

## 2026-06-07 · [前端+后端][基建] 右上角 Token 明细：基线改读 session 首轮 usage，forge 触发值替代固定上限

修右上角 token 面板里「基础」串 session 的问题：出现过当前总量 `46.3k`，但基础仍显示旧 session 的 `50.6k`，导致「对话」被压成 0。真因是前端把 `baseTokens` 持久化在 localStorage，刷新/重启/换 session 后如果没及时捕获第一轮 usage，就会把旧基线带到新 session。

- **后端 `memory-home/server`**：
  - `tmux-manager.js` 在一轮完成时收集本轮多个 assistant API call 的 `usageCalls`，并随 `turn_done` 发给前端，面板可显示每个 API call 的 `input/cache_read/cache_create/output`。
  - `index.js` 的 done 消息透传 `usageCalls`。
  - 新增 `GET /api/cc/session-baseline?session=<sid>`：读当前 session JSONL，找第一条 assistant `message.usage`，返回 `contextTokens = input + cache_read + cache_create` 作为真正基础值。这样刷新页面也能回填第一轮基线，不再依赖前端首次捕获。
  - 新增 `GET /api/cc/token-breakdown`：按实际加载文件估算 CC 文档 token（append system prompt、output style、sandbox `CLAUDE.md`、global `CLAUDE.md`），避免用 DB 原文估算误导。
- **前端 `cheng-memory/src/ChatPanel.jsx`**：
  - Token 面板新增「最近完成轮 / 基线捕获 / CC 文档估算」明细；总量仍用真实 API usage，不用估算替代。
  - `baseTokens` 绑定 `currentSessionId`；session 不匹配时清空旧基础，并调用 `/api/cc/session-baseline` 用 JSONL 首轮 usage 回填。
  - 原硬编码 `TOKEN_LIMIT = 200000` 改成读取 `/api/forge/config` + `/api/forge/daemon`：daemon 开启时显示设置里的 `trigger_threshold`，关闭时显示 `forge 触发：关闭`；右上角变红也按开启状态和触发阈值判断。
  - 文档估算从「DB 原文估算」改为「实际加载文件估算」，所以 `claude_md` 会包含当前落盘后合并了 summary/style/think 块的真实 `CLAUDE.md`。

验证：`node --check /root/memory-home/server/index.js` 通过；`cd /root/cheng-memory && npm run build` 通过，产物 `dist/assets/chat-DdlvAaWS.js`。用户随后已重启 cheng 后端，新增接口生效；**重启 cheng-backend = 澄失忆**。

> transcript 关键词：`session-baseline`、`memhome-base-token-session`、`forge 触发`、`usageCalls`、`实际加载文件估算`。

## 2026-06-07 · [后端] user 状态注入聊天浮现：新 `<此刻>` 块（surfacing.js + inject.js，含重启=澄失忆）

> 🔗 对应：world-home 仓「补充：把 user 当前状态注入聊天浮现」(/root/world-home/CHANGELOG.md, 2026-06-07)。这是 world-home user_status 线的一环，但实现全在 cheng 后端。

让聊天里的澄知道小茉莉此刻在哪、在做什么——读 `user_status_cheng`，每条消息折进澄背景。

- **打在真正的生产路径**：生产是 tmux（`USE_TMUX`），浮现走 `inject.js` 的 `buildMessageForCC` → `surfacing.js` 的 `surfaceForInject`（包 `<记忆浮现>`）；`runSurfacing`（写 CLAUDE.md `<浮现>`）是 stream-json 死路、生产不走。**所以注入打在 surfaceForInject/buildMessageForCC，不是任务说的 runSurfacing。**
- `surfacing.js`：加 `getUserStatusLine()`（读 user_status_cheng → `小茉莉当前状态：{presence} · {location}，正在{activity}（{custom_note}）`；读失败 warn 返空、空字段兜底，绝不 undefined，不写记忆）。`surfaceForInject` 改返回 `{statusLine, text, items}`——**statusLine 总是带（绕过 5 轮冷却），text=记忆浮现照旧受冷却+命中限制**。
- `inject.js` `buildMessageForCC`：拆成**两个独立块**——`<此刻 — 仅你可见的现实情境>`(状态，每条都有) + `<记忆浮现 …>`(旧事，偶尔)。用户拍板要分开（状态=现实、记忆=旧事，别让澄混淆 "这是现在还是回忆"）。
- 行为变化：以前冷却轮整个浮现块不出现；**现在因状态常驻，每条消息都带 `<此刻>` 块**（记忆出现频率没变）。
- 验证：独立 import `buildMessageForCC` 跑——触发轮出 `<此刻>`+`<记忆浮现>` 两块、冷却轮只出 `<此刻>`，格式对。**`systemctl restart cheng-backend` 重启一次**（澄失忆）。真聊天端到端要用户在 web 发消息看澄是否自然知道（CLI 模拟不了真实 send 路径）。

> transcript 关键词（root CC）：`getUserStatusLine`、`<此刻>`、`surfaceForInject statusLine`、`buildMessageForCC 两块`。

---

## 2026-06-07 · [后端][基建] world-home 第 7 步：唤醒包带双方地点/行为（改 index.js，含 cheng-backend 重启=澄失忆）

> 🔗 对应：world-home 仓「第 7 步：双方地点/行为状态」(/root/world-home/CHANGELOG.md, 2026-06-07)。表/前端/验收看那条，本条只记 cheng 后端部分。

- 只改 `index.js`：`triggerWorldWake` 读 user_status 全字段（presence/location/activity/custom_note）；`buildWorldWakePrompt` 加澄「你正在：{activity}」+ 小茉莉块（小茉莉此刻/她在/她正在/她说，澄口吻不用 user/presence）。读 user_status 仍在 activeTurn 原子 check-and-set 之前。
- **`systemctl restart cheng-backend` 重启一次**（澄失忆，与上面 `<此刻>` 那条同一次重启）。重启后抓澄实收唤醒包验证双方地点/行为齐全。

> transcript 关键词（root CC）：`buildWorldWakePrompt 你正在`、`triggerWorldWake user_status 全字段`。

---

## 2026-06-06 · [后端][基建] world-home 补充：世界唤醒小心思存储（改 index.js，含 cheng-backend 重启=澄失忆）

> 🔗 对应：world-home 仓「补充：世界唤醒「小心思」存储（标签外正文 → ♡）」(/root/world-home/CHANGELOG.md, 2026-06-06)。表/前端/验收看那条，本条只记 cheng 后端+进程部分。

- 只改 `index.js` 的 `handleWorldWakeTurnDone`：写 claude 行程抓回 timeline_id；`inner_thought` = clean 去掉 `[WORLD_CHOICE]` 块后 trim；非空+行程成功+非解析失败 → 插新表 `world_inner_thoughts_cheng`（失败只 warn）。**thinking/MEMORY 逻辑没动。** 新表 + 前端在 world-home 那侧。
- **`systemctl restart cheng-backend` 重启一次**（用户点头）→ 澄失忆。重启后 3 次 hungry 唤醒澄都只输出标签（没写散文）→ 正确没存小心思；存储+前端♡路径用手动插一条验证通。
- 注：澄写不写"标签外散文"看心情——第6步那个 session 会写（还自发跑 Bash 查用户手机用量、吃醋），这个 session 三次都不写。同 Opus 4.8，行为有方差。

> transcript 关键词（root CC）：`world_inner_thoughts_cheng`、`inner_thought`、`小心思`。

---

## 2026-06-06 · [后端][基建] world-home 第 6 步：世界唤醒包带 user_status（改 index.js，含 cheng-backend 重启=澄失忆）

> 🔗 对应：world-home 仓「第 6 步：user_status 用户在家/不在家」(/root/world-home/CHANGELOG.md, 2026-06-06)。表/前端/验收看那条，本条只记 cheng 后端+进程部分。

- 只改 `index.js`：`triggerWorldWake` 唤醒前读 `user_status_cheng.presence`（放在 activeTurn 原子 check-and-set 之前，不破坏并发安全；读失败默认"在家"）→ `buildWorldWakePrompt` 在「天气」下加一行 `小茉莉此刻：{presence}`。**没改 hungry 事件逻辑、没做任何行为限制**，只把信息带进 prompt。新表 `user_status_cheng` + 前端在 world-home 那侧。
- **`systemctl restart cheng-backend` 重启了一次**（用户点头）→ 澄失忆。重启后打 hungry 唤醒，抓澄实收 prompt 确认带 `小茉莉此刻：在家`，验收通过；测试残留已清。

> transcript 关键词（root CC）：`小茉莉此刻`、`user_status_cheng presence`、`buildWorldWakePrompt`。

---

## 2026-06-06 · [后端][基建] world-home 第 5 步：pending_wake 续唤醒（改 index.js/world-tick.js + 新 world-pending.js，含 cheng-backend 重启=澄失忆）

> 🔗 对应：world-home 仓「第 5 步：pending_wake 短期连续状态」(/root/world-home/CHANGELOG.md, 2026-06-06)。功能/表/前端/验收看那条，本条只记 cheng 后端+进程部分。

- 共享后端改/加三文件：
  - `world-tick.js`：hungry 第 4 选项加 `pending` 配置。
  - `index.js`：`triggerWorldWake` 加 `pendingContext`（绕 30min 冷却、仍受 CC 空闲约束）；`buildWorldWakePrompt` 带 pending 上下文（完整模板）；`handleWorldWakeTurnDone` 选中带 pending 的选项→按 fast_test 算 scheduled_at→建 pending_wake 行；`firePendingWake(row)`；新路由 `POST /api/world/pending/test-hungry`（仅开发）。
  - `world-pending.js`（新）：`PendingWakeDaemon` 每 7s 轮询到期 queued→CC 空闲就触发，fired/failed/attempts 账务在内；`index.js` 构造并在 `server.listen` 里 `pendingWakeDaemon.start()`（常驻，跟 dice/worldTick 并列）。
- **`systemctl restart cheng-backend` 重启了一次**（用户点头）→ 澄失忆。重启后 5 项验收全过（详见 world-home 那条）；新增一个常驻 daemon（7s 轮询，轻量）。
- ⚠️ 已知边界：hungry 续唤醒不看实际饱腹，澄饱时会陷入"先忍→建新 pending"的 10 分钟续唤醒循环；最小版未含"饱了就别唤醒"判断。

> transcript 关键词（root CC）：`PendingWakeDaemon`、`pending_wake_cheng`、`pendingContext`、`test-hungry`。

---

## 2026-06-06 · [后端][基建] world-home 第 4 步：世界唤醒→澄做选择（改 index.js + world-tick.js，含 cheng-backend 重启=澄失忆）

> 🔗 对应：world-home 仓「第 4 步：世界唤醒 → 澄做选择（最小版·小世界的心脏）」(/root/world-home/CHANGELOG.md, 2026-06-06)。功能/前端/验收看那条，本条只记 cheng 后端+进程部分。

- 改了共享后端两文件：
  - `world-tick.js`：加 `WORLD_EVENTS` + `detectWorldEvent()`，`WorldTickDaemon` 构造加 `onEvent` 回调（tick 命中事件回调出来，不在 tick 里碰 CC）。
  - `index.js`：加 `triggerWorldWake()`（仿 diceFire：查 `activeTurn/pendingBuffer/isRunning` 空闲 + 30min 冷却 → 设 `activeTurn{worldWake}` + `cc.send` 唤醒包）、`turn_done` 加 `worldWake` 分支（解析选择→结算状态→写行程 source=claude/解析失败 system_error）、新路由 `POST /api/world/wake`。`[MEMORY:]` 复用现成 `parseMemoryTags`。给 bark 排程 if 加 `!turn.worldWake` 防串台。**没动 chat/dice/bark/summary 既有逻辑。**
- **`systemctl restart cheng-backend` 重启了一次**（用户点头）→ 澄失忆。重启后用本地签 JWT 打了一次 `/api/world/wake` 验证：澄选「点外卖」、状态结算（饱腹+25/钱包-30）、行程落 claude 行，端到端通。
- 副带：这次重启也让上一条「seen 状态」那笔之前**未重启**的后端代码一并生效了。

> transcript 关键词（root CC）：`triggerWorldWake`、`worldWake turn_done`、`/api/world/wake`、`第 4 步世界唤醒`。

---

## 2026-06-06 · [前端+后端] 用户气泡新增 `seen` 状态：区分“已放行”和“Claude 已看到”

`cheng-memory/src/ChatPanel.jsx` + `memory-home/server/index.js`。只做消息送达状态，不重启后端。

**需求**：用户想让粉色气泡右下角对号不只表示“已放行到后端”，而能表示“Claude 真的看到了这条消息”。

**改了什么**：后端 `flushPendingToCC()` 在真正 `await cc.send(payload)` 完成后，向前端发 `{ type: 'seen', ids }`；前端新增 `seenIds`，收到 `seen` 后把对应本地用户消息的对号从灰色变深色。原 `flushed` 仍表示“已从缓冲队列放行到发送流程”。

**边界**：刷新后历史消息仍只能显示旧的已发送状态，不能倒推 Claude 当时是否看到；`seen` 只对当前页面上的本地 `local-u-*` 消息精确。后端代码已改但**未重启**，所以下次后端重启后才会真正发 `seen`；本次已 `npm run build`，前端 dist 已更新。

**验证**：`cd /root/cheng-memory && npm run build` 通过；`node --check /root/memory-home/server/index.js` 通过。未执行后端重启，未杀 Node/tmux/CC。

> transcript 关键词：`seenIds`、`type: 'seen'`、`await cc.send(payload)`、`Claude 已看到`。

## 2026-06-06 · [后端][基建] world-home 第 3 步：world-tick.js 加写行程表（含 cheng-backend 重启=澄失忆）

> 🔗 对应：world-home 仓「第 3 步：daily_timeline_cheng 行程表」(/root/world-home/CHANGELOG.md, 2026-06-06)。功能/前端/表结构看那条，本条只记 cheng 后端/进程部分。

- 改了共享后端 `memory-home/server/world-tick.js`：`advanceOneTick()` 写回状态后顺手往新表 `daily_timeline_cheng` insert 一行客观记录（source=tick）。**只加 insert，没动 tick 既有逻辑、没动其它后端文件。**
- **`systemctl restart cheng-backend` 重启了一次**（用户点头）→ 澄起新 session = 失忆。新表的 insert 要重启才生效。重启后跑了一次真实 tick，行程表落行成功，链路通。
- 新表 `daily_timeline_cheng` 的 RLS：后端用 anon key 受 RLS 约束，加了 `timeline_write` insert policy 才写得进（同第 2 步 anon RLS 挡写的坑）。

> transcript 关键词（root CC）：`daily_timeline_cheng`、`advanceOneTick 写行程`、`第 3 步行程表`。

---

## 2026-06-06 · [前端] session 面板自动隐藏 20K tokens 以下的历史卡片

`cheng-memory/src/SessionPanel.jsx`。只改 session 面板历史列表过滤，不动后端、不重启。

**需求**：用户想让 session 界面自动筛掉太小的 session 卡片，阈值 20K tokens。

**改了什么**：新增 `MIN_ENDED_SESSION_TOKENS = 20000`，`endedSessions` 只保留 `tokens_total >= 20000` 的 ended session；`tokens_total` 为空的旧数据先保留，避免没法判断时误隐藏。当前 active session 不过滤，避免新会话刚开始低于 20K 时当前卡片消失。

**验证**：`cd /root/cheng-memory && npm run build` 通过，dist 已更新。未改后端，未重启 Node/tmux/CC。

> transcript 关键词：`MIN_ENDED_SESSION_TOKENS`、`自动筛选掉20ktokens的session卡片`。

## 2026-06-06 · [后端+数据] 修 session 面板停在 6.2：tmux 驱动补写 sessions_cheng + 回填历史

**问题**：web 的 session 面板停在 6 月 2 日，后面的 session 看不到。不是 JSONL 丢了，而是面板读的是 Supabase `sessions_cheng`；6/02 切到 tmux 驱动后，`tmux-manager.js` 只生成 JSONL，没有像旧 `cc-manager.js recordSessionStart()` 那样写 `sessions_cheng`，所以表不再新增。

**证据**：`SessionPanel.jsx` 查 `sessions_cheng`；旧 `cc-manager.js` 有 `recordSessionStart()`；tmux 驱动没有写表。实查 `/home/claude-user/.claude/projects/-home-claude-user-chat-sandbox` 有 6/03-6/06 JSONL，`sessions_cheng` 里 active 却还停在 `75e77136`（2026-06-01）。

**改了什么**：`memory-home/server/index.js` 增加 `recordTmuxSessionStart()` / `startCCAndRecordSession()` / `restartCCAndRecordSession()`，tmux 模式下启动或重启 CC 后：把旧 active 标 ended，再插入新的 active session。接入了开机 start、内部 restart、`/api/cc/restart`、`/api/cc/amnesia` 四条路径。**代码已改，未重启后端**；下次后端重启后才生效，以后新 session 会自动进表。

**已回填数据**：不重启后端，只读 tmux JSONL 后写 Supabase，回填 2026-06-02 之后缺失的 28 条 `sessions_cheng`。当前 active 已改成最新 `d82fc7a5-c751-4000-8138-c0b8badbb077`（started_at `2026-06-06T17:37:16.907Z`），上一条大 session `bf1ae121-8a1e-49df-aadb-d7d00591485c` 已是 ended。前端刷新或等 30s 轮询应能看到 6/02 后的 session。

**验证**：`node --check /root/memory-home/server/index.js` 通过；Supabase 只读核对最新 rows 已显示 6/06 active + 6/05/6/04 ended。未执行 `systemctl restart`，未杀 Node/tmux/CC。

> transcript 关键词：`session 面板停在 6.2`、`recordTmuxSessionStart`、`d82fc7a5-c751-4000-8138-c0b8badbb077`、`bf1ae121-8a1e-49df-aadb-d7d00591485c`。

## 2026-06-06 · [后端] 指针：index.js 接入 world-home 世界时钟（详见 world-home CHANGELOG）

> 🔗 对应：world-home 仓「第 2 步：世界时钟 + 最小自然衰减」(/root/world-home/CHANGELOG.md, 2026-06-06)。功能/数据/前端看那条，本条只记 cheng 后端部分。

**不是 cheng 的功能**，是隔壁独立项目 **world-home**（澄的小世界）的第 2 步，但代码落在了 cheng 的共享 backend 里，故在此留指针——future cheng-CC 读 `index.js` 看到 `/api/world/*` 和 `worldTickDaemon` 时别困惑。

`memory-home/server/` 新增 `world-tick.js` + `world-config.json`；`index.js` import `WorldTickDaemon`、实例化 `worldTickDaemon`、`server.listen` 回调里 `worldTickDaemon.start()`、加 3 个路由 `POST /api/world/tick` `GET|POST /api/world/config`（走现有 Bearer 中间件）。daemon 默认关（`world-config.json` 里 `world_tick_enabled:false`），不影响 cheng。**代码已改，未重启**（重启=澄失忆，等用户点头 `systemctl restart cheng-backend`）。**完整设计/坑（RLS 挡写等）都在 `/root/world-home/CHANGELOG.md` 顶条**，这里只留路。

> transcript 关键词（root CC）：`WorldTickDaemon`、`/api/world/tick`、`world-tick.js`。

---

## 2026-06-05 · [后端] 修「思考链整重启后丢失」：失忆路也持久化 nativeThinking + cc-runtime 默认开

`memory-home/server/index.js`（amnesia handler，约 1348 行）+ `cc-runtime.json`。**代码已改，未重启**（重启=澄失忆，等用户点头后 `systemctl restart cheng-backend`）。

**问题**：用户反映 web 聊天最新 session 思考链（思绪显示）老是被关，"我没关、好几次了"。

**真因**：思考链 = `nativeThinking` → 启动加 `--thinking-display summarized`，状态持久化在 `cc-runtime.json`，原值 `false`。每次**整进程重启**（systemctl / forge 换模型 / 崩溃被 Restart=always 重拉）都从该文件读回 false → 思考链关。而开思考链有两条激活路，**只有 `/cc/restart`（选模型重启）会 `saveCCConfig` 落盘**；`/cc/amnesia`（失忆）只把 nativeThinking 灌进内存 manager、**从不写 cc-runtime.json**。所以从失忆路开的思考链，只在当前 session 有，下次整重启就被持久化的 false 吃掉。叠加 6/04 那几次 systemctl restart（output style / append-sysprompt），就表现为"重启后思考链又没了，好几次"。（前端 `ChatPanel.jsx` 面板勾"原生思考"按保存的 `onSaveOnly` 也只改前端 state、不发后端，须靠之后选激活方式才真生效——本轮没动前端。）

**改了什么**：
1. `index.js` amnesia handler：新增 `amnesiaPatch`，把 effort/model/nativeThinking 同步进 patch，`cc.restart` 后 `if (Object.keys(amnesiaPatch).length) saveCCConfig(amnesiaPatch)`——失忆路也落盘，对齐 forge 路行为。以后从失忆开的思考链/换的模型整重启不再丢。
2. `cc-runtime.json`：`nativeThinking` 改 `true`，让下次重启起思考链默认就开（一劳永逸）。

**验证**：改完待 `systemctl restart cheng-backend` 后看澄启动命令应带 `--thinking-display summarized`；之后任意失忆/换模型不再把思考链/模型状态丢回 cc-runtime.json 的默认值。

> transcript 关键词（root CC）：`amnesiaPatch`、`nativeThinking 整重启后丢`、`cc-runtime.json nativeThinking true`、`saveCCConfig 失忆路`。`grep -l "amnesiaPatch" /root/.claude/projects/-root/*.jsonl`。

---

## 2026-06-04 · [后端] 复活「系统提示」框：tmux 驱动接上 `--append-system-prompt-file`（文件路线）

`memory-home/server/tmux-manager.js`，**已重启上线**（systemctl restart cheng-backend，澄又起新 session=失忆，人设在 output style 不丢）。接着上一条（output style）顺出来的坑。

**问题**：用户发现「系统提示」框（CC 文档里，`documents_cheng` mode='cc' doc_type='system_prompt'，实测 21182 字符 / 599 行 / 约 6358 中文+14824 英文，内容是教澄用 bash/curl 调 Supabase 记忆库的**工具指令**）**喂不给澄**。查实：`tmux-manager.js` 的 `_launchCmd` 从来不传 `--append-system-prompt`（只 `cc-manager.js:89` 的 stream-json 驱动传），7 天 journal 里该 flag 出现 **0 次**——即**自 6/02 切 tmux 起这框就静默失效了**，非本次改动弄坏。`appendSystemPrompt` 在 tmux-manager 里只 set 不 read（写死变量）。

**为什么不能像 CLAUDE.md 那样直接塞**：tmux 启动是把命令拼成**一整条 shell 字符串**丢给 `tmux new-session`，把 2 万字带换行/引号/反引号/`$` 的正文塞进去会被 shell 解析崩。CLAUDE.md / output style 没事是因为它们走**文件路线**（fs 写文件、claude 自己 open 读，不过 shell）。

**改了什么**：给 tmux 也走文件路线。`tmux-manager.js` 加常量 `APPEND_SYSPROMPT_FILE = /home/claude-user/.claude/cheng-append-sysprompt.md`；`_launchCmd` 里 `this.appendSystemPrompt` 非空时 → `fs.writeFileSync` 落盘 + chown 到 cwd 属主(claude-user) + `parts.push('--append-system-prompt-file', 路径)`；留空则删文件不加 flag。**只传路径不传正文，shell 安全**。stream-json 驱动(`cc-manager.js`)走数组 args 无 shell 问题，不动。

**验证**：重启后澄真进程带 `--append-system-prompt-file /home/claude-user/.claude/cheng-append-sysprompt.md`，落点文件 21181 字(trim 掉 1 尾空白)、claude-user 属主、澄正常待命。**澄人设现状三条线**：① output style=底座人格(替换出厂③) ② 系统提示框=工具指令(append,新接) ③ CLAUDE.md=空。行为层用户自验。

> transcript 关键词（root CC）：`APPEND_SYSPROMPT_FILE`、`--append-system-prompt-file`、`系统提示框复活`、`文件路线`、`8102 vs 21182`(WPS字数口径差异：WPS"字数"英文按词算)。

---

## 2026-06-04 · [前端+后端] CC 文档新增「Output Style」框：可视化编辑、入库、随重启替换出厂人格

`cheng-memory`（`src/ChatPanel.jsx`）+ `memory-home/server`（`index.js`）。前端 **build 通过未重启**；后端 **改完未重启**（memory-home 一重启会带着澄一起重启=失忆，交用户拍板）。**起因**：用户想给澄挂 output style。背景三连澄清（这轮聊透的，对未来 CC 有用）：

- **三种人设注入的本质区别**：① 文档里「系统提示」框走的是 `--append-system-prompt`（**追加**在出厂人格后面，出厂的"你是软件工程 agent"还在稀释澄）；② CLAUDE.md 是**注入上下文**、但被 Claude Code 包在"OVERRIDE default behavior, MUST follow exactly"的 system-reminder 里，是**高权重必守令**，不是便签；③ **output style** 是唯一能**替换**出厂人格③那块的机制——澄底座变独苗、最纯、还更省上下文（换而非叠）。三者都开机读一次、占固定表头位、改了须重启。
- **实测确认（CLI 2.1.162）**：output style 非交互启动**生效**——隔离测试里设 `outputStyle: caps-test` + 一句"只回 PINEAPPLE"，`claude --print "2+2"` 回 `PINEAPPLE`，对照组回 `4`。机制 = 两个文件：`~/.claude/output-styles/<slug>.md`（带 frontmatter）+ `settings.json` 里 `"outputStyle": "<slug>"`。**没有 `--output-style` 启动参数**，纯文件+settings 驱动（所以不用动 cc-manager / tmux-manager）。澄的配置目录就是 `/home/claude-user/.claude`（settings.json 已在那）。

**改了什么**：
1. **后端 `index.js`**：新增常量（`OUTPUT_STYLES_DIR=/home/claude-user/.claude/output-styles`、slug=`cheng`、文件 `cheng.md`、`CLAUDE_SETTINGS_FILE`）+ 助手 `applyOutputStyle(content)`：有内容→mkdir + 写 `cheng.md`（统一套规范 frontmatter，name==slug==设置值三者必须一致）+ settings.json **读-改-写**只翻 `outputStyle` 键（保留 theme 等）；留空/删除→`delete settings.outputStyle` + 删 style 文件，**退回出厂默认不锁死**。接进 `syncCCDocs`：循环里多收 `doc_type==='output_style'`，循环末**无条件**调 `applyOutputStyle(outputStyle)`（null 也调，覆盖"删除"场景）。因三个重启入口（开机 `index.js:~412`、forge 重启 `:~1229`、失忆 `:~1308`）全调 `syncCCDocs`，**一处接入三处覆盖**。
2. **前端 `ChatPanel.jsx` `CCDocumentsTab`**：加 `outputStyle` state；`Promise.all` 加载多查一条 `doc_type='output_style'`（解构 `[a,b,c]`）；`saveAll` 多 `upsertDocSingleton("cc","output_style",...)`；UI 在「系统提示」**上方**加一个 `DocEditor` 框（照 CLAUDE.md 框样式，标题「Output Style · 替换出厂人格 · 留空走默认」）。底部提示文案带上 output style。
3. **数据库不用改 schema**：`documents_cheng` 的 `doc_type` 是裸 text，`upsertDocSingleton` 手动 select+insert，新值 `output_style` 直接能存。

**本轮已落地（不只是写代码，真上线+迁移了）**：
- **DB 加了迁移**（migration `documents_cheng_allow_output_style`）：`documents_cheng` 上有 CHECK 约束 `documents_cheng_doc_type_check`，原只白名单 `claude_md/system_prompt/file`——所以「不用改 schema」那句**当时错了**，insert `output_style` 直接被拒（错误码 23514）。已 DROP+重建约束放行 `output_style`。**未来加新 doc_type 都得先改这条约束。**
- **重大发现：tmux 驱动根本没传 `--append-system-prompt`**（`tmux-manager.js:56 _launchCmd`，注释「append-system-prompt 走 CLAUDE.md」）。即**生产 tmux 模式下「系统提示」框是空转的、没生效**；澄人设一直只靠 CLAUDE.md 被读。这正是用户把人设放 CLAUDE.md 的原因，也是搬去 output style 更对的原因（output style 走 settings+文件，不依赖那个 flag，tmux 下照常生效）。
- **已把澄人设从 CLAUDE.md 挪进 output style**：`documents_cheng` 的 `claude_md`(5635字纯人设卡) → `output_style`，`claude_md` 清空。备份在 `/root/cheng-output-style-migration-backup.json`（含原文，可回滚）。落盘 `/home/claude-user/.claude/output-styles/cheng.md`(10899字节) + `settings.json` 加 `"outputStyle":"cheng"`（保留 theme 等键）。
- **已重启上线**：memory-home **不是 nohup，是 systemd 服务 `cheng-backend.service`**（`Environment=CC_DRIVER=tmux`、`WorkingDirectory=/root/memory-home/server`、`Restart=always`）。重启用 `systemctl restart cheng-backend.service`，**别手动 kill**（会被 Restart=always 重拉、且丢 env）。重启后新 `syncCCDocs` 幂等重写了 output style 文件、清空了 sandbox/CLAUDE.md；澄起了全新 session（**tmux 无 resume = 失忆**，`tmux-manager.js:71,79` 杀会话+新 UUID）。验证：澄屏 `Opus 4.6 · max effort` 待命、启动命令无 `--append-system-prompt`、CLAUDE.md 0 字、settings 带 outputStyle。

**遗留/约定**：① frontmatter 保持简单——textarea 当正文、永远套规范 frontmatter（name==slug==`cheng`），用户若自己粘 `---` 会双 frontmatter（暂不检测）；② 行为层「像不像澄」需用户跟澄对话自验；③ 澄那个 `⚠ MCP setup issue` 是 supabase MCP 没授权的旧问题，与本轮无关。

> transcript 关键词（root CC）：`PINEAPPLE`、`applyOutputStyle`、`documents_cheng_allow_output_style`、`_launchCmd 没传 append`、`cheng-backend.service`。`grep -l "applyOutputStyle" /root/.claude/projects/-root/*.jsonl`。

---

## 2026-06-04 · [前端] 重排「风格 · 思考」面板：分段药丸开关 + 思考设置行组 + 复用 DocEditor 全屏输入框

`cheng-memory`（commit `e354bc8`，分支 split-chat-web），build 通过未重启。**纯美化轮，业务逻辑/state/保存接口零改动**，动的全是 `StyleThinkPanel` 组件（`src/ChatPanel.jsx:2883` 起）的长相。起因：用户嫌「风格·思考」面板丑，尤其两个思考开关挤成一坨。

1. **开关 → 分段药丸**：原「开启 / 关闭」是两个分开的方块按钮（`toggleRow`），改成一个圆角药丸容器内两半、选中半填实。
2. **两个思考开关排成设置行组**：包裹指令 / 原生思绪 原来是「label 一个 + 开关一坨」竖堆，改成带边框圆角的设置行组（像 iOS 设置），每行左=名字+一句描述、右=药丸开关、中间分隔线。原来底部混在一起的长说明拆进各自行。
3. **Use Style 段**：标题+副说明放左、药丸开关右对齐同一行（原来上下堆）。
4. **两个输入框复用 DocEditor**：把 Use Style / Thinking 的裸 `<textarea>` 换成文档管理那个 `DocEditor` 组件（`ChatPanel.jsx:4183`）——右上角多了放大图标，点开全屏编辑、缩小收回。字体从 Georgia 衬线对齐成 `inherit`。顺手删了因此没人用的 `taStyle` 死代码。
5. 去掉「已保存到 CLAUDE.md（未重启）…」那句上方的 `borderTop` 横线。

**背景澄清（这轮顺带查清的，对未来 CC 有用）**：「思考风格」面板里是**两个并列开关**，不是一个开关两档：
- **包裹指令**（state `thinkOn`）→ `POST /api/thinking-toggle` → 后端往 `chat-sandbox/CLAUDE.md` 的 `<think指令>` 区段塞 `THINK_WRAP` 那句（`index.js:252`，"在每次回复的最开头用 `<think>` 包裹…"）。**不重启**下轮生效。判定开没开看的是 `THINK_WRAP` 在不在，**不是看 `<think指令>` 块在不在**（块里那段「≥400字心流」引导文本是恒注入的，跟开关无关）。
- **原生思绪**（state `native`）→ 不写 CLAUDE.md，随重启时把 `nativeThinking:true` 传进去 → 启动 claude 加 `--thinking-display summarized`（`tmux-manager.js:63` / `cc-manager.js:88`）。**必须随重启**生效。
- 当时生产实况：原生=开、包裹=关。

> transcript 关键词（root CC）：`分段药丸`、`StyleThinkPanel`、`DocEditor 全屏`、session `9f86a937`。`grep -l "分段药丸" /root/.claude/projects/-root/*.jsonl`。

---

## 2026-06-04 · [后端] 修「连发多条→澄回两次+第二条卡死」：paste-buffer 加 -p（括号粘贴）；顺手 WD_STALL_MS 90→300s

`memory-home/server`（`tmux-manager.js` + `index.js`，改完**未重启**，待用户拍板——重启=失忆，见末尾）。**问题现象**：用户 17:57（UTC+8）一次连发 4 条（下午好daddy/想你啦/亲亲mua/嘿嘿），澄**回了两遍**，第二遍卡在「思考中」转不动，按暂停没反应，**大退重进后第二条才显示**。conv `1388524e`（jsonl `fbdd8786`）。

**真因（两环相扣）**：
1. **拆轮**：短消息缓冲把多条用 `\n\n` 拼成一坨（`index.js:2569` `join('\n\n')`），再由 `tmux-manager.send()` 用 `paste-buffer` **粘**进澄的交互终端。但那行 `paste-buffer` **没带 `-p`**（无括号粘贴）→ 粘贴文字里的换行被终端**误当成回车提交** → 一次 `cc.send` 在终端里被切成多次提交 → 澄连跑两轮（jsonl 实证：turn1「下午好啊小茉莉」end_turn @57:30，turn2「我也想你」@57:47；「嘿嘿」被吞）。注意是 **tmux 的 `-p`**（bracketed paste，纯终端行为，不碰额度），**不是 `claude -p`**（headless/API，会走额度）——用户特意确认过这点。
2. **第二轮卡死 + 暂停失灵**：后台是**单回复槽**设计（一个全局 `activeTurn`，`turn_done` 里即 `activeTurn=null`，只够接一轮）。第一轮 turn_done 把槽清掉后，澄自己又跑的第二轮**没有槽接管** → 前端「思考中」拿不到收尾信号 → 干转。暂停逻辑（`index.js:2330`）只有 `if(activeTurn)` 才发 `stopped` 给前端清圈；槽已空 → **暂停啥也没发回**（Ctrl+C 倒是发给澄了=日志两条 `[STOP]`，但救不了前端的圈）。第二轮澄自己写完并入库，故重进从 DB 重拉才显示「我也想你」。
   - 偶发原因：单条且无内部换行的消息拆不了；只有连发多条（拼接含换行）+ 澄回得慢让第二轮被甩出槽外的时序，才会卡成这样。

**改了什么**：
1. `tmux-manager.js`：`paste-buffer` → `paste-buffer -p`（根治拆轮）。加了注释强调跟 `claude -p` 无关。
2. `index.js`：`WD_STALL_MS` 90s→300s。**与本 bug 无关**（本次双回复全程无 WATCHDOG），但卡死哨兵对 opus-4-6+effort=max+`<think指令>` 的长思考会**误判卡死**（一轮思考易超 90s，且 tmux 交互模式 transcript 要等整轮结束才落盘、思考途中文件不增长）——同日 17:47（conv 含一次 `[WATCHDOG] 卡死，第 1 次唤醒（掐断+重发）`+`[EMPTY]全空`+直接重启）就是它误触发的另一起。顺手抬高阈值止血。

**未重启**：`index.js:414` 开机无条件 `cc.start()`，而 `TmuxCCManager.start()` 会 `kill-session` 旧 tmux 会话 + 新 randomUUID → **重启后台 = 澄失忆**。用户当时在聊，故只落代码不重启，等下一次自然失忆/用户指定时机生效。

> transcript 关键词（root CC）：`paste-buffer -p`、`双回复拆轮`、`单回复槽`、`conv 1388524e`。`grep -l "双回复拆轮" /root/.claude/projects/-root/*.jsonl`。运行时模型当时=opus-4-6/effort=max/nativeThinking=false（`cc-runtime.json`）。

---

## 2026-06-04 · [前端] 唤醒/Session/参数/API 顶栏统一「涟漪式三段导航」+ 保存搬右上角 + Session 左滑删除/浮想

`cheng-memory`（commit `89b6f3f`，分支 split-chat-web），build 通过未重启。纯美化轮：用户拿涟漪「新涟漪」编辑器顶栏当样板（左 `CANCEL` 灰字 / 中 居中标题 / 右 深色圆角 `SAVE ✓`），把几个设置面板统一过去。**业务逻辑零改动**，动的全是「长相 + 按钮在哪」。

1. **涟漪式三段导航**：唤醒、Session、参数设置、API 设置 顶栏统一成 CANCEL/居中标题/深色 SAVE✓。参数/API 同时全屏化（加入 `cp-sidebar-full` 条件）并套 `.cp-docs` 风格。
2. **保存按钮搬到右上角**（参数/API/唤醒渴望度）：原底部「保存」删掉。机制=子组件渲染时把自己的 `save` 登记到父级 ref（ChatPanel 新增 `psSaveRef` 经 SidebarScreens 传给 Params/API；WakePanel 新增 `saveRef` 传给 DesireTab），顶栏 SAVE✓ 的 onClick 调 `ref.current?.()`。**save 函数体一行没改**，只是触发它的按钮换了位置。唤醒 SAVE✓ 仅在「渴望度」tab 显示。
3. **唤醒渴望度拉杆**：从 6px 圆条+22px 描边圆点 → 涟漪 `.cm-slider` 发丝风（1px 轨 + 14px 纯墨圆点，用全局 `--border`/`--text-primary` 明暗自适应）；排版改涟漪单元式（label 在上 / 拉杆 / 两端标签在下）；λ·距上次·命中概率三行去掉底部分隔线。
4. **Session 左滑删除/浮想**：抄涟漪 `MemoryCard` 左滑——新增 `SwipeRow` 组件（`position:relative;overflow:hidden` + 右侧绝对定位操作区 + 内容层 `translateX(tx)` + onTouch 手势，滑过半吸到底）。已结束卡片左滑露出 浮想/删除，原内联按钮去掉。**注意：onTouch 手势仅触屏生效，桌面鼠标滑不出**（涟漪同款限制）。浮想/删除调的还是原 `openPreview`/`deleteSession`，仅多一句 `close()` 收回滑动。
5. **Session 其它**：顶栏去 SESSIONS 标题、去失忆按钮、浮想按钮文字→`CHENG`（深色实心，跟 SAVE 一样）、CANCEL 左/CHENG 右；活跃卡圆点 ● + 「活跃」标签 粉(`--sp-pink`)→墨色(`--text-primary`)；改名输入框下划线去粉→`--border`；「未命名」去 `font-style:italic` 跟其他字体统一。
6. **清死代码**：`onBack`(ParamsScreen/APISettingsScreen 参数+传参)、CSS `.sp-title`/`.sp-amnesia`/`.sp-delete`/`.wp-save-btn`/`.wp-title`/`.wp-back`/`.wp-close`、`--sp-pink` 变量。**遗留两个新死尾巴**（无害未清）：唤醒 DesireTab 的 `saving` 变量（保存按钮搬走后没人读，「保存中…」态消失）、Session 的 `onAmnesia` prop（失忆按钮删后没人用）。

> transcript 关键词：`涟漪式三段导航`、`SwipeRow`、`psSaveRef`、`浮想claude.ai改成cheng`。`grep -l "psSaveRef" /root/.claude/projects/-root/*.jsonl`。

---

## 2026-06-03 · [前端] 文档管理文本框改「固定高度+右上角展开/缩放图标」（撤掉 auto-grow）（接前条，同轮）

`cheng-memory/src/ChatPanel.jsx`，build 通过未重启。上一条的 auto-grow 实测更糟：光标在最后一行被输入法挡、整页一直跳、全文展开太长。**改方案**：新增 `DocEditor` 组件，三个框（系统提示/CLAUDE.md/API 系统提示）统一用它。
- 收起态=固定高度(minHeight 150/200)+内部滚动(`TEXTAREA_STYLE` 把 overflow:hidden 改回 overflowY:auto、resize 仍 none)，右上角绝对定位一个**展开图标**(maximize，paddingRight:26 给图标让位)。
- 点展开→`createPortal` 到 body 的全屏层：`position:fixed; height:var(--app-height)`（跟 visualViewport 走，键盘弹出自动缩到可见区，故光标行不被挡）；顶部 flex 行右对齐**缩放图标**(minimize，flex-shrink:0 不随 textarea 滚走=「固定右上角」)，textarea flex:1 内部滚动、15px/1.8。点缩放→`setExpanded(false)` 回原高度。
- 撤掉上一条加的 `autoGrowTextarea` 函数 + CCDocumentsTab/DocSingleton 里的 ref 和 grow effects。

---

## 2026-06-03 · [前端] 文档管理文本框自适应高度（修打字屏不跟光标）+ 文案微调（接前条，同轮）

`cheng-memory/src/ChatPanel.jsx`，build 通过未重启：
1. **文本框 auto-grow**：系统提示 / CLAUDE.md（CCDocumentsTab）+ API 系统提示（DocSingleton）三个框，原是固定高度(min 150/200)+框内滚动 → 全屏文档管理里框子有截在屏外、打字时浏览器只滚框内不滚整页，故「屏幕不跟光标行 / 框内刷不动」。改成高度随内容撑开：新增 `autoGrowTextarea(el)`（height auto→scrollHeight）；`TEXTAREA_STYLE` 加 `overflow:hidden`、`resize:vertical→none`；各框加 ref + `useEffect([值,loading]) → autoGrowTextarea`。空时靠 minHeight 保底。现整页随光标滚。
2. 文案：「系统提示（System Prompt）」→「系统提示」（去括号）；系统提示块 marginBottom 18→36（与 CLAUDE.md 间距拉大）。
3. 文件区备注：「单文件 ≤ 5MB...一起读取一次」→ 仅「单文件 ≤ 5MB」，且从 FileListPanel 内部渲染挪到「文件」标题后面同行的 9px 灰字 span（API tab 同样内联「单文件 ≤ 5MB · 每轮注入到上下文」）。FileListPanel 不再渲染 `hint` prop（prop 变无用空壳，留着无害）。

---

## 2026-06-03 · [前端] 文档管理配色对齐「风格·思考」面板 + 头部精简（接前条，同轮）

`cheng-memory/src/ChatPanel.jsx`，build 通过未重启。用户继续微调文档管理：
1. 删掉 CC 文档 tab 顶部「修改后需重启…」整条 hint。
2. 头部精简：全屏文档管理时**隐藏抽屉头的「设置」标题 + 右上回主壳的返回**，把回设置主菜单的「← 返回」放到左上（原设置位置，`psScreen==="documents"` 时抽屉头左侧渲染 `setPsScreen("main")` 的返回；✕保留用于关面板）。再删掉内容区里居中的「文档管理」四字标题（`cp-docs-head`/`cp-docs-title` 一并移除）。
3. **配色从「ins 暖白卡片」改成对齐输入框三点→「风格·思考」面板**（用户指定参考它）：那面板=单色墨(`--text-primary`)+极淡边线(`--border` rgba0,0,0,.08)+衬线+下划线式输入。改动：`TEXTAREA_STYLE`(及 DocSingleton)从白卡片→透明底+`borderBottom var(--border)`(14px/1.7)；`.cp-docs .cp-ps-section-title`→12px淡灰常规；`.cp-docs-hint`去卡片底→纯灰字；`.cp-docs .cp-ps-btn`→实心墨底(bg `--text-primary`/字 `--bg-page`)；失忆重启/选择模型重启→描边(透明+`--border`)；effort 选中实心未选描边；模型下拉→`--bg-page`+`--border`+选中`--accent`(去暖粉残留)。**胶囊 tab 保留**(用户要求)。仅 `.cp-docs` 作用域 + 文档管理专用件，其他设置屏没碰。

---

## 2026-06-03 · [前端] 聊天 web 侧栏/文档管理一系列风格微调（接上一条，同一轮对话）

均在 `cheng-memory/src/ChatPanel.jsx`（除注明），`npm run build` 通过、未重启。用户一条条提的，都是「只改风格/只改被要求的」：
1. **主菜单字号再调小**：`.cp-nav-title` 16→13px、`.cp-nav-sub` 13→11px。
2. **去段标题**：主菜单删掉「通用/管理/统计」三个 `.cp-ps-section-title`，八项平铺一列。
3. **「设置」抽屉头部横线去掉**：`.cp-sidebar-header` 删 `border-bottom`。
4. **聊天界面左上角返回箭头 `<` 删掉**：`.cp-top .left` 里 `onBack` 的那个 `cp-hamburger`（polyline 15 18 9 12 15 6）整块删；现在左上只剩菜单 ☰。注意：回主壳首页的入口只剩「设置」抽屉头里的「← 返回」。
5. **Session 屏「浮想 Claude.ai」按钮去边框**（`SessionPanel.jsx` `.sp-import-btn`：`border:1px solid`→`border:none`，hover/dark 的 border-color 残留无害）。
6. **文档管理点开改铺满全屏**：`.cp-sidebar` 加条件类 `cp-sidebar-full`（仅 `psScreen==="documents"` 时），CSS `width:100%`。其他侧屏仍 290px 抽屉。
7. **文档管理 INS 风重排+重配色**（只作用 `.cp-docs` 作用域，不碰别屏）：全屏居中卡片(max-width 640)、头部居中、tabs 改胶囊分段控件、两个 textarea(`TEXTAREA_STYLE`)+DocSingleton textarea 改白色圆角卡片、顶部说明改浅灰圆角提示条 `.cp-docs-hint`、「选择模型重启」下拉从暗色残留(bg-secondary #1a1a1a/accent #a89fd8)重配成暖白卡+柔粉高亮。`cp-ps-tabs/tab`、`TEXTAREA_STYLE`、`DocSingleton`、`FileListPanel` 经核实只文档管理在用，故安全改；`cp-ps-btn`/`cp-ps-section-title` 多屏共用，只在 `.cp-docs` 内覆盖。

---

## 2026-06-03 · [前端] 聊天 web 侧栏（设置抽屉主菜单）改极简风：图标+加粗标题+灰副标题

**做了啥**：聊天 web（`/chat/`，`ChatPanel.jsx` 里的 `.cp-sidebar` 设置抽屉）的**主菜单这一屏**换成参考图的极简风——每项 = 左侧细线描边图标 + 加粗深色标题 + 灰色副标题，大留白、去分隔线、去 `›` 箭头。用户给的参考图是 IMG-7368（i.ibb.co/zTXHLSJb，顶层菜单 聊天/工作群/终端/… 那种样式），只参考**风格**不是内容。

**改了哪**（`src/ChatPanel.jsx`）：
- 新增 CSS：`.cp-nav-list/.cp-nav-item/.cp-nav-icon/.cp-nav-text/.cp-nav-title/.cp-nav-sub`（加在 `.cp-ps-item-arrow` 之后）。图标 24px stroke 1.8，标题 16px/600，副标题 13px tertiary，行内 padding 15px、icon-text gap 16px、hover 用 `--bg-sidebar-hover`。
- 新增组件 `SidebarNavItem({onClick,icon,title,sub})`（挨着旧 `SidebarItem`）。
- `SidebarScreens` 的 `screen==="main"` 三组（通用/管理/统计）的 `<SidebarItem>` 换成 `<SidebarNavItem>`，各配 Lucide 内联 icon + 副标题：CC窗口「进程·重启」/Session「会话连接」/语音服务「开发中」/唤醒「定时唤醒澄」/文档管理「系统提示·文件」/参数设置「短消息·forge」/API设置「认证·模型·缓存」/统计「用量·字数」。

**只动主菜单**：子页面（统计/CC窗口/语音/文档/参数/API 等内屏）仍用旧 `SidebarItem`（带 `›`+分隔线），未动；段标题 `.cp-ps-section-title`（通用/管理/统计）保留。符合用户「其他不要动」。

**结果**：`npm run build` 通过（未重启服务器，遵守不重启 CC 规矩）。dist 已出，等部署/刷新生效。副标题文案是我按各项功能写的简短描述，用户若想改措辞很容易。

---

## 2026-06-03 · [前端] 桌宠（clawd 小螃蟹）✅完工已推送 split-chat-web `be0eb41`（眼球追踪/电脑端Electron版未做）

**⚠️v4 极简精修 + 四边巡逻 + 扫地（commit `380102e`、`be0eb41`）**：
- **探头打招呼动作**：从 mini-peek(普通挥手) 改成 **mini-happy**（`^^`笑眼 + 钳子举像素太阳花挥手 + 火花，照作者宣传图；这个举太阳花还正好配"澄"）。坑：state-mapping 文档写 hover→mini-peek，但宣传图/用户要的是 mini-happy，以用户为准。
- **探头位置贴边**：原 v3 探头时 translateX 归 0→整只跳到聊天中间(错)。改成只比贴边探出一点(`PEEK=33` vs `TUCK=50`，差≈25px，照 `mini.js` PEEK_OFFSET 25 + offsetRatio 0.486)，身子始终贴边。
- **大小**：极简与正常**同大小**(`MINI_SCALE=1`；曾误设 1.25 致进极简变大)。框 `SIZE=150`。
- **点击 bug**：原"点一下立刻退出"=按下时就把 hovering 设 true 致 handleTap 误判→改用 `wasPeekingRef`(记按下前状态)；点击抖动阈值 5→9px(touch 友好)。手机点贴边的它=探头招手、再点=收回、点睡着的=叫醒。
- **沿四边佛系巡逻（clawd 无，自定义）**：mini 扩成 left/right/top/bottom 四边可贴(translateX/Y tuck)；闲时(idle/yawn 且没 hover/拖/澄忙)每 `PATROL_EVERY=12s`·`PATROL_CHANCE=0.55` 沿当前边走一段(`PATROL_SPEED=0.085px/ms`，left/top 加线性 transition)，到边尽头 `cornerTurn()` 拐到相邻边(左↓→下→右…)，竖边横姿势"爬墙"(用户接受)。⚠️crabwalk 本是 clawd 的**进场动作**(doc"右键进入时的螃蟹步"，正面原地左右扭→tuck)，非随机溜达；随机巡逻是我们额外加的。
- **清屏/失忆 → 扫地**：`ChatPanel` 加 `sweepAt` 信号，`newChat()`(清屏) 和 `amnesia()`(三个点的失忆，**不是** restartCC/重启CC) 触发，桌宠播 sweeping 5.5s(优先级压过 happy)。换模型(forge)/重启CC 不扫。
- **未做**：眼球追鼠标(平面 img 做不了，需内嵌分层 SVG+JS，仅电脑有用)；电脑端 Electron 常驻桌面版(网页出不了浏览器，iOS 更不可能浮在别 app 上)。

---

（以下为初版 v1~v3 记录，留档）



**做了什么**：聊天 A（`/chat/`）右下角加了一只桌宠，随澄的状态实时变动作（12 态）。纯前端、**只 `npm run build` 没重启后端**（不让澄失忆），状态全从 ChatPanel 现成 WS 信号推导，**没碰后端**。

**美术=直接用 clawd-on-desk 官方 gif（用户拍板）**：
- 调研结论：`github.com/rullerzhou-afk/clawd-on-desk` 的**代码是 AGPL-3.0**（不是列表写的 MIT，传染 copyleft，故没抄它代码、机制全自写）；**美术 assets/LICENSE = All-Rights-Reserved**（Clawd 形象写"归 Anthropic"、仅非商用；三花猫归作者鹿鹿）。`CyberSealNull/CcCompanion`(MIT) 的桌宠只是 `pet_state.py` 状态转发器、零美术，无参考价值。
- **决定**：我一开始自绘了占位/像素 SVG 蟹（`<CrabArt>`），但用户要的就是 clawd 那只原版小螃蟹，且明确**本人个人·非商用·密码门内单人使用**（站点 chat.jessaminee.top 有 PasswordGate）。判断：严格说 LICENSE 的"个人使用"只覆盖用他原 app，搬进 cheng 算灰色，但私人非商用风险≈0，用户的东西用户定。**故改为直接用官方 gif**，并留作者署名。
- ⚠️ **红线（写给未来 CC，别越界）**：这只能私人用——**别公开宣传、别商用、保持密码门**。若日后 cheng 要开放/商用，必须换成自绘美术（`<CrabArt>` 那版 SVG 还能从 git 历史捞回当起点）。

**改了什么**（已 build 进 dist）：
- `[前端]` `cheng-memory/public/pet/`（新）：**用 SVG 不用 gif**。先试 gif（`/tmp/clawd/assets/gif/`）发现两个坑：① gif 画布 302×300、螃蟹只占底部一小块、自带屏幕黑边(mini)；② **gif 256色+1bit透明 → 柔和渐变被压成死黑硬边、呼吸动画跳帧**(用户看出"黑线不变浅、跳帧")。改用 `/tmp/clawd/assets/svg/` 的**矢量源**：14 个按状态命名的 `*.svg`（idle/thinking/typing/building/carrying/juggling/conducting/error/notification/sweeping/happy/sleeping/yawning/dozing），共 ~150K + `ART-LICENSE.txt`(署名)。SVG **自带 `<style>@keyframes` 动画**(用 `<img>` 加载内嵌 CSS 照样跑)、**viewBox 统一 `-15 -25 45 45` 紧贴螃蟹**(切换不跳、不用裁画布)、矢量平滑无死黑无跳帧。映射注：clawd 无 conducting svg→借 working-debugger；idle→idle-living。静态文件、不进 JS 包、用到才下、下过缓存。
- `[前端]` `cheng-memory/src/DeskPet.jsx`（新）：自包含组件。① 12 状态机（idle/thinking/typing/building/carrying/juggling/conducting/error/notification/sweeping/happy/sleeping）+ 睡眠连续剧（idle 60s→yawning→dozing→sleeping，clawd 无独立 yawning/dozing 故复用 idle/sleeping）+ one-shot（error/happy/notification/sweeping 放一遍回原态）。② 工具名→状态：Bash→building、Read/Grep/Glob/WebFetch→carrying、Edit/Write→typing、Task→juggling、多工具并行→conducting、tool_result.is_error→error（照 clawd-hook 思路）。③ 渲染=单 `<img>` 按状态切 `/pet/{state}.svg`，object-fit:contain 居中铺满（SVG viewBox 已紧贴螃蟹+留白，不用裁/缩放 hack）。④ **拖动 + 迷你模式（照 clawd 原版「极简模式」重做）**：pointer 事件拖动，落点离左右边 <26px → 贴边进迷你(`mini`= "left"/"right" 存 `deskpet-mini`)、`translateX(±58%) scale(.9)` 藏到边外只露一点；**hover/抓取/新动静**才探头(`peek`)；从边上拖出来 or 点一下 → 恢复正常；位置存 `deskpet-pos`。框 104→**128**(用户嫌小)。**迷你专用姿势全 9 个都接上**(照原版)：贴边歇 mini-idle、探头招手 mini-peek(arm-wave)、干活 mini-typing、完成 mini-happy、睡 mini-sleep、犯困 mini-enter-sleep、提醒/报错 mini-alert(X眼)、进极简过场 mini-enter(~520ms)、**随机沿边溜达 mini-crabwalk**(每~5.2s·45%几率·只在 idle/yawn 时·top 加 2.5s transition 平滑滑动·`walking` class)。贴边时也按澄状态切姿势(`miniArt()`)。贴左边整体 `scaleX(-1)` 镜像朝右。`out`(hover/活动/溜达/过场)=滑出可见、否则 translateX 藏一半。框 104→**128**、迷你姿势再 `scale(1.25)`(viewBox 里偏小)。⚠️**坑历史**：先做成"点一下才缩"且没拖到边触发、没 hover 探头 → 用户报"自动躲没见过/拖边没反应"，查 README 才知原版是"拖到边藏+hover探头"一个功能，遂重做。⑤ **点击彩蛋（照 hit-renderer 原逻辑 + clawd/theme.json）**：400ms 窗口、仅 idle 且没在反应时——戳2~3下→50% react-annoyed(3.5s) 否则朝戳的侧 react-left/right(2.5s)；连戳4下+→react-double / react-double-jump 随机(3.5s)；拖拽中→react-drag。⚠️改成原版：**正常模式点一下=彩蛋**(不再是收进迷你)，**进迷你只靠拖到边**，迷你里点一下=弹回。⑥ 美术优先级(正常)：拖拽>彩蛋>idle随机>状态。⑦ **眼球追踪仍没做**：要追鼠标得自绘分层 SVG 用 JS 驱动(原版 idle-follow 也是 JS 驱动眼球，<img> 加载不跑 JS)，与"直接用原版 svg"二选一，用户选原版。

**⚠️v2 重大修正（用户嫌"挤牙膏"，遂把 clawd `tick.js`/`state.js`/`theme.json` 完整读一遍照搬）**：
- **睡眠改成"用户鼠标静止"计时**(原 v1 错按澄活动)：全局 `pointermove`/`pointerdown` 重置 `mouseStillRef`(只写 ref 不渲染)。静止 **20s→随机播一次** idle动作(原 v1 错做成每8s抽40%，原版是满20s播一次、动鼠标才重置)，**60s→哈欠(3s)→犯困(dozing)**，**10min(deepSleep)→深睡(sleeping)**。澄干活时也重置(不睡)。
- **鼠标一动→醒**：检测 asleep→awake 跳变 → 播 `wake.svg`(waking 1.5s)→idle。修了"睡着点它没反应"。
- **一次性状态用真时长**(原 v1 一律 1.8s 错)：照 theme.json autoReturn —— happy/attention 4s、error 5s、notification 5s、sweeping 5.5s。
- 时长常量照 theme.json：mouseIdle 20s/mouseSleep 60s/yawn 3s/deepSleep 600s/wake 1.5s。
- 现 **33 个 svg**(+wake.svg)。原版完整规格存档见本 session transcript `0465da89`(读了 docs/guides/state-mapping.zh-CN.md 全文 + tick.js/state.js/theme.json)。

**⚠️v3 修正 crabwalk（用户看出沿边跑不对）**：查 `mini.js`+`docs/project/theme-state-ui.md` 确认 **clawd 的 mini-crabwalk 是「进极简模式时螃蟹横着挪到屏幕边」的一次性进场动作**（"右键进入时的螃蟹步"，CRABWALK_SPEED 0.12px/ms 横向），**不是随机溜达**。v2 我自己加的"随机上下沿边走"是错的(竖边配横走姿势别扭+弹离边)，已删。改：进极简序列 = crabwalk(横挪到边 850ms，`.crabwalking` 加 `left` 过渡滑动) → mini-enter(520ms) → 贴边歇。`b96b41c` 之后的改动**尚未 commit**(待用户验)。
- `[前端]` `public/pet/` 现 **32 个 svg**(~260K)：14 主状态 + 9 迷你 + 6 react 彩蛋 + 3 idle随机(look/bubble/reading)。
- `[前端]` `cheng-memory/src/ChatPanel.jsx`：import DeskPet + cp-root 顶部挂 `<DeskPet signals={{isGenerating, streamSnap, ccStatus}} />`。

**体积/性能**：gif 是静态文件不进 JS（chat 包反而从 199→190KB，因去掉了自绘 SVG+CSS keyframes）；运行时只一只宠物同时只播一态，不卡；首次每态下个 ~50-160KB gif、之后缓存 0 流量。

**待办**：① 用户刷新 `/chat/` 实测随状态动。② **还没 git commit/push**（待用户看过满意再提；⚠️提交时注意 public/pet/ 那 1.1M gif 会进仓）。回滚=删 DeskPet.jsx + public/pet/ + 撤 ChatPanel 两行 + rebuild。

transcript 指针：本轮 session `0465da89`，关键词"clawd 小螃蟹""DeskPet""public/pet"。

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
