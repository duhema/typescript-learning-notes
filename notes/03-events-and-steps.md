# 03 · 用数组管理步骤，用一个按钮推进

一组说明可以写成数据：每一项提供文字，以及可选的目标选择器。DOM 节点只创建一次，点击按钮时切换数据。

## 定义一条步骤的数据

```ts
type GuideStep = {
  content: { title: string; message: string };
  selector: string | null;
};
```

selector 为 string 时，表示一个 CSS 选择器；为 null 时，表示不依附页面元素。content 只保存文字，不是 DOM 节点。

例如，一个流程可以包含开始说明、几个页面区域和结束说明：

```ts
const steps: GuideStep[] = [
  { content: tips.popup1, selector: null },
  { content: tips.step1, selector: tips.selectors.step1 },
  { content: tips.step2, selector: tips.selectors.step2 },
  { content: tips.step3, selector: tips.selectors.step3 },
  { content: tips.step4, selector: tips.selectors.step4 },
  { content: tips.step5, selector: null },
];
```

这段使用 tips 中的文案与选择器：每项文案包含 title、message 两个字符串，selectors 中的每项保存目标的 CSS 选择器。增加一条数据不等于增加一个按钮。

## 把准备工作和流程分开

initialize 负责创建按钮、文字节点与 steps。start 接收这些对象，组织显示和事件。函数参数可以明确写出类型：

```ts
const start = (
  closeButton: HTMLButtonElement,
  stepButton: HTMLButtonElement,
  title: HTMLParagraphElement,
  message: HTMLParagraphElement,
  steps: GuideStep[]
): void => {
  // 在这里准备 popup、mask、定位方法和步骤事件。
};
```

这是函数签名示例；流程代码应放入同一个 start 函数体内，避免重复声明同名函数。参数中的节点是传进来的同一批对象，不需要再创建副本。

## 状态要在首次使用之前初始化

下面是需要避免的顺序：

```ts
// 阅读用反例，不运行。
// const showStep = (index: number): void => { currentStep = index; };
// showStep(0);
// let currentStep = 0;
```

调用 showStep 时会读取或赋值 currentStep。let 变量还没有完成初始化，就会触发运行时错误。正确顺序是先声明状态，再注册事件，最后开始显示。

## null 不是选择器

```ts
const target = step.selector === null
  ? null
  : document.querySelector<HTMLElement>(step.selector);
```

这段假设 step 是已取得的步骤。querySelector 接收字符串，不应把 string|null 直接传进去。

“没有目标”与“目标没找到”也要区分：前者是正常的开始或结束说明；后者可能是选择器错误，应保留当前状态并给出提示。

## 组织显示、下一步和结束

以下内容位于 start 内，使用已经创建的 popup、mask、positionPopup 和 drawMask。positionPopup 见第 02 节，drawMask 见第 04 节。

```ts
// 状态先初始化，之后才调用 showStep(0)。
let currentStep = 0;
let ended = false;

const finish = (): void => {
  if (ended) return;
  ended = true;
  mask.remove();
  popup.remove();
};

const showStep = (index: number): void => {
  if (ended) return;
  if (index >= steps.length) {
    finish();
    return;
  }

  const step = steps[index];
  // null 表示这一条没有目标；不把 null 传给 querySelector。
  const target = step.selector === null
    ? null
    : document.querySelector<HTMLElement>(step.selector);

  if (step.selector !== null && target === null) {
    console.error('找不到引导目标：', step.selector);
    return;
  }

  title.textContent = step.content.title;
  message.textContent = step.content.message;
  stepButton.textContent = index === steps.length - 1
    ? tips.button.overButton
    : tips.button.stepButton;
  currentStep = index;
  popup.replaceChildren(closeButton, title, message, stepButton);

  // 切换步骤时先把目标带入视口；即时滚动完成后再测量。
  // 普通手动滚动期间没有定位回调。
  target?.scrollIntoView({ block: 'center', inline: 'nearest', behavior: 'instant' });
  positionPopup(target, popup);
  drawMask(target);
};

stepButton.addEventListener('click', () => showStep(currentStep + 1));
closeButton.addEventListener('click', finish);
showStep(0);
```

执行顺序可以这样读：

```text
currentStep = 0
      ↓
showStep(0) → 开始说明
      ↓ 点击
showStep(1) → 第一个目标
      ↓ 点击
……
      ↓
showStep(steps.length - 1) → 最后一条，按钮显示「終了」
      ↓ 点击
showStep(steps.length) → finish()
```

只有目标确认存在后，才更新 title、message 和 currentStep。最后一项的索引是 length-1；再往后才是越界结束，两个判断不要混在一起。

replaceChildren 使用的是已有按钮和文字对象，按钮上的事件仍然属于原节点。点击监听只注册一次，不能在每次 showStep 中重复注册新的匿名回调。

## 小练习

1. steps 有 6 项时，最后一项的索引是多少？
2. 一条步骤 selector=null，另一条 selector='.missing' 但页面找不到它，两者应如何处理？
3. 只在点击事件里 console.log('下一步')，是否会更改页面说明？
4. 若要增加一条说明，需要再创建 stepButton2 吗？

<details>
<summary>参考答案</summary>

1. 5。索引 6 表示已经走完整个数组。
2. null 正常显示不依附目标的说明；找不到非空选择器的目标时，保留当前画面并报告问题。
3. 不会。需要实际更新文案、位置和步骤状态。
4. 不需要，添加一项 steps 即可；最后一步的判断依然根据数组长度。

</details>

继续阅读：[SVG 遮罩与操作区域](04-svg-and-interaction.md)。
