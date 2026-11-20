# 简化气泡定位，把尖角固定在气泡边线中间

接[四向定位教程](review-step-display.md)。位置和尖角共用一次方向判断。修改 `index.ts` 的 `updateBox()`。

## 1. 找到要替换的范围

从 `let x: number = ...` 那一行开始，到 `arrow.style.display = 'block';` 为止，包含首尾两行。保留前面计算 `below`、`above`、`right` 的三行，以及后面结束方法的 `};`。

## 2. 简化气泡的位置

用下面这段替换选中的代码：

```ts
let x: number = rect.left;
let y: number = rect.top;
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

方法开头的 `if (target === null)` 保留。它处理没有目标时的居中弹窗。气泡靠近窗口边缘时可能部分超出，尖角始终位于气泡边线中间。
