# 03 · 将当前步骤封装在 proceedToStep() 内

前提：已经完成 [01 · Spotlight 挖孔遮罩](01-spotlight-mask.md) 和 [02 · Bubble 尖角](02-step-state-and-bubble-arrow.md)。本节沿用已有元素、遮罩和定位函数，调整步骤状态与事件监听的位置。

## 1. 用参数表示当前步骤

原来的 `currentStep` 位于 `start()` 内，是局部变量。如果希望进一步把步骤状态收进 `proceedToStep()`，可以直接使用它已有的 `index` 参数：

- `proceedToStep(0)` 展示第 0 步。
- 这次调用创建的下一步回调使用 `index + 1`。
- resize 回调使用这次调用取得的 `step`，只重新测量当前目标。

这样不再需要外层的 `let currentStep = 0`，也不再需要 `currentStep = index`。本次调用的 `index` 不会被重新赋值，回调通过闭包保留它。[TypeScript 作用域与变量捕获](https://www.typescriptlang.org/docs/handbook/variable-declarations.html)。

如果更喜欢 `currentStep` 这个名字，可以将函数参数 `index` 统一改名为 `currentStep`；不需要额外声明第二个变量。不要在函数内重新写 `let currentStep = 0`，否则每次调用都从 0 开始。

这是与“状态归属当前步骤”要求相符的一种实现。原来的函数局部状态也合法；这次调整的收益是步骤参数与对应回调放在一起，无须维护两份步骤编号。

## 2. 监听放进函数时，必须一起处理清理

如果每次进入 `proceedToStep()` 都直接 `addEventListener()`，却不移除上一条回调，旧回调会继续留在按钮和 window 上。重复触发可能导致日志重复、步骤跳转混乱。它是需要防止的原因之一，不能据此断定某次旧实验一定是这个问题。

本节为每个步骤创建一个局部 `AbortController`，让本步的下一步、关闭和 resize 监听使用同一个 `signal`。离开当前步骤时调用 `abort()`，一次移除这三个监听，再进入下一步或关闭引导。DOM 标准支持通过 `addEventListener()` 的 `signal` 选项管理监听的移除。[DOM 事件监听规范](https://dom.spec.whatwg.org/#dom-eventtarget-addeventlistener)。

这里用 `abort()` 移除的是指定 signal 对应的监听，不会清除页面其他功能的事件。它也不会自动删除 DOM，所以元素仍由 `cleanUp()` 移除。

## 3. 替换整个 proceedToStep()

位置：`start()` 内原来的同名函数。用下面代码整段替换，不要将两份函数或两套监听同时保留。

```ts
const proceedToStep = (index: number): void => {
  const step = steps[index];
  if (!step) {
    cleanUp();
    return;
  }

  title.textContent = step.content.title;
  message.textContent = step.content.message;
  stepButton.textContent = index === steps.length - 1
    ? tips.button.overButton
    : tips.button.stepButton;

  popupContent.replaceChildren(closeButton, title, message, stepButton);

  // 此函数属于当前步骤，每次执行时重新查找目标。
  const updateLayout = (): void => {
    const selector = step.content.selector;
    const target = selector === null
      ? null
      : document.querySelector<HTMLElement>(selector);

    updateBubblePosition(target, popup);
    updateMask(target);
  };

  updateLayout();
  console.log('[step]', index);

  // 当前步骤的三个监听统一创建、统一清理。
  const stepEvents = new AbortController();
  const { signal } = stepEvents;

  stepButton.addEventListener('click', () => {
    stepEvents.abort();
    proceedToStep(index + 1);
  }, { signal });

  closeButton.addEventListener('click', () => {
    stepEvents.abort();
    cleanUp();
  }, { signal });

  window.addEventListener('resize', updateLayout, { signal });
};
```

第一次调用的 `index` 是 0，它的按钮回调会请求第 1 步；进入第 1 步时，新回调捕获的是 1。旧回调已经被移除，因此一次点击只执行当前步骤的下一步操作。

这不是在一次同步调用中递归跑完所有步骤。代码先注册回调并返回，下一次点击事件到来时，才执行新的 `proceedToStep()`。

`updateLayout()` 不调用 `proceedToStep()`，因此 resize 不会切步、不会重复绑定事件，也不会打印新的步骤编号。它每次重新查询 selector，即使页面用新节点替换了目标，也能测量新的节点。

## 4. 替换 cleanUp()

位置：原来的 `cleanUp()`，整段替换：

```ts
const cleanUp = (): void => {
  mask.remove();
  popup.remove();
};
```

删除旧的 `window.removeEventListener('resize', handleResize)`。新的 resize 回调属于步骤内部，由对应的 `stepEvents.abort()` 移除。

三条退出路径分别是：

| 路径 | 监听如何清理 | 元素如何清理 |
| --- | --- | --- |
| 当前步骤点击关闭 | 先 `stepEvents.abort()` | 再 `cleanUp()` |
| 最后一步点击完成 | 先 `stepEvents.abort()`，再请求下一编号 | 找不到下一步时 `cleanUp()` |
| 初始 steps 为空 | 尚未创建步骤监听 | 直接 `cleanUp()` |

本版本的 `cleanUp()` 只负责移除元素。其他地方若以后增加“结束引导”的入口，也必须先结束当前步骤的监听，不能只调用元素清理函数。

## 5. 替换 start() 底部的启动代码

把 `start()` 底部从创建 mask 开始，到原来的关闭按钮监听结束，整段替换为下面三行；`start()` 最后的 `};` 保留。

```ts
const { mask, spotlight, border } = createMask();
document.body.append(mask, popup);
proceedToStep(0);
```

这次替换会删除原来的：

- `let currentStep = 0` 和 `proceedToStep(currentStep)`。
- 外层 `handleResize` 函数及 `window.addEventListener('resize', handleResize)`。
- 外层下一步按钮监听和关闭按钮监听。

新的监听都已经在 `proceedToStep()` 中，不能继续留着旧监听，否则仍会重复响应。完成修改后，旧变量名 `currentStep` 不再出现在可执行代码中。

执行顺序仍然重要：先定义 `proceedToStep()`、`cleanUp()` 等函数，再创建并挂载 mask 和 popup，最后调用 `proceedToStep(0)`。空列表触发清理时，元素引用也已经初始化。

## 6. 与上一节的连接

| 上一节的部分 | 本节处理 |
| --- | --- |
| `createPopup()` 与返回值解构 | 保留，包括 `popupContent` 和 `arrow` |
| `createMask()`、`updateMask()` | 保留 |
| `updateBubblePosition()` | 保留 |
| `proceedToStep()` | 替换为第 3 节 |
| `cleanUp()` | 替换为第 4 节 |
| 元素挂载、外层状态和监听 | 替换为第 5 节 |

Bubble 尖角和 Spotlight 仍使用同一个当前目标。普通整页滚动继续依靠文档坐标保持对齐，不新增 scroll 监听。

上一节提到在动态布局完成后调用外层 `handleResize()`；本版本已经没有这个外层函数。如果以后增加步骤内的展开面板等处理，在那次布局完成后调用本步的 `updateLayout()`，并将对应事件也纳入本步的监听清理。

## 7. 检查日志和事件行为

对于 4 个步骤：

```text
首次展示              [step] 0
点击下一步            [step] 1
调整窗口大小          不新增步骤日志
再点击下一步          [step] 2
再点击下一步          [step] 3
点击完成              移除引导，不打印 [step] 4
```

| 操作 | 预期 |
| --- | --- |
| 连续点击下一步 | 每次点击只前进一条，日志没有旧编号混入 |
| 在不同步骤调整窗口 | 只重新定位当前目标，不回到旧目标 |
| 切步后多次 resize | 步骤编号和文案不变 |
| 目标节点被替换，再 resize | 使用同一 selector 找到新目标 |
| 中途关闭或点击完成后 resize | 不再执行引导布局，不重新出现遮罩 |
| 目标内部点击、输入 | 孔洞仍可操作，Bubble 尖角仍对齐 |
| 空步骤列表 | 直接清理，不发生初始化顺序错误 |

如果一次实际点击仍出现多个步骤日志，先检查旧的按钮监听是否全部删除，以及 `start()` 是否被重复调用；不要通过额外递增、递减 index 来掩盖重复回调。
