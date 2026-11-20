# 04 · 用四块 SVG 矩形围出操作区域

让某个区域保持清晰并能操作，可以把灰色背景拆成上、下、左、右四块，中间不画遮罩。另一块仅描边的矩形负责强调目标。

## 先画出位置关系

```text
┌───────────────────────────────┐
│              上方             │
├────────┬──────────────┬───────┤
│  左侧  │   目标区域   │ 右侧  │
├────────┴──────────────┴───────┤
│              下方             │
└───────────────────────────────┘
```

上方和下方占整个文档宽度；左右两侧只占目标的高度，所以四块之间不会重复叠色。

设文档宽高为 W、H，目标文档边界为 left、top、right、bottom：

| 区域 | x | y | width | height |
| --- | --- | --- | --- | --- |
| 上方 | 0 | 0 | W | top |
| 下方 | 0 | bottom | W | H-bottom |
| 左侧 | 0 | top | left | bottom-top |
| 右侧 | right | top | W-right | bottom-top |

坐标计算与提示框使用同一套文档坐标：getBoundingClientRect 加上 window.scrollX / scrollY。根 SVG 采用 absolute，普通整页滚动由浏览器处理。

## 让图形和鼠标区域一致

根 SVG 覆盖文档，但不应该整块接收鼠标。把根设置为 pointerEvents=none，再让灰色矩形接收事件。中间没有矩形，就能点击下面的真实控件。

```ts
// SVG 根只是容器，灰色图形再单独接收鼠标事件。
svg.style.pointerEvents = 'none';

// 灰色区域接收鼠标。
bgRect.style.pointerEvents = 'all';

// 强调框只画边，不挡住目标控件。
border.setAttribute('fill', 'none');
border.style.pointerEvents = 'none';
```

这是属性分工的阅读片段，三个变量需分别对应已创建的 SVG 根、灰色矩形和边框。仅仅降低 opacity 并不能保证可操作；仅把所有 pointerEvents 都设成 none，又会让灰色区域也挡不住点击。

## 绘制函数

在 start 内，mask 是已经加入 body 的 SVG 根，svgNS 是 SVG 命名空间。根元素的 left 和 top 为 0，层级低于 popup，pointerEvents 为 none。

```ts
const drawMask = (target: HTMLElement | null): void => {
  // 测量文档前暂时隐藏旧遮罩，避免旧尺寸影响 scrollWidth/scrollHeight。
  mask.style.display = 'none';
  mask.replaceChildren();

  const addRect = (
    x: number, y: number, width: number, height: number
  ): SVGRectElement => {
    const rect = document.createElementNS(svgNS, 'rect');
    rect.setAttribute('x', String(x));
    rect.setAttribute('y', String(y));
    rect.setAttribute('width', String(Math.max(0, width)));
    rect.setAttribute('height', String(Math.max(0, height)));
    rect.setAttribute('fill', 'rgba(0, 0, 0, 0.5)');
    // 灰色矩形接收鼠标，阻止点击落到它下面的页面控件上。
    rect.style.pointerEvents = 'all';
    mask.append(rect);
    return rect;
  };

  if (target === null) {
    // 无目标的说明只需遮住视口，随页面滚动仍覆盖整个屏幕。
    mask.style.position = 'fixed';
    mask.style.width = '100%';
    mask.style.height = '100%';
    addRect(0, 0, window.innerWidth, window.innerHeight);
  } else {
    // 提示框和遮罩都使用文档坐标，整页滚动时自然一起移动。
    const width = document.documentElement.scrollWidth;
    const height = document.documentElement.scrollHeight;
    mask.style.position = 'absolute';
    mask.style.width = width + 'px';
    mask.style.height = height + 'px';

    const rect = target.getBoundingClientRect();
    const left = rect.left + window.scrollX;
    const top = rect.top + window.scrollY;
    const right = left + rect.width;
    const bottom = top + rect.height;

    // 四块矩形恰好覆盖目标以外的区域，中间没有接收鼠标的遮罩。
    addRect(0, 0, width, top);                        // 上方
    addRect(0, bottom, width, height - bottom);       // 下方
    addRect(0, top, left, rect.height);               // 左侧
    addRect(right, top, width - right, rect.height);  // 右侧

    const border = addRect(left, top, rect.width, rect.height);
    border.setAttribute('fill', 'none');
    border.setAttribute('stroke', '#2563eb');
    border.setAttribute('stroke-width', '2');
    // 强调边框只负责显示，不拦截目标内控件的鼠标操作。
    border.style.pointerEvents = 'none';
  }

  mask.style.display = 'block';
};
```

没有目标时，用 fixed SVG 覆盖整个视口，适合开始和结束说明。有目标时，切换为 absolute，用文档尺寸画四块灰色区域。

测量前隐藏旧 SVG，是为了避免前一次的宽高影响文档尺寸。绘制完成后立即显示，整个过程在同一次函数调用中完成，不需要定时刷新。

## 显示一个步骤时的调用顺序

```ts
// 文字先更新，才能测到这一条说明实际需要的尺寸。
title.textContent = step.content.title;
message.textContent = step.content.message;

// 进入步骤时定位一次，再按当前文档尺寸绘制一次。
positionPopup(target, popup);
drawMask(target);
```

这段放在显示步骤的函数中。普通整页滚动期间，没有新的图形创建，也没有新的坐标计算。

和绝对定位一样，这里假设 html、body 不改变定位参照，且当前步骤中目标的文档位置与尺寸保持稳定。若某次操作展开了内容，在操作完成后再调用这两个函数；遮罩不会自动识别所有布局变化。

## 结束时清理

```ts
mask.remove();
popup.remove();
```

结束方法移除引导创建的两个根节点，其内部的文字、按钮和图形也随之离开文档。页面原有的目标区域与控件继续保留。

## 小练习

文档 W=1000、H=2400，目标 left=120、top=600、right=480、bottom=720：

1. 下方矩形的 y 和 height 是多少？
2. 左侧矩形的 width 和 height 是多少？
3. 为什么左右两块的高度不写整个文档高度？
4. 为什么蓝色边框既要 fill=none，也要 pointerEvents=none？

<details>
<summary>参考答案</summary>

1. y=720，height=1680。
2. width=120，height=120。
3. 否则会与上下区域重叠，半透明颜色叠加后出现深色条带。
4. fill=none 保证不把目标涂满；pointerEvents=none 保证边框不拦截鼠标。视觉和命中行为是不同属性。

</details>

继续阅读：[练习与自测](05-learning-checklist.md)。
