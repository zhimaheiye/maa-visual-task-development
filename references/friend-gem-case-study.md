# Completed Case Study：MaaHappyFish「好友摸宝」任务

这个案例展示一种完整的 AI 辅助视觉自动化开发方式：用户只提供基于记忆的业务描述，执行 Agent 负责逐步实机探索，高能力模型根据现场证据持续校准假设，形成确定性 MaaFramework Pipeline；随后再经过用户真实使用验收，发现 Agent 长跑未暴露的问题，并以已知限制的形式完成交付。

案例来源：[`zhimaheiye/MaaHappyFish`](https://github.com/zhimaheiye/MaaHappyFish)

关键提交：

```text
6369811  feat(friend): 新增好友水族箱摸宝巡访与产物收取自动化
867ddeb  feat(friend): 捕获长尾系统弹窗并记录 44 分钟 E2E
03ccef8  fix(friend): 扩大 FriendGemExhausted ROI 防字符截断并增加跳过日志
```

当前源码基线以 `03ccef8` 为准。

最终状态不是“100% 无失败”，而是：

```text
主要 ROI 截断根因已修复
→ 用户实际使用反馈明显改善
→ 仍有小概率漏判
→ 用户接受当前效果，暂不继续优化
→ 已推源码，未发布新 Release
```

---

## 1. 起始需求：用户只提供可能不准确的记忆

最初业务规则大致是：

```text
从好友列表第一个好友开始
→ 进入好友水族箱
→ 检查可能出现的欢迎弹窗
→ 收取跟着鱼移动的产物
→ 每位好友最多处理若干次，V1 先以 12 次作为安全上限
→ 通过右上角“下一位”连续切好友
→ 最后进入加好友页面时结束整个任务
```

用户明确提醒这些描述来自记忆，需要实机验证。

因此把用户记忆视为：

```text
业务方向 / 探索提示
```

而不是：

```text
已验证 UI 事实
```

---

## 2. Start Contract

最终任务明确要求：

> 必须在「我的星级好友」列表页面顶部开始任务，并确保第一排好友卡片可见。

V1 不负责从自己的鱼缸导航到好友列表，只负责：

```text
点击第一位好友
→ 巡访好友水族箱
→ 收取产物
→ 跳过耗尽好友
→ 切下一位
→ 好友末尾结束
```

Start Contract 避免把额外导航无必要地塞进核心状态机。

---

## 3. 显眼视觉元素不等于业务特征

第一轮好友列表截图里，某些好友卡片有明显金色装饰角标。

AI 一度把它列为“可能与可收产物有关”的候选特征，但用户根据游戏经验明确指出：

```text
金色角标只是装饰
≠ 是否有产物
≠ 是否应该进入
```

因此立即移除。

经验：

> 用户可靠的领域知识可以否决 AI 对截图的视觉联想；不要因为某个元素显眼就赋予它业务意义。

---

## 4. 异常页面也可能是高价值 visual variant

进入第一位好友后，网络原因导致鱼显示为黑色问号占位，但鱼头上的金币气泡仍正常出现。

没有把它丢弃为“坏截图”，而是记录为：

```text
FriendTank
└─ variant: fish_assets_unloaded
```

项目已有模板：

```text
assets/resource/image/金币气泡.png
```

在黑色占位鱼和正常彩色鱼上都能稳定命中，证明目标视觉与鱼模型资源基本解耦。

---

## 5. UI 数字可能不是实时业务状态

首次单点气泡后，当前页面的 `12 剩余` 没有立即变化，也没有明显收益动画，只出现“存档中...”。

如果只看当前帧，很容易错误认为点击无效。

切换好友后，服务器状态刷新，金币变化证明点击已经到账。

经验：

```text
当前页面静态显示
≠ 服务器已同步的真实状态
```

尤其在联网游戏里，不要把“数字没立刻变”当成动作失败的唯一证据。

---

## 6. 局部 Exhausted ≠ 全局 Done

进入一位此前已经处理过的好友时出现：

```text
灰色闪电
0(0点刷新体力)
```

一度可能被误解为整个任务体力耗尽。

继续探索确认：

```text
X 剩余
```

是当前好友自己的局部配额。

因此：

```text
FriendGemExhausted
≠ TaskDone

FriendGemExhausted
→ FriendGemNextFriend
```

真正全局结束由好友末尾的加好友页面决定。

经验：

> `0 / exhausted / 无剩余` 必须先确认作用域：当前对象、当前轮次还是整个任务。

---

## 7. OCR 识别业务语义，不绑定恢复时间

耗尽文字可能是：

```text
0(0点刷新体力)
0(12点刷新体力)
```

对业务而言真正稳定的是：

```text
刷新体力
```

因此使用：

```text
expected = "刷新体力"
```

而不是绑定 `0点` 或 `12点`。

这与“开贝壳”的 OCR 分框案例共同说明：

> OCR expected 应选择决定业务状态的稳定最小语义片段，避免绑定变化数字、时间或装饰文本。

---

## 8. 单目标时序实验决定 post_delay

为了确认气泡点击后是否会残留导致重复点击，执行 Agent 选择与其它目标距离大于 110 px 的孤立气泡并高速抓帧：

```text
0.00s：score = 0.9595
+0.25s：原位置 score = 0.1528
+0.65s：原位置 score = 0.4060
+1.20s：附近已有其它带泡鱼游入
```

结论：成功点击的气泡在 0.25 秒内快速消失。

正式节点使用：

```text
post_delay = 500ms
```

这是建立在实机时序上的安全余量，不是拍脑袋等待。

---

## 9. 安全 ROI：移动目标也不必全屏搜索

好友鱼缸四周有大量固定 UI：

```text
左侧金币 / 体力 HUD
右侧菜单
顶部好友切换按钮
底部区域
```

正式气泡识别收窄到：

```text
ROI = [230, 140, 820, 470]
threshold = 0.75
```

200 个水族箱长链路中，气泡匹配 score：

```text
0.7658 ~ 0.9845
```

没有发生左侧 HUD 或边缘按钮误触。

经验：

> 对会持续移动的目标，保守安全 ROI 往往比“全覆盖”更可靠。

---

## 10. 每好友 12 次是硬保护，不是假装解析服务器体力

用户记忆里不同好友可能有 12 / 10 / 5 等档位。

V1 不解析全部数字，而采用：

```text
每好友最多执行 12 次气泡点击
```

并明确：

```text
attempts
= Pipeline 执行过多少次“识别气泡 → Click”
```

不是：

```text
获得了多少金币
实际消耗了多少服务器体力
```

因为重叠气泡可能让一次点击产生多个业务效果。

---

## 11. 运行时状态职责分离

为了让 Recognition 与 Action 共享状态又避免循环导入，新增：

```text
agent/runtime_state.py
```

共享：

```python
friend_gem_state = {
    "attempts": 0,
    "max_attempts": 12,
    "current_friend_index": 1,
}
```

职责：

```text
InitFriendGemStateAction
→ 显式初始化

RecordFriendGemAttemptAction
→ attempts += 1

CheckFriendGemLimitReco
→ 纯判断，不自增

ResetFriendGemAttemptsAction
→ 切换后清零 attempts
→ friend_index + 1
```

Recognition 尽量保持纯判断，状态变化由明确 Action 驱动。

---

## 12. 核心状态机

```text
FriendGemTask
  InitFriendGemStateAction
        ↓
FriendGemEnterFirstFriend
        ↓
FriendGemFriendRouter
  ├─ FriendGemAddFriendPage → FriendGemDone
  ├─ FriendGemWelcomePopup → Close → Router
  ├─ FriendGemSpecialPopup → Close → Router
  ├─ FriendGemExhausted → FriendGemNextFriend
  ├─ FriendGemAttemptLimitReached → FriendGemNextFriend
  ├─ FriendGemCollectBubble
  │      ↓
  │   FriendGemRecordAttempt
  │      └─ Router
  └─ FriendGemNextFriend
         ↓
      FriendGemResetAttempts
         └─ Router
```

当前关键识别：

```text
FriendGemAddFriendPage
OCR: "全部添加"
ROI: [750, 330, 160, 70]

FriendGemExhausted
OCR: "刷新体力"
ROI: [60, 210, 400, 140]

FriendGemCollectBubble
TemplateMatch: 金币气泡.png
ROI: [230, 140, 820, 470]
threshold: 0.75
post_delay: 500ms

FriendGemNextFriend
TemplateMatch: 好友_下一位.png
ROI: [1140, 50, 130, 80]
threshold: 0.70
post_delay: 1200ms

FriendGemSpecialPopup
TemplateMatch: 绿色勾选按钮.png
ROI: [760, 420, 120, 120]
threshold: 0.8
```

`FriendGemWelcomePopup` 仍属于“用户确认可能存在，但完整长跑未获得真实样本”的防护骨架，不能与已验证的系统弹窗混为一谈。

---

## 13. 分层验证：单点 → 短跑 → 长链路

### 单点实验

验证一个动作的真实因果：

```text
点一个孤立气泡
→ 是否生效
→ 是否消失
→ 多久消失
```

### 5 位好友短跑

验证局部循环：

```text
初始化
→ 12 次上限
→ 切好友
→ attempts 清零
→ Exhausted 0 点击跳过
→ NPC / 真人背景兼容
```

### 200 水族箱完整 E2E

冻结主要实现，完整遍历。

当时统计：

| 指标 | 结果 |
|---|---:|
| 总访问水族箱 / 好友数 | 200 |
| 正常采集好友数 | 198 |
| Exhausted 直接跳过 | 2 |
| 气泡匹配 score | 0.7658 ~ 0.9845 |
| 下一好友按钮 score | 0.7510 ~ 1.0000 |
| 未知稳定异常 / 卡死 | 0 |
| 金币净增长 | +597,637 |
| 连续运行时间 | 2645.3 秒 / 44.1 分钟 |

第 200 位还首次触发了低概率系统弹窗，证明长链路不是简单重复短跑。

但是这个案例后来又证明了另一面：

> **完整 E2E 跑通也不能替代用户实际使用验收，更不能证明每个 Recognition 已覆盖所有位置 variant。**

---

## 14. 长尾弹窗：已验证与未覆盖要分开写

第 200 位水族箱首次出现：

```text
进化鱼特效已被关闭，在【设置-进化鱼特效】一栏中可打开哦。
```

随后固化：

```text
绿色勾选按钮.png
FriendGemSpecialPopup
```

证据状态：

```text
FriendGemSpecialPopup
= 已实机验证

FriendGemWelcomePopup
= 用户确认可能存在，但本次 200 水族箱 E2E 未触发
```

不能因为一种弹窗跑通，就声称所有弹窗分支都覆盖。

---

## 15. 用户验收暴露 Agent E2E 没发现的 Bug

在 Agent 完成长跑并认为功能“端到端收敛”后，用户亲自在桌面客户端实际使用，发现：

> 对体力已经耗尽的好友，仍然有概率会继续点击气泡。

这直接推翻了一个过度结论：

```text
Agent 长跑通过
≠ 功能已经正式完成
```

当时 Router 顺序其实正确：

```text
AddFriendPage
→ WelcomePopup
→ SpecialPopup
→ Exhausted
→ AttemptLimit
→ CollectBubble
```

因此问题不是 Exhausted 分支优先级，而是它自己没有被识别出来。

---

## 16. Smoking Gun：ROI 在识别前就把文字裁坏了

现场高频复现得到：

```text
完整画面 OCR：0(0点刷新体力)
完整 bbox：x = 108 ~ 393
```

旧配置：

```text
FriendGemExhausted ROI = [60, 220, 260, 120]
右边界 = 60 + 260 = x320
```

ROI 内 OCR 因此只能得到：

```text
0(0点刷新体
```

而 expected 是：

```text
刷新体力
```

最后一个“力”在识别输入阶段已经被 ROI 裁掉，所以必然漏判。

日志形成完整证据链：

```text
FriendGemExhausted
→ OCR = "0(0点刷新体"
→ expected "刷新体力" 失败
→ AttemptLimit 失败
→ CollectBubble 高分命中
→ 误点
```

这个案例说明：

> **OCR 漏检不一定是 OCR 模型不稳定；先检查输入给 OCR 的 ROI 是否已经把目标裁坏。**

---

## 17. 修 ROI，而不是先削弱 expected

正确修复：

```text
旧 ROI = [60, 220, 260, 120]
新 ROI = [60, 210, 400, 140]

expected 保持："刷新体力"
```

同时增加：

```text
LogFriendGemExhaustedAction
```

明确打印：

```text
[好友摸宝] 当前好友体力已耗尽，跳过气泡采集
```

这样以后可以快速区分：

```text
Recognition 没命中
vs
Recognition 命中但后续状态机有问题
```

这里还有一条重要反模式：

```text
发现 ROI 截断
→ 不要为了“容错”顺手把 expected 从“刷新体力”放宽到“刷新”
```

因为这会把一个确定的漏检 Bug 转化成未来潜在误报。

应优先修复已经找到的物理根因，一次只改主要变量，然后做正反样本回归。

---

## 18. Agent 回归通过仍然不是最终验收

修复后执行 Agent 在多个明确耗尽好友上重新验证，扩大 ROI 后完整 OCR 能覆盖 `刷新体力`，正常好友也没有出现对应误报。

这说明已确认的 ROI 截断根因得到明显修复。

但用户随后再次亲自使用，最终反馈是：

> “依然有概率失败，但比之前好多了，暂时不用优化了。”

因此最终文档不能写：

```text
100% 修复
彻底杜绝
```

正确状态是：

```text
Known Limitation
+
User Accepted
```

也就是说：

- 已知主要根因被修复；
- 用户实际体验明显改善；
- 仍存在小概率残余漏判，可能还有其它未定位原因；
- 用户当前认为继续优化的收益不值得额外成本；
- 因此接受当前版本并允许提交源码。

工程完成不要求虚构“理论 100%”，也不要求无止境追求完美；关键是准确描述当前可靠性和用户接受阈值。

---

## 19. Candidate Ready 与提交纪律

这一轮最终明确了新的交付流程：

```text
Implemented
→ Agent Verified
→ Candidate Ready
→ User Accepted
→ commit / push
→ Release 单独决定
```

执行 Agent 可以自主完成：

```text
探索
代码修改
Debug
ROI 调整
模板回测
本地回归
与高能力模型技术闭环
```

这些普通技术动作不需要频繁停下来打断用户。

真正需要用户介入的主要停点是：

```text
功能已经达到 Candidate Ready
→ 请用户真实使用
```

或：

```text
出现高能力模型也无法替用户决定的业务语义问题
```

用户验收前，不应因为“Agent 自测通过”就擅自提交最终功能。

本案例中，用户接受修复后的已知限制后，源码提交为：

```text
03ccef8
```

同时用户明确要求：

```text
只推源码
暂不打包发布新版本
```

所以：

```text
commit / push
≠ release
```

必须分开授权。

---

## 20. 三方协作的最终升级策略

```text
实机事实未知
→ 执行 Agent 自己截图 / OCR / 测试

技术判断未知
→ 证据交给高能力模型
→ 继续本地修改 / 回归

业务语义未知
→ 高能力模型确认确实需要用户决策
→ 再询问用户

功能达到 Candidate Ready
→ 停下来让用户真实使用验收
```

用户不需要持续充当多 Agent 之间的人工路由器。

---

## 21. 这个案例最终验证出的开发原则

1. 用户的模糊记忆可以作为探索入口，但不能直接当 UI 真相。
2. Start Contract 要明确从哪里开始以及脚本不负责什么。
3. 显眼视觉元素不等于业务特征。
4. 异常页面也可能是高价值 visual variant。
5. 模板高分命中只证明看到了目标，业务效果仍需单点因果实验。
6. UI 数字可能延迟刷新。
7. 局部 Exhausted 与全局 Done 必须分开建模。
8. OCR 应识别稳定业务语义，不绑定变化时间和数字。
9. 移动目标可以通过高速抓帧测量消失时延。
10. 安全 ROI 比无脑全屏覆盖更重要。
11. OCR 异常要比较完整 bbox 与 ROI 边界，排查是否在识别前就被裁断。
12. 已找到 ROI 物理根因时，优先修 ROI，不要无证据削弱 expected。
13. 计数器只记录 Pipeline 真正知道的事件。
14. Recognition 尽量纯判断，状态变化由 Action 驱动。
15. 测试应分为单点、短跑、长链路 E2E、用户实际使用验收。
16. 长跑可以发现低概率状态，但不能证明所有 Recognition variant 都已覆盖。
17. Agent Verified 不等于 User Accepted。
18. 普通技术 Debug 应由 Agent + 高能力模型自主闭环，不频繁打断用户。
19. 用户可以接受带已知限制的版本；此时应准确记录 Known Limitation，而不是宣称 100% Fixed。
20. commit / push 与 Release 是不同交付动作，需要分开处理。

---

## 22. 为什么这个案例特别适合作为 Skill 示例

“开贝壳”主要展示：

```text
状态机修正
OCR 分框
visual variants
完整轮次计数
```

“好友摸宝”则进一步展示：

```text
模糊业务记忆
→ Agent 自主逐步采样
→ 高能力模型持续局部校准
→ 单点时序实验
→ 状态作用域判断
→ 安全 ROI
→ 共享运行时状态
→ 5 位短跑
→ 200 水族箱 / 44 分钟 E2E
→ 第 200 位长尾弹窗
→ 用户真实使用发现残余 Bug
→ ROI 边界截断的 smoking gun
→ 局部修复与回归
→ 用户接受 Known Limitation
→ 提交源码但不 Release
```

它完整体现了本 Skill 的核心目标：

> **让执行 Agent 负责机械但高价值的实机证据采集，让高能力模型负责状态建模和根因判断，让用户只在真正需要业务决策或最终验收时介入，最终把经验固化回确定性的 MaaFramework Pipeline。**
