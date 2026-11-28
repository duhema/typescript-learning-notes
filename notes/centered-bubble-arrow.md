# 简化气泡定位，把尖角固定在气泡边线中间

接[四向定位教程](review-step-display.md)。位置和尖角共用一次方向判断。修改 `index.ts` 的 `updateBox()`。

## 1. 已按上一版改好的，只替换两行

在 `updateBox()` 里找到 `let x: number = rect.left;` 和紧接着的 `let y: number = rect.top;`，用下面两行替换。

```ts
let x: number = rect.left + (rect.width - size.width) / 2;
let y: number = rect.top + (rect.height - size.height) / 2;
```

尖角固定在气泡边线的中点。目标宽 100px、气泡宽 280px 时，如果两者左边对齐，尖角会落在目标右边界外 40px。第一行把气泡往左移 90px，尖角就对准了目标的水平中点。第二行用相同的方式对齐垂直中点。

上下放置时，方向判断会重新设置 `y`，保留算好的 `x`。左右放置时，会重新设置 `x`，保留算好的 `y`。这样四个方向的尖角都能对准目标对应边的中点。靠近窗口边缘时，气泡仍可能部分超出窗口。

## 2. 核对完整的定位片段

下面是从 `let x` 到 `arrow.style.display` 的最终代码。

```ts
let x: number = rect.left + (rect.width - size.width) / 2;
let y: number = rect.top + (rect.height - size.height) / 2;
let angle: number;

arrow.style.left = '50%';
arrow.style.top = '50%';

if (below >= size.height) {
  y = rect.bottom + tip;
  arrow.style.top = '0';
  angle = 0;
} else if (above >= size.height) {
  y = rect.top - size.height - tip;
  arrow.style.top = '100%';
  angle = 180;
} else if (right >= size.width) {
  x = rect.right + tip;
  arrow.style.left = '0';
  angle = -90;
} else {
  x = rect.left - size.width - tip;
  arrow.style.left = '100%';
  angle = 90;
}

bubble.style.left = `${x + window.scrollX}px`;
bubble.style.top = `${y + window.scrollY}px`;
arrow.style.transformOrigin = '50% 100%';
arrow.style.transform = `translate(-50%, -100%) rotate(${angle}deg)`;
arrow.style.display = 'block';
```

两行 `'50%'` 先把尖角放在气泡中央。每个方向只改一个坐标，把尖角推到对应的边上。例如气泡放在目标下方时，`arrow.style.top = '0'` 把尖角推到上边，横向位置仍是 `50%`。

如果前面还留着以下两行，删掉它们：

```ts
const centerX: number = rect.left + rect.width / 2;
const centerY: number = rect.top + rect.height / 2;
```

## 3. 避免旧遮罩撑宽页面

在 `handleResize()` 和 `proceedToStep()` 中各找一次 `updateMask(target)`。确认两个地方都按下面的顺序执行。已有第一行的地方只保留一次。

```ts
box.replaceChildren();
const updatedMask: SVGSVGElement = updateMask(target);
```

`updateMask()` 会读取页面的宽高。复查时，把窗口从 970px 缩到 700px，如果旧遮罩还在页面上，测到的宽度仍是 970px。先移除旧遮罩，再测量，新遮罩就能缩到 700px。

## 4. 目标被隐藏时是否显示弹窗

`updateBox()` 开头已有的 `if (target === null)` 处理没有选择器或找不到元素的步骤。如果每一步的目标都会显示，这里可以保留原样。

如果某一步的目标可能通过 `display: none` 隐藏，而且此时应该显示居中弹窗，把原来的判断改成下面这样。隐藏的元素仍然能被 `querySelector()` 找到，但它没有布局矩形。

```ts
if (target === null || target.getClientRects().length === 0) {
  box.append(createPopup());
  return;
}
```
