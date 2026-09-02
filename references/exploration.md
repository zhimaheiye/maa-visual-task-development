# 实机探索与采样规范

本文件用于执行 Agent 在正式修改 Pipeline 前进行实机勘察。

## 1. 探索目标

把用户已经知道的人工流程，转成可用于状态机设计的证据：

```text
当前状态
→ 如何确认当前状态
→ 应执行什么动作
→ 动作后实际进入什么状态
→ 大约耗时多久
```

探索阶段不追求“马上把功能写完”。

---

## 2. 截图来源

优先使用：

```text
MaaFramework Controller
MaaMCP
ADB
```

如果项目运行时以 1280×720 标准化画面工作，那么探索、ROI、模板素材也尽量统一到同一坐标系。

不要把 Windows 桌面截图当作正式 TemplateMatch 素材。

---

## 3. 每一步操作规范

每进入一个稳定页面：

1. 等待画面稳定；
2. 截图；
3. 全屏观察，必要时 OCR；
4. 记录目标文字或视觉特征；
5. 记录 bbox；
6. 提出 recommended ROI；
7. 执行一个动作；
8. 重新截图 / 观察动作结果。

不要连续执行多个动作后再回忆中间发生了什么。

---

## 4. 推荐记录字段

每个状态至少记录：

```json
{
  "state": "C",
  "name": "普通奖励页",
  "screenshot": "screenshots/C_001.png",
  "resolution": [1280, 720],
  "recognition": {
    "type": "OCR",
    "expected": "继续开贝壳",
    "observed": "继续开贝壳",
    "bbox": [720, 620, 270, 65],
    "recommended_roi": [650, 570, 380, 130]
  },
  "action": {
    "type": "ClickRecognitionResult"
  },
  "result_state": "C_or_D_or_E",
  "notes": "点击后进入约3秒开贝壳动画"
}
```

---

## 5. bbox 与 ROI

`bbox` 是本次实际目标框。

`recommended_roi` 是正式识别时建议搜索的区域。

ROI 不要贴 bbox 太紧，应给轻微位置变化和多视觉 variant 留余量。

如果发现同一状态在不同布局下 bbox 不同，应把多个 variant 都记录下来，再计算一个合理统一 ROI。

---

## 6. OCR 采样

第一轮可以先全屏 OCR 找目标。

找到目标后，再测试推荐 ROI 内 OCR 是否仍能稳定命中。

至少记录：

```text
expected
OCR 实际文本
是否被拆成多个 box
每个 box 的 bbox / score
```

如果完整按钮文案被拆开，不要立即认定 OCR 不可用。

例如：

```text
视觉：保留一个随机奖品
OCR：
- 保留
- 一个随机奖品
```

应记录这种分词事实，后续设计 `expected` 时使用单 box 内稳定、唯一的语义片段。

---

## 7. TemplateMatch 候选

只有以下情况才优先标记模板候选：

- 无稳定文字；
- OCR 连续失败；
- 图标视觉非常稳定。

探索阶段可以先记录：

```json
{
  "fallback_candidate": {
    "type": "TemplateMatch",
    "target": "某按钮",
    "bbox": [0, 0, 0, 0],
    "reason": "OCR repeated failure"
  }
}
```

不要求第一轮就到处生产 PNG。

---

## 8. 动画耗时

建议记录三个时间：

```text
action_time
first_changed_frame_time
stable_target_detected_time
```

目标不是科研级精度，而是判断后续应该：

```text
短 post_delay
还是
依靠 next 持续等待
```

不要凭感觉随手填大量固定 sleep。

---

## 9. UNKNOWN 状态

出现未计划页面时：

```text
停止连续点击
→ 截图
→ 标记 UNKNOWN_xxx
→ 记录由哪个动作进入
→ 记录文字和按钮
```

如果 3~5 秒后自动消失，可视为动画候选。

如果稳定存在并需要选择，应当视为新状态，询问用户业务策略。

---

## 10. Exploration 包建议

```text
dev/exploration/<task>/
├─ README.md
├─ flow.json
└─ screenshots/
```

`README.md` 记录：

- 起点 / 终点；
- 一次任务定义；
- 实测分辨率；
- 状态转移；
- 耗时；
- 未观察到的状态；
- 已知 variants；
- OCR 不稳定项。

这些文件默认是开发证据，不自动提交 Git。
