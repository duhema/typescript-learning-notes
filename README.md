# TypeScript 与 DOM 学习笔记

## 学习资料

- [TypeScript 快速上手（PDF）](materials/TypeScript%20快速上手.pdf)
- [装饰器（Markdown）](materials/装饰器.md)
- [装饰器（PDF）](materials/装饰器.pdf)

## 浏览器界面基础

通过按钮、提示框与 SVG 练习 TypeScript 在浏览器中的用法。每篇包含知识说明、分段示例、运行观察和小练习，说明与代码放在同一处。

| 顺序 | 内容 | 学习重点 |
| --- | --- | --- |
| 1 | [DOM 元素、类型与样式](notes/01-types-and-style.md) | 创建节点、SVG 命名空间、style 与引用 |
| 2 | [fixed、absolute 与页面坐标](notes/02-dom-and-position.md) | 文档坐标换算、目标旁定位、整页滚动 |
| 3 | [步骤状态与事件](notes/03-events-and-steps.md) | 数组、null、初始化顺序、下一步与结束 |
| 4 | [SVG 遮罩与操作区域](notes/04-svg-and-interaction.md) | 四块矩形、强调边框、鼠标命中 |
| 5 | [练习与自测](notes/05-learning-checklist.md) | 坐标计算、状态边界、运行观察 |


## 练习方法

按 01—04 的顺序学习，每次理解一个知识点、完成一处调整，再用浏览器验证效果。标注为片段的代码需要相应变量和上下文；同名函数只保留一份实现。最后使用 05 的练习与自测检查理解。

定位练习使用整页滚动和普通文档定位参照。提示框使用 absolute，在进入步骤时计算文档坐标；手动滚动由浏览器处理，不注册滚动定位监听。SVG 遮罩在目标四周绘制矩形，让原有控件保持可操作。

示例选择器代表页面中已经存在的区域，按练习页面的 DOM 设置即可。代码注释与讲解使用中文，小练习附可展开的参考答案。所有笔记代码均保存在 Markdown 中。
