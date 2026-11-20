# TypeScript 与 DOM 学习笔记

## 学习资料

- [TypeScript 快速上手（PDF）](materials/TypeScript%20快速上手.pdf)
- [装饰器（Markdown）](materials/装饰器.md)
- [装饰器（PDF）](materials/装饰器.pdf)

## DOM 实践

- [01 · 用 SVG 路径实现 Spotlight 挖孔遮罩](notes/01-spotlight-mask.md)：用 outer、inner 轮廓创建单个镂空遮罩，并让目标区域保持可操作。
- [02 · 理解步骤状态，为 Bubble 添加指向目标的尖角](notes/02-step-state-and-bubble-arrow.md)：确认局部步骤状态，通过内容层滚动与独立尖角保持指向。
- [03 · 将当前步骤封装在 proceedToStep() 内](notes/03-step-local-state.md)：使用 index 参数与本步回调，切步时统一清理监听，去掉外层 currentStep。
- [04 · 点击高亮元素后，启用「次へ」](notes/04-click-rule.md)
- [05 · 勾选复选框后，启用「次へ」](notes/05-checkbox-rule.md)
- [06 · 选中指定的单选项后，启用「次へ」](notes/06-radio-rule.md)

## 代码复盘

- [吹き出し（气泡）四向定位：换边、尖角对齐和小窗口缩放](notes/review-step-display.md)
- [简化气泡定位，把尖角固定在气泡边线中间](notes/centered-bubble-arrow.md)
