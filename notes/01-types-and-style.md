# 01 · DOM 元素、类型与样式

浏览器中的元素既有结构，也有样式。TypeScript 可以检查能否访问某个属性；浏览器还会判断赋给样式的值是否有效。两层检查解决的是不同问题。

## 创建按钮与文字节点

```ts
const closeButton: HTMLButtonElement = document.createElement('button');
closeButton.type = 'button';
closeButton.textContent = '关闭';
closeButton.style.padding = '10px 16px';

const title: HTMLParagraphElement = document.createElement('p');
title.textContent = '操作说明';
title.style.fontSize = '20px';
```

这里的 createElement 只创建对象。要显示出来，还需要把它们加入页面或已经连接到页面的容器。

```ts
const popup: HTMLDivElement = document.createElement('div');
popup.append(closeButton, title);
document.body.append(popup);
```

这两段可接在同一个练习文件中运行。按钮 type=button 表示普通按钮，避免它进入表单后成为默认提交按钮。

## SVG 为什么使用 createElementNS

HTML 与 SVG 使用不同的命名空间。创建 SVG 根元素时，可以这样写：

```ts
const svgNS = 'http://www.w3.org/2000/svg';
const svg: SVGSVGElement = document.createElementNS(svgNS, 'svg');

svg.style.position = 'fixed';
svg.style.left = '0';
svg.style.top = '0';
svg.style.width = '100%';
svg.style.height = '100%';
```

SVGSVGElement 支持 style。若把变量标成更宽泛的 Element，TypeScript 不会允许访问这个类型未声明的属性。

```ts
const element: Element =
  document.createElementNS('http://www.w3.org/2000/svg', 'svg');

// element.style.position = 'fixed';
// 取消上行注释会出现类型错误；对象本身没有因此变成另一种元素。
```

不要为了避开检查而直接加 as any。先看元素的创建方式，以及函数返回类型有没有过于宽泛。

## 样式和图形属性分开理解

| 写法 | 用途 |
| --- | --- |
| svg.style.position | 定位方式 |
| svg.style.zIndex | 图层顺序 |
| rect.setAttribute('width', '100%') | SVG 矩形宽度 |
| rect.setAttribute('fill', 'rgba(0, 0, 0, 0.5)') | SVG 矩形填充颜色 |

SVG 根是容器，里面的矩形才负责画灰色区域。给根元素设置 position，并不会自动画出背景。

命名空间字符串是固定标识，不是要下载的资源地址。变量名叫 mask，也不表示节点一定是 SVG 的 `<mask>`；变量名和标签名是两回事。

## 字符串通过类型检查，样式仍可能无效

```ts
// 错误值示例：padding 不能为负数，浏览器会忽略这条声明。
closeButton.style.padding = '-8px 12px';

// 有效值：上下 8px，左右 12px。
closeButton.style.padding = '8px 12px';
```

第一行不一定触发 TypeScript 错误，因为属性接受字符串。但 CSS 有自己的有效值规则。排查外观时，要同时看代码和浏览器的计算样式。

## 节点引用不是副本

```ts
const newPopup = popup;
console.log(newPopup === popup); // true：两个变量引用同一个对象。
```

给变量另起名字不会生成新节点。remove 把节点移出文档，变量仍可引用它；append 同一个节点时，移动的是这个对象。

## 小练习

1. 创建一个按钮，设置文字但不 append，页面上会出现它吗？
2. `const newPopup = popup` 与 `document.createElement('div')` 的区别是什么？
3. 把 SVG 标成 HTMLElement，是不是正确的类型修复方法？

<details>
<summary>参考答案</summary>

1. 不会，节点还没连接到文档。
2. 前者增加一个引用；后者创建新的 div 对象。
3. 不是。HTML 与 SVG 属于不同的元素类型，应确认命名空间和实际元素类型。

</details>

继续阅读：[定位与坐标](02-dom-and-position.md)。
