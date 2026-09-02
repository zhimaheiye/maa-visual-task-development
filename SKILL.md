---
name: maa-visual-task-development
description: >
  面向 MaaFramework 项目的 AI 辅助视觉自动化任务开发流程。用于把用户已经知道如何人工操作的 GUI 流程，交给本地/低成本执行 Agent 做实机勘察、截图、OCR、bbox/ROI 记录与重复测试，再由高能力模型设计状态机、处理复杂逻辑和疑难 Bug，最终固化为稳定、确定性的 MaaFramework Pipeline。
---

# MaaFramework 视觉任务开发 Skill

## 1. 目标

本 Skill 用于把“用户已经知道怎么人工操作”的流程，低摩擦地转化为稳定、可验证、确定性的 MaaFramework 自动化任务。

核心模式：

```text
用户定义业务规则
    ↓
执行 Agent 实机勘察
    ↓
生成 exploration 包
    ↓
高能力模型设计状态机
    ↓
实现正式 Pipeline
    ↓
执行 Agent 实机验证
    ↓
失败证据回流
    ↓
局部修正并复测
```

最终运行时应尽量依赖 MaaFramework 的确定性能力，而不是长期依赖 LLM 临场操作。

---

## 2. 开始前先读项目规则

开始任何任务前，先查找并阅读项目自身约束，例如：

```text
AGENTS.md
PRODUCT.md
docs/handoff/CURRENT.md
README.md
已有 feature 文档
```

优先级：

```text
用户当前明确指令
>
项目根级规则
>
本 Skill
>
Agent 自身默认习惯
```

如果本 Skill 与项目已有架构冲突，不得强行套用本 Skill。

---

## 3. 三方分工

### 用户

用户负责：

- 任务名称；
- 起点与终点；
- 人工操作规则；
- 分支策略；
- “什么算一次”；
- 循环次数等业务参数的真实含义；
- 禁止操作；
- 实时纠正 Agent。

用户不需要负责：

- 手工截图；
- 手工量 bbox；
- 手工写 ROI；
- 手工写完整 Pipeline；
- 主动枚举所有视觉变体。

### 执行 Agent

执行 Agent 优先负责：

```text
连接模拟器
截图
OCR
点击 / 滑动
测量 bbox
推荐 ROI
记录状态转移
记录耗时
生成 exploration 包
Schema 校验
运行 Pipeline
保存失败截图与日志
```

探索阶段，执行 Agent 不是默认架构师。

如果不知道下一步怎么办：

```text
停止
→ 截图
→ 描述当前页面
→ 询问用户
```

禁止凭感觉连续乱点。

### 高能力模型

高能力模型优先负责：

- 状态机设计；
- 区分业务状态与视觉变体；
- OCR / TemplateMatch / FeatureMatch / ColorMatch 取舍；
- `next` / `on_error` / loop 设计；
- 跨节点计数与计时；
- 是否需要 Python Agent；
- 疑难 Bug 分析；
- 控制修改范围。

不要让昂贵模型浪费大量额度在：

```text
按钮在哪里
bbox 是多少
OCR 这次读成了什么
动画到底 2.8 秒还是 3.2 秒
```

这些优先由执行 Agent 实机采集。

---

## 4. 先定义业务语义，再碰 Pipeline

必须先明确：

```text
任务名称
起始状态
结束状态
一次任务的定义
是否循环
循环参数的业务含义
失败时停止还是重试
绝对禁止操作
```

特别注意：“一次”不能靠截图猜。

例如：

```text
开贝壳次数 = 2
```

如果业务上“一次”定义为：

```text
点击「立即开始」
→ 完成整轮
→ 点击「太好了」
→ 重新回到初始页
```

那么只有完整闭环后才算：

```text
completed += 1
```

不能擅自解释成“点击继续两次”。

---

## 5. 实机探索阶段

探索阶段不是正式开发。

目标是把用户脑中的流程转成：

```text
真实画面
+ 状态名称
+ 识别候选
+ 动作
+ 动作后的真实结果
+ 耗时
```

每个稳定状态遵循：

```text
等待稳定
↓
截图
↓
观察 / OCR
↓
记录 bbox / 推荐 ROI
↓
只执行一个动作
↓
重新观察
```

禁止“看一眼以后连续点三步”。

详细探索规范见：

```text
references/exploration.md
```

---

## 6. 状态不是单张截图

强制区分：

```text
业务状态
vs
视觉 variant
```

错误模型：

```text
状态 E = E_001.png
```

正确模型：

```text
状态 E
├─ E_001_center
├─ E_002_share
└─ E_003_other_layout
```

如果多个界面变体业务处理相同，优先让一个节点覆盖它们，而不是每个布局建一个新节点。

例如结算页：

```text
单按钮居中「太好了」
双按钮时右侧「太好了」
```

业务上仍然都是同一个 Finish 状态。

---

## 7. 识别策略原则

先问：

```text
这个状态最稳定、最直接决定下一步的可观察特征是什么？
```

而不是：

```text
我要截哪张图？
```

### OCR 优先场景

稳定文字按钮：

```text
领取
继续
太好了
立即开始
打开贝壳
```

优先 OCR。

如果 OCR 结果本身就是按钮，优先让框架直接点击识别结果，不要同时硬编码固定坐标。

### OCR expected 不等于完整 UI 文案

这是重要规则。

PP-OCR 可能把完整按钮文字拆成多个文本框。

例如：

```text
视觉文案：保留一个随机奖品

OCR：
[保留]
[一个随机奖品]
```

此时：

```text
expected = 保留一个随机奖品
```

会失败，因为 expected 是在单个 OCR box 内匹配。

应优先选择：

> 在目标 ROI 内、单个 OCR box 中稳定存在、且足够唯一的最小语义片段。

例如：

```text
expected = 随机奖品
```

比完整长句更抗分词，同时比单独“奖品”更有区分度。

### TemplateMatch

适合：

- 没有稳定文字的图标；
- OCR 长期不稳定；
- 固定视觉特征。

生成模板前应：

```text
从与运行时一致的标准化截图源取图
→ 确定 bbox
→ 裁图
→ 立即 TemplateMatch 回测
→ 记录 score
```

不要因为一次 OCR 失败就立即切模板。

---

## 8. bbox 与 ROI

必须区分：

```text
bbox
= 本次实际检测到的目标范围

recommended_roi
= 正式识别时建议搜索的范围
```

ROI 应覆盖：

- 已知视觉变体；
- 轻微位置偏移；
- 合理 UI 动画余量。

但不要无证据无限扩大到全屏。

如果出现新 variant 导致漏检：

```text
先比较旧 bbox / 新 bbox / 旧 ROI
```

若业务状态相同，优先局部扩大已有 ROI，而不是新建专用状态。

---

## 9. 动画处理

第一版不要为了省几秒而过度设计。

如果：

```text
点击
→ 动画 3 秒
→ 稳定页面
```

优先让 Pipeline 等待可识别的稳定后继状态。

不要一开始就为“点击跳过动画”增加额外状态和分支，除非动画确实很长或跳过路径极其稳定。

---

## 10. Exploration 包

推荐：

```text
dev/exploration/<task_name>/
├─ README.md
├─ flow.json
└─ screenshots/
   ├─ A_001.png
   ├─ B_001.png
   ├─ E_001_center.png
   ├─ E_002_share.png
   └─ UNKNOWN_001.png
```

默认视为本地开发材料：

```text
不进入 Release
不自动提交 Git
```

示例结构见：

```text
assets/flow.example.json
```

---

## 11. 未知状态

出现计划外稳定界面：

```text
停止自动连续操作
↓
截图
↓
标记 UNKNOWN_xxx
↓
记录由哪个动作进入
↓
记录文字 / 按钮 / 页面特征
↓
判断是动画还是新状态
```

如果需要业务判断，询问用户。

禁止自行发明业务逻辑。

---

## 12. 状态机设计

探索完成后，先画状态图，再改代码。

状态节点应表达：

```text
“我确认当前是什么状态”
```

而不是：

```text
“我希望接下来发生什么”
```

复杂页面结束后，推荐使用路由节点观察可能的多个稳定后继状态，而不是固定 sleep 后假定结果。

例如：

```text
ResultRouter
├─ 特殊终止状态
├─ 正常终止状态
└─ 普通循环状态
```

---

## 13. 循环与计数

不要把 MaaFramework 节点的 `repeat` 误用于“重复整条子流程”。

`repeat` 通常只是重复当前节点 action。

若业务需要：

```text
完整 A → ... → A
循环 N 次
```

应明确建立轮次计数逻辑。

计数点必须放在业务上真正完成一轮的位置。

如果项目已有 Python Agent，可以使用最小 Custom Recognition / Action 管理：

```text
completed
target_count
task_id
```

并用 `task_id` 防止新任务继承上一次状态。

---

## 14. 开发测试策略

不要开发阶段一上来跑 100 / 200 次。

优先：

```text
N = 1
→ 验证一轮完整闭环

N = 2
→ 验证循环与停止边界

任务重启 N = 1
→ 验证状态重置
```

只有小样本稳定后，才增加长时间或大次数验证。

---

## 15. 现场 Bug 处理

遇到失败时：

```text
先采样
再修改
```

禁止：

```text
一失败就乱加 fallback
一失败就扩大所有 timeout
一失败就改成固定坐标
```

现场至少保留：

```text
失败节点
完整截图
OCR 实际结果
bbox
最近日志摘要
预期状态
实际状态
关键参数
```

如果本地截图路径对远端模型不可见：

> 执行 Agent 应优先把现场 PNG 直接作为聊天附件注入，并同时附上失败节点、预期、实际和日志摘要。

已验证可用的方式包括：

```text
系统剪贴板 + 模拟真实粘贴
或
DataTransfer File 自动化挂载
```

远端模型确认看到图片后，再进行视觉分析。

详细 Bug 处理规则见：

```text
references/troubleshooting.md
```

---

## 16. Git 安全

默认：

```text
允许本地修改
允许本地测试
不自动 commit
不自动 push
```

以下默认视为私人 / 临时开发材料，除非用户明确要求，不得提交：

```text
dev/exploration/
notes/
临时截图
测试日志
私人待办
待办.md
scratch/
```

禁止默认执行：

```text
git add .
git add -f <ignored-file>
```

提交前必须检查：

```bash
git status --short
git diff --cached --name-only
```

若发现用户明确说过不要上传的文件，停止提交。

---

## 17. 完成标准

一个任务不能因为“Schema 通过”就视为完成。

至少要求：

```text
Schema 校验通过
关键状态实机命中
完整闭环成功
循环边界正确
任务重启状态正确
至少覆盖已知视觉 variants
没有误触危险操作
```

出现新视觉变体时，优先补充 exploration 证据并局部修正，不推翻已经正确的主状态机。

---

## 18. 推荐阅读

本 Skill 的详细参考：

```text
references/exploration.md
references/troubleshooting.md
references/open-shell-case-study.md
assets/flow.example.json
```
