# 02 · 理解步骤状态，为 Bubble 添加指向目标的尖角

前提：已经完成 [01 · Spotlight 挖孔遮罩](01-spotlight-mask.md)。本节沿用 `start(steps)`、`step.content.selector` 和现有的下一步、关闭、窗口 resize 处理。

## 1. currentStep 与步骤封装

原来的 `currentStep` 位于 `start()` 内，是局部变量。这一节先沿用该结构，完成 Bubble 尖角，步骤切换与 resize 的代码仍按第 01 节执行。

如果要求把步骤状态进一步收进 `proceedToStep()`，可以直接使用它的 `index` 参数，并把当前步骤的按钮和 resize 回调一起移进去。关键是离开步骤时移除旧监听，避免重复响应。完整的删除、替换和清理步骤见 [03 · 将当前步骤封装在 proceedToStep() 内](03-step-local-state.md)。

完成本节后继续第 03 节，即可去掉外层 `currentStep`；Bubble 和 Spotlight 的绘制函数不需要再次修改。

## 2. 为什么要分成外层和内容层

Bubble 放在目标下方，顶部的白色小三角指向目标。结构改为：

```text
popup                         外层：定位、阴影、允许尖角伸出去
├── popupContent              内层：背景、内边距、圆角、内容滚动
│   ├── closeButton
│   ├── title
│   ├── message
│   └── stepButton
└── arrow                     装饰用尖角，不接收鼠标事件
```

原来 `popup` 自身使用 `overflowY = 'auto'`。如果直接把尖角放到它的顶部外侧，尖角会被滚动容器裁剪；只把另一轴设置成 `visible` 也不能解决这个组合的裁剪问题。[CSS Overflow 规范](https://www.w3.org/TR/css-overflow-3/#overflow-properties)。

因此让 `popup` 保持 `overflow = 'visible'`，把滚动交给 `popupContent`。尖角与内容层是兄弟节点，长内容滚动时尖角仍固定在外层顶部。

本节继续使用 `.style` 设置样式。小三角使用 CSS 边框，不需要图片、SVG 或新的依赖；这是常见的实现方式，并非唯一的标准写法。

## 3. 替换 createPopup()

位置：`start()` 内。用下面代码替换整个 `createPopup()`。按钮文案仍来自现有的 `tips.button`，不用修改步骤数据。

```ts
const createPopup = (): {
  popup: HTMLDivElement;
  popupContent: HTMLDivElement;
  arrow: HTMLSpanElement;
  closeButton: HTMLButtonElement;
  stepButton: HTMLButtonElement;
  title: HTMLParagraphElement;
  message: HTMLParagraphElement;
} => {
  const closeButton = document.createElement('button');
  closeButton.id = 'closeButton';
  closeButton.type = 'button';
  closeButton.textContent = tips.button.closeButton;
  closeButton.style.padding = '4px 8px';
  closeButton.style.cursor = 'pointer';
  closeButton.style.float = 'right';

  const stepButton = document.createElement('button');
  stepButton.id = 'stepButton';
  stepButton.type = 'button';
  stepButton.textContent = tips.button.stepButton;
  stepButton.style.padding = '8px 16px';
  stepButton.style.cursor = 'pointer';

  const title = document.createElement('p');
  title.id = 'title';
  title.style.marginBottom = '15px';
  title.style.fontSize = '20px';

  const message = document.createElement('p');
  message.id = 'message';
  message.style.marginBottom = '15px';

  const popup = document.createElement('div');
  popup.style.position = 'absolute';
  popup.style.width = '320px';
  popup.style.boxSizing = 'border-box';
  popup.style.borderRadius = '8px';
  popup.style.boxShadow = '0 4px 10px rgba(0, 0, 0, 0.2)';
  popup.style.textAlign = 'center';
  popup.style.zIndex = '9999';
  popup.style.overflow = 'visible';

  const popupContent = document.createElement('div');
  popupContent.style.boxSizing = 'border-box';
  popupContent.style.backgroundColor = '#fff';
  popupContent.style.padding = '20px';
  popupContent.style.borderRadius = 'inherit';
  popupContent.style.maxHeight = 'calc(100vh - 24px)';
  popupContent.style.overflowY = 'auto';
  popupContent.style.overflowWrap = 'anywhere';

  const arrow = document.createElement('span');
  arrow.ariaHidden = 'true';
  arrow.style.display = 'none';
  arrow.style.position = 'absolute';
  arrow.style.top = '-8px';
  arrow.style.width = '0';
  arrow.style.height = '0';
  arrow.style.borderLeft = '8px solid transparent';
  arrow.style.borderRight = '8px solid transparent';
  arrow.style.borderBottom = '8px solid #fff';
  arrow.style.transform = 'translateX(-50%)';
  arrow.style.pointerEvents = 'none';

  popupContent.append(closeButton, title, message, stepButton);
  popup.append(popupContent, arrow);

  return { popup, popupContent, arrow, closeButton, stepButton, title, message };
};
```

宽高为 0 的元素，加上左右透明边框和底部白色边框，会形成朝上的三角形。`top = '-8px'` 让它伸到 Bubble 顶部；宽度是两边各 8px，共 16px。背景色与三角颜色都用 `#fff`，连接处才不会出现色差。[CSS 边框定义](https://www.w3.org/TR/css-backgrounds-3/#borders)。

`ariaHidden` 表明这是装饰，`pointerEvents = 'none'` 防止它拦截鼠标。这里没有给 Bubble 加上 `role="tooltip"`：它包含下一步和关闭等交互控件，不能只因外形像提示框就套用 tooltip 语义。

## 4. 接收新元素引用

紧接 `createPopup()` 定义之后，将原来的整段解构声明（包括后面手写的对象类型）替换为：

```ts
const {
  popup,
  popupContent,
  arrow,
  closeButton,
  stepButton,
  title,
  message,
} = createPopup();
```

返回类型已在函数上声明，TypeScript 会自动推断这些变量的类型，不需要再复制一遍。这只是接收新引用，不会创建第二个弹窗。

## 5. 替换 updateBubblePosition()

位置：`start()` 内原来的同名函数，整段替换。参数保持不变，所以 `proceedToStep()` 和 `handleResize()` 的调用方式不用改。

```ts
const updateBubblePosition = (
  target: HTMLElement | null,
  bubble: HTMLDivElement,
): void => {
  const gap = 12;
  const viewportWidth = document.documentElement.clientWidth;

  // 先约束宽度，再测量。clientWidth 不包含垂直滚动条。
  bubble.style.maxWidth = `${Math.max(0, viewportWidth - gap * 2)}px`;
  bubble.style.transform = '';
  arrow.style.display = 'none';

  if (target === null) {
    bubble.style.position = 'fixed';
    bubble.style.left = '50%';
    bubble.style.top = '50%';
    bubble.style.transform = 'translate(-50%, -50%)';
    return;
  }

  bubble.style.position = 'absolute';
  const rect = target.getBoundingClientRect();
  const bubbleWidth = bubble.getBoundingClientRect().width;
  const targetCenter = rect.left + window.scrollX + rect.width / 2;

  // 优先水平居中；贴近窗口边缘时，将气泡推回可见范围。
  const minLeft = window.scrollX + gap;
  const maxLeft = Math.max(
    minLeft,
    window.scrollX + viewportWidth - gap - bubbleWidth,
  );
  const left = Math.max(
    minLeft,
    Math.min(targetCenter - bubbleWidth / 2, maxLeft),
  );

  bubble.style.left = `${left}px`;
  bubble.style.top = `${rect.bottom + window.scrollY + gap}px`;

  // 气泡移动后，尖角单独对准目标中心。
  const arrowLeft = targetCenter - left;
  const arrowInset = 16; // 8px 圆角 + 三角形半宽 8px。
  if (
    rect.width > 0 && rect.height > 0 &&
    arrowLeft >= arrowInset &&
    arrowLeft <= bubbleWidth - arrowInset
  ) {
    arrow.style.left = `${arrowLeft}px`;
    arrow.style.display = 'block';
  }
};
```

要分清两种坐标：

- `targetCenter`、`left` 和 Bubble 的 `top`：文档坐标，所以使用 `getBoundingClientRect()` 后加页面滚动距离。
- `arrowLeft`：相对于 Bubble 左边缘的坐标，所以用目标中心减去 Bubble 的 `left`。

`translateX(-50%)` 将尖角的中心移到 `arrowLeft`，不是让它的左边缘对准目标。气泡在窗口边缘被推开时，尖角也会跟着重新计算，不能始终写死 `left = '50%'`。

气泡与目标相隔 12px，尖角向上伸出 8px，尖端距目标底边还有 4px。想改间距可以调整 `gap`；改三角形尺寸时，需要一起调整边框、负 top 和 `arrowInset`。

本节延续下方展示，页面空间不足时通过整页滚动查看气泡。它没有实现上下自动翻转。先将目标滚动到可见范围再展示该步；如果目标中心在窗口外，或极窄目标贴边导致无法在圆角之间放置尖角，就暂时隐藏尖角，不画一个指错位置的三角形。正常可见的目标会显示尖角。

## 6. 修改切步时的 replaceChildren()

位置：`proceedToStep()` 的末尾。把原来的 `popup.replaceChildren(...)` **替换**成下面这三行，不要另加一份调用：

```ts
popupContent.replaceChildren(closeButton, newTitle, newMessage, stepButton);
updateBubblePosition(target, popup);
updateMask(target);
```

现在文字和按钮在内容层里。如果仍然对外层 `popup` 调用 `replaceChildren()`，会把整个 `popupContent` 和 `arrow` 移除，尖角自然也会消失。

使用原有按钮节点，已经注册的 click 监听仍然保留。更新文字和内容结构后，再测量气泡宽度、摆放尖角；最后更新 Spotlight，继续覆盖气泡可能撑出的文档范围。

上一节的 `handleResize()` 会调用新的定位函数，所以窗口宽度变化后，Bubble 和尖角会一起重新计算。普通整页滚动时，目标、绝对定位的气泡和尖角一起移动，不需要新增 scroll 监听。沿用上一节的文档定位参照前提；对于嵌套滚动容器、fixed/sticky 目标，需要另做滚动跟踪。

`cleanUp()` 不需要再加 `arrow.remove()`：移除外层 `popup` 时，它的内容层和尖角会一起移出页面。上一节的 resize 监听清理继续保留。

## 7. 验证结果

| 操作 | 预期 |
| --- | --- |
| 首尾没有目标的说明 | 气泡居中，不显示尖角 |
| 进入有目标的步骤 | 气泡在目标下方，尖角朝上对准目标 |
| 目标靠近窗口左右边缘 | 气泡横向避让，尖角仍指向目标中心；无法放置时隐藏 |
| 连续点击下一步 | 文案、目标和孔洞切换，内容层与尖角始终保留 |
| 鼠标点击、输入目标内部控件 | Spotlight 孔洞仍可操作 |
| 给 message 放入很长的内容 | 内容层出现滚动，尖角不被裁掉，也不随内部文字滚动 |
| 上下或左右滚动整页 | 尖角与普通文档目标保持对齐 |
| 调整窗口宽度 | 气泡宽度与尖角位置重算，步骤编号不变 |
| 关闭或完成引导 | 气泡、尖角、遮罩一起移除，resize 不再触发引导布局 |

验收时可以在开发者工具中检查：外层有两个直接子节点 `popupContent` 和 `arrow`；只有内容层的 `overflowY` 是 `auto`，外层的 `overflow` 是 `visible`。
