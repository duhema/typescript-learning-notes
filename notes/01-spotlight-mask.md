# 01 · 用 SVG 路径实现 Spotlight 挖孔遮罩

效果：页面外侧变暗并拦截鼠标，目标区域保持可见、可点击、可输入。没有目标时，整页遮盖；切换目标时移动孔洞。

## 1. 先理解 outer 和 inner

一个 SVG `path` 可以包含多个闭合轮廓：

- `outer`：整个页面的外轮廓。
- `inner`：目标区域的内轮廓。
- `fillRule = 'evenodd'`：外轮廓内填色，两层轮廓重叠的内部留空。

因此，真正遮盖页面的只有一个 `spotlight` 路径。`outer` 和 `inner` 是两段路径字符串，不是两层 DOM 遮罩。另一个 `border` 路径只画蓝色边框，不填色、不接收点击。

四块矩形本身也能实现遮罩；这里改用复合路径，让“整页覆盖、目标挖孔”直接由一个形状表达。SVG 的复合路径与填充规则支持这种用法。[SVG 路径规范](https://www.w3.org/TR/SVG2/paths.html)、[填充规则](https://www.w3.org/TR/SVG11/painting.html#FillRuleProperty)。

**透明不一定意味着可以点穿。** 本节让孔洞不属于路径的填充区域，再用 `pointerEvents = 'fill'` 只拦截外侧。单纯给全屏矩形套透明度遮罩，透明位置仍可能接收点击。[SVG 命中测试规则](https://www.w3.org/TR/SVG2/interact.html#PointerEventsProperty)。

## 2. 删除旧矩形绘制代码

在 `start()` 内：

1. 删除整个 `createRect()`。
2. 用下一节替换整个 `createMask()`。
3. 用第 4 节替换整个 `updateMask()`，包括其中的 `addRect()`、四次外围绘制和旧边框代码。

不保留 `mask.replaceChildren()` 或 `svg.append(createRect())`。新结构只创建一次，之后修改路径数据。

`createPopup()`、步骤收集循环和 `updateBubblePosition()` 继续使用已有实现。

## 3. 创建一个 Spotlight 路径

位置：`start()` 内原来的 `createMask()` 处。已有的 `svgNS` 保留一份。

```ts
const createMask = (): {
  mask: SVGSVGElement;
  spotlight: SVGPathElement;
  border: SVGPathElement;
} => {
  const mask = document.createElementNS(svgNS, 'svg');
  mask.style.position = 'absolute';
  mask.style.left = '0';
  mask.style.top = '0';
  mask.style.width = '0';
  mask.style.height = '0';
  mask.style.zIndex = '9998';
  mask.style.pointerEvents = 'none';

  const spotlight = document.createElementNS(svgNS, 'path');
  spotlight.style.fill = 'rgba(0, 0, 0, 0.5)';
  spotlight.style.fillRule = 'evenodd';
  spotlight.style.pointerEvents = 'fill';

  const border = document.createElementNS(svgNS, 'path');
  border.style.fill = 'none';
  border.style.stroke = '#2563eb';
  border.style.strokeWidth = '2';
  border.style.pointerEvents = 'none';

  mask.append(spotlight, border);
  return { mask, spotlight, border };
};
```

`mask` 是 SVG 容器，`spotlight` 是有孔的遮盖形状。容器设置 `none`，让空白区域不挡鼠标；子路径明确设置 `fill`，让外侧继续拦截。边框设置 `none`，不会压住目标边缘的控件。

返回三个引用的写法与 `createPopup()` 返回多个元素相同，后面的更新函数可以直接修改它们，不需要反复查询 DOM。

本节固定使用一套约定：颜色、定位、尺寸和交互样式全部用 `.style`；SVG 路径数据 `d` 全部用 `setAttribute('d', ...)`。`d` 表示绘制指令，不是颜色或尺寸设置，同一个属性不会两种写法混用。这样也不依赖 CSS `d: path(...)` 的浏览器支持。

## 4. 用整页轮廓减去目标轮廓

位置：原来的 `updateMask()`，整段替换：

```ts
const updateMask = (target: HTMLElement | null): void => {
  // 测量前隐藏旧 SVG，避免旧尺寸撑大页面的测量结果。
  mask.style.display = 'none';

  const root = document.documentElement;
  const width = Math.max(root.scrollWidth, root.clientWidth);
  const height = Math.max(root.scrollHeight, root.clientHeight);

  mask.style.width = `${width}px`;
  mask.style.height = `${height}px`;

  // 页面外轮廓：左上 → 右上 → 右下 → 左下 → 闭合。
  const outer = `M 0 0 H ${width} V ${height} H 0 Z`;
  let inner = '';

  if (target !== null) {
    const rect = target.getBoundingClientRect();

    // rect 是视口坐标；加滚动距离，换成文档坐标。
    // 限制在外轮廓内，避免超出页面的目标产生多余填色。
    const left = Math.max(0, Math.min(width, rect.left + window.scrollX));
    const top = Math.max(0, Math.min(height, rect.top + window.scrollY));
    const right = Math.max(left, Math.min(width, rect.right + window.scrollX));
    const bottom = Math.max(top, Math.min(height, rect.bottom + window.scrollY));

    if (right > left && bottom > top) {
      inner = `M ${left} ${top} H ${right} V ${bottom} H ${left} Z`;
    }
  }

  spotlight.setAttribute('d', `${outer} ${inner}`);
  border.setAttribute('d', inner);
  mask.style.display = 'block';
};
```

先理解这四个路径命令：

| 命令 | 意思 |
| --- | --- |
| `M x y` | 把画笔移到这个坐标，开始新轮廓 |
| `H x` | 水平画到指定 x |
| `V y` | 垂直画到指定 y |
| `Z` | 连回起点，闭合轮廓 |

`outer` 和 `inner` 都是闭合矩形，但它们写在**同一个 path 的 d** 里面。`evenodd` 负责决定哪些区域填色，不需要计算上、下、左、右四块遮盖物。

没有目标时，inner 是空字符串：spotlight 只剩整页轮廓，border 的空路径也不会留下旧边框。目标没有可见尺寸时，同样保持整页遮盖。

遮罩一直使用 `absolute`，尺寸取整份文档，孔洞也使用文档坐标。普通整页滚动时，遮罩和目标自然一起移动，不需要滚动监听。这里的 `absolute` 以文档原点为参照，SVG 直接放在 body 下；如果 body 或祖先另有定位、transform 等样式，需要按实际定位参照换算，不能直接套用文档坐标。

## 5. 接上元素创建与步骤切换

位置：`start()` 底部“组装元素”的地方。将原来的单个 mask 创建和两次 append 替换为：

```ts
const { mask, spotlight, border } = createMask();
document.body.append(mask, popup);
```

`createMask()` 现在返回的是对象，不能继续写 `let mask: SVGSVGElement = createMask()`。

这两行必须位于首次 `proceedToStep(currentStep)` 之前。虽然 updateMask 的定义在前面，但执行它时，这三个引用必须已经创建完成。

`proceedToStep()` 末尾保持下面顺序，不要再加一套绘制调用：

```ts
updateBubblePosition(target, popup);
updateMask(target);
```

先摆放弹窗，再测量文档尺寸，可以将弹窗撑出的页面范围也纳入遮盖。弹窗的 `zIndex = '9999'` 高于遮罩，关闭和下一步仍能操作。

## 6. 窗口变化后重新测量

缩放窗口可能改变页面尺寸和目标位置。把以下代码插到 `let currentStep = 0;` **之后**、首次 `proceedToStep(currentStep);` **之前**：

```ts
const handleResize = (): void => {
  const step = steps[currentStep];
  if (!step) return;

  const selector = step.content.selector;
  const target = selector === null
    ? null
    : document.querySelector<HTMLElement>(selector);

  updateBubblePosition(target, popup);
  updateMask(target);
};

window.addEventListener('resize', handleResize);
```

这里读取的是 `step.content.selector`，与步骤数据结构一致。每次都查当前步骤的目标，不会继续使用上一条的孔洞。

将已有的 `cleanUp()` 替换为：

```ts
const cleanUp = (): void => {
  window.removeEventListener('resize', handleResize);
  mask.remove();
  popup.remove();
};
```

cleanUp 的定义可以留在原处；首次调用之前，handleResize 必须已初始化。上述插入顺序满足这一点。下一步与关闭按钮原来的监听继续使用。

若页面通过展开面板、加载内容等方式改变布局，在那次布局更新完成后调用 `handleResize()`，复用同一套测量逻辑。普通页面滚动不调用它。本节处理鼠标命中与视觉遮盖，不会自动限制键盘 Tab 的焦点范围。

## 7. 按这些操作验证

| 操作 | 应观察到的结果 |
| --- | --- |
| 显示没有目标的说明 | 整页变暗，没有旧孔洞与蓝框 |
| 进入有目标的步骤 | 目标范围透明，外侧变暗 |
| 点击孔洞里的按钮、输入文字 | 原页面控件正常响应 |
| 点击孔洞外的原页面按钮 | 按钮不响应 |
| 切换下一条目标 | 旧孔洞关闭，新目标处挖孔 |
| 上下、左右滚动页面 | 孔洞与普通文档中的目标一起移动 |
| 调整窗口大小 | 重新测量页面与孔洞，外侧继续遮盖 |
| 关闭引导后调整窗口 | 遮罩不再出现，页面正常操作 |

还可以在开发者工具中检查 SVG：始终只有两个 path，分别负责遮盖和边框，没有四个外围 rect。切步时这两个节点保持不变，只更新 d。
