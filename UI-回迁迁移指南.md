# BOSS 自动沟通脚本：回退稳定版本后恢复当前 UI 排版

## 目的

如果需要回退到上一个能够正常进入聊天页并自动发送的发行版本，可以只回退自动化运行逻辑，再把当前工作区的面板排版移植回去。

本说明的核心原则是：

> 以稳定发行版的自动化流程为底，以当前工作区的 UI 结构和 CSS 为面；不要把当前工作区整份脚本覆盖回退版本。

当前工作区已经出现过聊天页运行权交接问题，因此整文件复制会把 `RunLease`、导航交接、生命周期围栏等运行逻辑一并带回去，UI 虽然相同，聊天发送问题也会重新出现。

## 版本基准

仓库中的版本关系如下：

| 用途 | 版本/提交 | 说明 |
| --- | --- | --- |
| 稳定运行底稿 | `d18cf50` / `release: v0.1.17` | 作为回退后的自动化逻辑基准 |
| 中间 UI 版本 | `e675e20` / `release: v0.1.18 BOSS control panel UI` | 有部分 UI 改动，但不是最终截图中的排版 |
| 最终 UI 参考 | 当前工作区 `zhipin-auto-greeting.user.js` | 以当前文件中的 `UI.mount()` 和 `injectStyle()` 为准 |

特别注意：`e675e20` 仍然有“任务控制”栏和单独的 `.za-control-bar`，而最终 UI 已经把启动/停止按钮压缩到头部中间、把“今日已投递”放到左侧标题下面。因此不能只把 `e675e20` 当作最终 UI 文件。

## 一、回退时只保留哪些内容

先将脚本恢复到稳定发行版，再从当前工作区手动移植以下三类内容：

1. `UI.mount()` 中的头部 DOM 和监控卡 DOM。
2. `injectStyle()` 中与头部、按钮、监控卡、折叠板块相关的 CSS。
3. UI 初始化和渲染所需的少量 DOM 引用与方法调用。

以下内容不要从当前工作区复制到稳定版本：

- `RunLease`、`FenceLostError`、`RunContext`、`expectedNavigation`。
- `navigator.locks`、`BroadcastChannel`、导航 nonce、租约续期和页面围栏逻辑。
- `RunClock`、`DeliveryLedger` 以及发送恢复链的改写。
- `autoCloseOnTaskEnd` 的运行时实现，除非另行验证自动关闭功能。
- 当前版本对 `resumeIfNeeded()`、`continueFromChat()`、`watchChatPageTransition()` 的改动。

这份 UI 回迁只解决视觉和交互布局，不把新的聊天页恢复机制带回稳定版本。

## 二、头部 DOM 回迁结构

在稳定版 `UI.mount()` 中，找到原来的 `<header class="za-header">`，将头部改成以下层级：

```text
header.za-header
└─ div.za-header-row
   ├─ div.za-header-title
   │  ├─ strong              BOSS自动沟通
   │  └─ span.za-daily-count 今日已投递：x/150
   ├─ div.za-header-stack
   │  └─ div.za-control-actions
   │     ├─ button[data-action="start"]
   │     └─ button[data-action="stop"]
   └─ div.za-header-actions
      ├─ button[data-action="toggleFeaturePanel"] + 编辑 SVG
      └─ button[data-action="toggle"] + 关闭 SVG
```

要点：

- “岗位问候自动化”副标题在最终布局中已删除，不要再放回标题区域。
- “今日已投递”放在 `za-header-title` 内，显示在标题下方左侧。
- 启动和停止按钮放在标题右侧、两个图标按钮左侧。
- 右上角只保留编辑图标和关闭图标，不恢复“板块管理”文字按钮。
- 启动按钮继续使用 `data-action="start"`，停止按钮继续使用 `data-action="stop"`，不要改这两个选择器，否则原有事件绑定会失效。
- 编辑和关闭按钮必须保留原来的 `data-action` 值，只替换显示内容和 class。

“今日已投递”建议保留以下 data-role，保证旧版计数刷新方法仍可使用：

```html
<span class="za-daily-count" data-role="dailyDeliveryCount" aria-live="polite">
  <span class="za-daily-count-label">今日已投递：</span>
  <strong class="za-daily-count-value" data-role="dailyDeliveryCountValue">0</strong>
  <span class="za-daily-count-limit" data-role="dailyDeliveryCountLimit">/150</span>
</span>
```

如果稳定版没有 `dailyDeliveryCountLimit` 这个引用，可以保留旧的整体文本刷新方式；不要为了 UI 回迁修改计数来源。

## 三、监控卡回迁结构

删除稳定版中分开的：

```text
div.za-status
div.za-guard-panel
```

改为一张：

```text
section.za-monitor-card[data-role="guardPanel"]
├─ div.za-realtime-log
│  ├─ span.za-realtime-log-label       实时日志：
│  └─ div/span[data-role="status"]    原状态文本
├─ div.za-guard-grid                   两列网格
│  ├─ 任务状态
│  ├─ 本轮沟通
│  ├─ 本小时刷新
│  ├─ 下次刷新
│  ├─ 已运行时间
│  └─ 预估剩余时间
└─ div.za-guard-current[data-role="guardCurrentStatus"]
```

六个值的 data-role 应保持如下名称：

| 显示项 | data-role |
| --- | --- |
| 任务状态 | `guardRunState` |
| 本轮沟通 | `guardSentCount` |
| 本小时刷新 | `guardRefreshCount` |
| 下次刷新 | `guardNextRefresh` |
| 已运行时间 | `guardElapsedTime` |
| 预估剩余时间 | `guardRemainingTime` |

稳定版没有运行时间字段时，先显示“未开始”和“待估算”即可。不要为了显示这两个字段直接移植 `RunClock` 或修改聊天恢复逻辑；如果以后需要真实时间估算，应单独做一个不依赖运行权的功能补丁。

## 四、UI 引用和方法调用

将 `UI.mount()` 中的 `runtime.ui` 引用补齐：

```js
header: root.querySelector('.za-header'),
realtimeLog: root.querySelector('.za-realtime-log'),
dailyDeliveryCountLimit: root.querySelector('[data-role="dailyDeliveryCountLimit"]'),
guardElapsedTime: root.querySelector('[data-role="guardElapsedTime"]'),
guardRemainingTime: root.querySelector('[data-role="guardRemainingTime"]'),
```

原有的 `status`、`guardPanel`、`guardRunState`、`guardSentCount`、`guardRefreshCount`、`guardNextRefresh`、`guardCurrentStatus` 引用要保留。

在 `runtime.ui = {...}` 完成后调用：

```js
this.observeHeaderHeight();
```

如果稳定版没有 `observeHeaderHeight()`，可以先不迁移这个方法，把板块管理弹层的 `top` 改成固定的当前头部高度；更推荐一起迁移该方法，因为它只负责测量头部高度，不参与自动化运行。

停止按钮的 UI 状态必须保持最高优先级。稳定版的 `setRunning()` 应满足：

```js
if (start) start.disabled = Boolean(running);
if (stop) stop.disabled = false;
```

不要让异常页、人工确认页或“运行中”状态把停止按钮禁用。手动点击停止后，启动按钮才允许恢复点击。

## 五、CSS 回迁重点

从当前工作区 `injectStyle()` 中按选择器迁移，不要整段覆盖稳定版的所有 CSS。需要迁移的选择器和目标值如下：

```css
.za-header-row {
  position: relative;
  min-height: 64px;
  padding: 4px 86px 4px 14px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
}

.za-header-actions {
  position: absolute;
  top: 2px;
  right: 4px;
  display: flex;
  gap: 4px;
}

.za-header .za-icon-btn,
.za-header .za-feature-button {
  width: 36px;
  min-width: 36px;
  height: 36px;
  min-height: 36px;
  border: 0;
  border-radius: 50%;
}

.za-header .za-control-actions button {
  width: 60px;
  min-width: 60px;
  height: 38px;
  min-height: 38px;
  border-radius: 8px;
  font-size: 13px;
}

.za-monitor-card {
  margin: 10px 12px 0;
  padding: 10px 12px;
  border: 1px solid var(--za-divider);
  border-radius: 8px;
}

.za-monitor-card .za-guard-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 6px 10px;
}
```

需要同时迁移：

- `.za-realtime-log` 和 `.za-realtime-log-label`。
- `.za-monitor-card .za-status` 的静态化样式，避免旧 `.za-status` 的阴影和外边距在监控卡内形成“卡片叠卡片”。
- `.za-monitor-card .za-guard-current` 的分隔线和自动换行。
- `.za-feature-panel` 的 `top: calc(var(--za-header-height) + 10px)`。
- 编辑 SVG、关闭 SVG 的尺寸和 `:focus-visible` 焦点样式。

极窄屏只在 `max-width: 360px` 时把监控网格切为单列：

```css
@media (max-width: 360px) {
  #zhipin-auto-greeting-root .za-monitor-card .za-guard-grid {
    grid-template-columns: 1fr;
  }
}
```

430px 面板保持两列，不要使用过早的 `max-width: 430px` 单列规则，否则会导致当前截图中的监控卡过高。

## 六、不要误把 UI 改动带成运行逻辑改动

回迁过程中看到以下代码时，应停下来确认，不要直接复制：

```text
resumeIfNeeded
continueFromChat
watchChatPageTransition
RunLease
expectedNavigation
handoffNonce
BroadcastChannel
DeliveryLedger
RunClock
autoCloseOnTaskEnd
```

这些不是单纯的排版代码。尤其是 `continueFromChat()` 和 `watchChatPageTransition()`，它们直接决定聊天页是否发送问候语；当前工作区正是在这个恢复链上出现卡住问题。

## 七、回迁后的验收顺序

先做静态检查：

```powershell
node --check zhipin-auto-greeting.user.js
git diff --check -- zhipin-auto-greeting.user.js
```

再做 UI 检查：

1. 面板宽度约 430px 时，头部标题、今日投递、启动/停止和两个图标不换行、不重叠。
2. 右上角编辑和关闭按钮均可点击，且点击区域不小于 36px。
3. 启动和停止位于头部中间，不再出现“任务控制”大块白色区域。
4. 实时日志、任务状态和当前状态处于同一张监控卡中。
5. 监控卡在 430px 时保持两列，极窄屏才变成单列。
6. 设置区域仍可按大标题折叠，内部不要出现新的嵌套卡片。

最后做运行回归：

```text
岗位列表 → 点击沟通 → BOSS 默认弹窗 → 进入聊天页 → 稳定版自动发送指定问候语 → 返回列表
```

运行回归时重点确认：

- 聊天页能自动发送，不停在“等待发送”。
- 手动点击停止始终可用。
- 停止后启动按钮可重新点击。
- 刷新页面不会因为 UI 回迁引入新的运行权或人工确认状态。

只有静态检查和截图通过，不能证明聊天发送流程恢复；必须至少实际完成一次“进入聊天页并发送”的测试。

## 最终建议

最稳妥的回迁方式是：稳定版脚本作为唯一运行逻辑来源，当前工作区只作为 UI 参考，按本说明迁移 `UI.mount()`、指定 CSS 和 DOM 引用。不要使用整文件覆盖，也不要把当前未验证的聊天恢复代码一起带回去。
