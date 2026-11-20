# 02 · fixed、absolute 与页面坐标

页面说明通常有两种位置关系：一种始终留在屏幕中的同一处；另一种放在某个页面区域旁边，与页面内容一起滚动。

本节讨论整页滚动。示例中 html、body 没有通过 position、transform 等样式改变提示框的定位参照，absolute 的坐标原点与文档原点一致。若提示框位于其他定位容器中，需要使用那个容器的坐标，不能直接套用页面坐标。

## 先分清两套坐标

| 数据 | 相对于哪里 | 整页向下滚动 100px 后 |
| --- | --- | --- |
| getBoundingClientRect().top | 视口上边缘 | 普通文档元素的值减少 100 |
| window.scrollY | 页面已滚动距离 | 增加 100 |
| rect.top + scrollY | 文档上边缘 | 布局不变时保持不变 |

例如目标的视口 bottom=240，页面已经滚动 500，那么它的文档 bottom=740。提示放在其下方，间距 12，top 就是 752px。

```ts
const rect = target.getBoundingClientRect();
const absoluteTop = rect.bottom + window.scrollY;
const absoluteLeft = rect.left + window.scrollX;

popup.style.position = 'absolute';
popup.style.top = absoluteTop + 12 + 'px';
popup.style.left = absoluteLeft + 'px';
```

这段假设 target 与 popup 已经取得，且 popup 已加入 body。赋值完成后，普通页面滚动不需要再调用定位函数。浏览器自然让提示框和页面内容一起移动。

## 一个完整的定位函数

使用顺序是：更新文字 → 把 popup 放入文档 → 调用 positionPopup。先设置影响尺寸的样式，再测量弹窗宽高。

```ts
const positionPopup = (
  target: HTMLElement | null,
  popup: HTMLDivElement
): void => {
  // 设置最大尺寸后再测量，避免内边距使外框超过限制。
  popup.style.boxSizing = 'border-box';
  popup.style.maxWidth = 'calc(100vw - 24px)';
  popup.style.maxHeight = 'calc(100vh - 24px)';
  popup.style.overflowY = 'auto';

  if (target === null) {
    // 开始和结束说明不依附页面元素，固定在屏幕中的同一位置。
    popup.style.position = 'fixed';
    popup.style.top = '30%';
    popup.style.left = '20%';
    popup.style.width = '';
    return;
  }

  popup.style.position = 'absolute';
  popup.style.width = 'max-content';
  const rect = target.getBoundingClientRect();
  const box = popup.getBoundingClientRect();
  const gap = 12;

  // 视口坐标加上页面滚动距离，得到文档坐标。
  let top = rect.bottom + window.scrollY + gap;
  // 初次显示时收住横向边界，避免提示框从右侧超出视口。
  let left = Math.max(gap, Math.min(
    rect.left, window.innerWidth - box.width - gap
  )) + window.scrollX;

  // 仅在显示这一条说明时选择方向：下方、上方、左侧。
  if (rect.bottom + gap + box.height > window.innerHeight) {
    if (rect.top >= box.height + gap) {
      top = rect.top + window.scrollY - box.height - gap;
    } else if (rect.left >= box.width + gap) {
      top = rect.top + window.scrollY;
      left = rect.left + window.scrollX - box.width - gap;
    }
  }

  popup.style.top = top + 'px';
  popup.style.left = left + 'px';
};
```

target=null 时，使用 fixed，表示这一条不依附页面区域；有目标时，使用 absolute。top:30%、left:20% 是示例位置，可按页面布局选择。

三个方向在调用时决定：

1. 优先目标下方。
2. 下方空间不足且上方足够时，放上方。
3. 上方也不够、左侧足够时，放左侧。

都放不下时保留下方位置，继续滚动查看。横向边界只在这次定位时处理，不会在滚动中把提示拽回视口。

## 怎样组织调用

```ts
// 进入一个说明步骤时，先更换文案。
title.textContent = step.content.title;
message.textContent = step.content.message;

// 需要把远处的目标带入视口时，可进行一次即时滚动。
target?.scrollIntoView({
  block: 'center',
  inline: 'nearest',
  behavior: 'instant',
});

// 即时滚动完成后，进行这一步的位置计算。
positionPopup(target, popup);
```

这里的调用片段放在步骤显示函数内。手动滚动不触发新的定位计算，也不需要 scroll 监听、定时器或动画循环。

## 布局变化与滚动不是同一件事

绝对定位保存的是一次测量得到的文档坐标。普通整页滚动不会改变这个坐标；控件展开、图片加载后挤动内容、窗口变窄引起换行，则可能改变目标在文档中的位置。

如果业务操作会改变布局，在那次操作完成后重新调用 positionPopup 和遮罩绘制函数即可。本练习只演示步骤切换时的布局，不自动处理所有异步重排。缩放窗口后可以重新进入步骤再观察。

## 阅读日志时保持变量名一致

```ts
const absoluteLeft = rect.left + window.scrollX;
popup.style.left = absoluteLeft + 'px';
console.log('横向位置', absoluteLeft);
```

日志读取的也应是已声明的 absoluteLeft。字符串标签可以任意命名，后面的变量则必须在当前作用域中存在。

## 小练习

1. rect.bottom 从 240 变成 140，scrollY 从 500 变成 600。提示的文档 top 应该改变吗？
2. 为什么 target=null 的说明采用 fixed？
3. absolute 提示放在 position:relative 的容器里，还能直接使用文档坐标吗？

<details>
<summary>参考答案</summary>

1. 不变，两次相加都是 740，加间距后仍为 752。滚动时重新给 top 赋值没有必要。
2. 它不依附某个页面区域，适合保持在固定屏幕位置。
3. 不能直接假定可以。它的包含块可能是这个容器，应转换成容器坐标，或让提示使用预期的页面定位参照。

</details>

继续阅读：[步骤状态与事件](03-events-and-steps.md)。
