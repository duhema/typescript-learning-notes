# 吹き出し（气泡）四向定位：换边、尖角对齐和小窗口缩放

这篇教程从截图里的 `index.ts` 开始，只改 `start()` 内的代码。完成后，气泡平时宽 **280px**。优先放在目标下方，下方空间不足再试上方、右方，最后放左方。靠近窗口边缘时，气泡可以平移，尖角指向目标对应边的中点。全部使用浏览器原生 API。

按下面四步做。只有 `updateBox()` 要整段替换；`tips.ts`、遮罩开关、按钮和步骤配置都沿用现有实现。行号以截图为准，粘贴时用函数名定位。

## 第一步　在 createBubble() 里加小窗口缩放

找到 `bubbleContent.style.overflowY = 'auto';`（原第 83 行），在它**下面**加：

```ts
if (window.matchMedia('(max-width: 304px), (max-height: 280px)').matches) {
  bubbleContent.style.setProperty('zoom', '0.875');
}
```

窗口宽度 ≤ 304px **或**高度 ≤ 280px 时，内容层里的标题、正文和按钮一起缩到 87.5%。截图里的 `bubble.style.position = 'absolute'`、内容层已有的换行和滚动样式，以及独立的三角形 `arrow` 都继续使用。三角形原来的高度是 8px，下一步会用到这个数。

## 第二步　整段替换 updateBox()

在 `start()` 里找到原第 244～247 行的 `const updateBox = ...`。从这行到函数结尾的 `};`，整段替换为：

```ts
const updateBox = (mask: SVGSVGElement, target: HTMLElement | null): void => {
  box.replaceChildren(mask);

  if (target === null) {
    box.append(createPopup());
    return;
  }

  const { bubble, arrow }: {
    bubble: HTMLDivElement,
    arrow: HTMLSpanElement,
  } = createBubble();

  const root: HTMLElement = document.documentElement;
  const tip: number = 8;

  bubble.style.width = 'min(280px, calc(100vw - 24px))';
  box.append(bubble);

  const rect: DOMRect = target.getBoundingClientRect();
  const size: DOMRect = bubble.getBoundingClientRect();
  const centerX: number = rect.left + rect.width / 2;
  const centerY: number = rect.top + rect.height / 2;

  const below: number = root.clientHeight - rect.bottom - tip;
  const above: number = rect.top - tip;
  const right: number = root.clientWidth - rect.right - tip;

  let x: number = Math.max(0, Math.min(centerX - size.width / 2, root.clientWidth - size.width));
  let y: number = Math.max(0, Math.min(centerY - size.height / 2, root.clientHeight - size.height));
  let angle: number;

  if (below >= size.height) {
    y = rect.bottom + tip;
    angle = 0;
  } else if (above >= size.height) {
    y = rect.top - size.height - tip;
    angle = 180;
  } else if (right >= size.width) {
    x = rect.right + tip;
    angle = -90;
  } else {
    x = rect.left - size.width - tip;
    angle = 90;
  }

  bubble.style.left = `${x + window.scrollX}px`;
  bubble.style.top = `${y + window.scrollY}px`;

  if (angle === 0 || angle === 180) {
    arrow.style.left = `${centerX - x}px`;
    arrow.style.top = angle === 0 ? '0' : '100%';
  } else {
    arrow.style.left = angle === -90 ? '0' : '100%';
    arrow.style.top = `${centerY - y}px`;
  }

  arrow.style.transformOrigin = '50% 100%';
  arrow.style.transform = `translate(-50%, -100%) rotate(${angle}deg)`;
  arrow.style.display = 'block';
};
```

这里有两个顺序值得记住：

- **先挂载，再测量。** `box.append(bubble)` 之后才读 `size.height`，读到的是这一步文案在实际宽度下换行后的高度。
- **先定位气泡，再定位尖角。** 上下放置时，`centerX - x` 是目标中点距气泡左边的距离；左右放置时，用 `centerY - y` 算它距气泡上边的距离。只有气泡的绝对定位要加页面滚动距离。

## 第三步　修改两个调用位置

分别找到 **`handleResize()`**（原第 256～262 行）和 **`proceedToStep()`**（原第 287～298 行）。这两个函数各改一次，操作完全一样：

1. 删除这行：`const updatedOverlay: HTMLDivElement = updateOverlayPosition(target);`。
2. 找到 `updateBox(updatedMask, updatedOverlay);`，把第二个实参改为 `target`。

改完后，两个函数的**渲染结尾**都是下面这样。保留各自前面的取步骤、查目标等代码：

```ts
const updatedMask: SVGSVGElement = updateMask(target);

if (step.maskEnable) {
  updatedMask.style.display = 'block';
}
updateBox(updatedMask, target);
```

如果已经照旧笔记在 `updateMask(target)` **前面**加过 `box.replaceChildren();` 来处理旧遮罩撑宽页面，可以保留。上面的渲染片段从 `const updatedMask` 开始替换即可。

## 第四步　删除旧定位方法，检查接线

删除原第 204～237 行的整个 `const updateOverlayPosition = ...`。原来由它选择气泡或居中弹窗，现在由 `updateBox()` 根据同一个 `target` 来选择。

在 `index.ts` 搜索确认：

- `updateOverlayPosition` 和 `updatedOverlay` 均应为 **0 处**，之前留下的相关注释也可以删除。
- `updateBox(updatedMask, target)` 应为 **2 处**，切步和调整窗口大小各一处。
- 原来的 `window.addEventListener('resize', handleResize)` 仍在。缩放时它会重建气泡并重新测量，所以小窗口缩字后，把窗口拉大也会恢复正常大小。

## 怎么验收

打开有引导的页面。先确认没有目标的第一步仍显示居中弹窗，然后点进有目标的气泡步骤。可以通过调整窗口宽高、滚动到合适位置后稍微拖动窗口边缘，触发重新定位。

| 怎么试 | 应该看到什么 |
| --- | --- |
| 目标下方留足空间 | 气泡在下方，尖角朝上，对准目标下边的中点 |
| 让目标靠近窗口底部，上方留足空间 | 气泡移到上方，尖角朝下，对准目标上边的中点 |
| 把窗口拉矮，让上下都放不下，目标靠左，右侧留足空间 | 气泡移到右侧，尖角朝左，对准目标右边的中点 |
| 保持上下都放不下，让目标靠右，左侧留足空间 | 气泡移到左侧，尖角朝右，对准目标左边的中点 |
| 目标靠近右边缘，但下方仍放得下 | 气泡仍在下方，整体向左平移，尖角在气泡上的位置随之偏右 |
| 把窗口调到 240px 宽，再恢复宽窗口 | 小窗口的气泡宽约 216px，文字和按钮也变小；恢复后气泡宽 280px，字号恢复 |

左侧是约定的最后一个方向。如果测试时四边都过窄，气泡可能部分超出左侧视口。最后点到「終了」或中途关闭，再调整窗口大小，引导应保持关闭。
