# 04 · 点击高亮元素后，启用「次へ」

这一节改一个步骤。进入该步时，「次へ」变灰。点击高亮元素后，按钮恢复，再点「次へ」才进入下一步。下面沿用你截图里的 `start()`、`proceedToStep(index)` 和 `cleanUp()`。

### 1. 在 `index.ts` 声明点击规则

找到 `type Content`，在 `selector` 下面加一行。

```ts
rule?: 'click';
```

`?` 表示这项可以省略。写了 `'click'` 的步骤要求点击，省略它的步骤可以直接点「次へ」。

> **练习 1 · 让欢迎步骤通过类型检查**
>
> ```ts
> type PracticeStep = {
>   title: string;
>   rule: 'click';
> };
>
> const welcome: PracticeStep = { title: '欢迎' };
> ```
>
> **要求**　只给类型声明加一个符号，让欢迎步骤可以省略 `rule`，同时让 `rule: 'hover'` 仍然报错。

### 2. 在 `tips.ts` 给一个步骤加规则

找到提示用户点击高亮元素的那一项，在它已有的 `selector` 下面加一行。

```ts
rule: 'click' as const,
```

沿用这一步原来的 `selector` 指定点击目标。`as const` 让 TypeScript 保留精确的类型 `'click'`。你的 `tips` 是单独定义再导出的，普通对象属性里的字符串会被推断成 `string`，传给 `Content` 时类型太宽。

> **练习 2 · 给预览按钮配置点击规则**
>
> ```html
> <button id="preview">预览</button>
> ```
>
> ```ts
> const previewStep = {
>   title: '预览',
>   message: '请先点击预览按钮。',
>   selector: '_____',
>   // 在这里补上点击规则
> };
> const step: Content = previewStep;
> ```
>
> **要求**　根据 HTML 填入选择器，再补一行规则。保留最后一行赋值，确认 TypeScript 能把这份单独声明的配置当作 `Content` 使用。

### 3. 在 `index.ts` 按当前步骤控制按钮

先在 `start()` 内部的第一行加上：

```ts
const clickController = new AbortController();
```

它用来在用户中途关闭引导时撤掉页面元素上的点击监听。每次调用 `start()` 都会创建一个新的控制器。

然后找到 `proceedToStep(index)`。在已有的 `currentStep = index;` 下面插入这一段：

```ts
stepButton.disabled = step.rule === 'click';

if (step.rule === 'click' && target !== null) {
  target.addEventListener('click', () => {
    stepButton.disabled = false;
  }, { once: true, signal: clickController.signal });
}
```

- 第一行每次切步都会执行。有点击规则时，`disabled` 是 `true`，按钮禁用。普通步骤得到 `false`，按钮可用。
- `target` 就是你上面用 `step.selector` 查到的元素。只有点到这个元素或它里面的内容，才会执行回调。如果选择器没找到元素，这一步会保持禁用。
- 回调里的 `disabled = false` 只负责启用按钮。进入下一步仍由你原来的 `stepButton` 点击事件负责。
- `once: true` 表示点击一次后自动撤掉监听。`signal` 把这条监听关联到上面的控制器。

最后在已有的 `cleanUp()` 里加一行，放在 `box.remove();` 前面：

```ts
clickController.abort();
```

如果用户还没点目标就关闭引导，这一行会撤掉还在等待的监听。你的 `updateBox()` 照旧负责把遮罩和提示框放进页面。

> **练习 3 · 点击后只解锁按钮**
>
> ```ts
> // target 是已找到的预览按钮
> stepButton.disabled = true;
> target.addEventListener('click', () => {
>   proceedToStep(currentStep + 1);
> }, { once: true, signal: clickController.signal });
> ```
>
> **要求**　只改回调里的一行。点「预览」后应该仍停在当前步骤，同时「次へ」变为可用；再点「次へ」才切步。

### 4. 在页面上验证

1. 打开欢迎步骤，直接点「次へ」，应能继续。
2. 进入加了 `rule` 的步骤，「次へ」应禁用。改变窗口大小、点击遮罩，按钮都应保持禁用。
3. 点击高亮元素，「次へ」应启用，提示仍停留在当前步骤。再点「次へ」，应进入下一步。

> **练习 4 · 验证中途关闭时会撤掉监听**
>
> ```ts
> const preview = document.createElement('button');
> const controller = new AbortController();
> let clicks = 0;
> preview.addEventListener('click', () => { clicks += 1; }, {
>   signal: controller.signal,
> });
>
> // 在这里模拟关闭引导
> preview.click();
> console.assert(clicks === 0, '关闭后仍然响应了点击');
> ```
>
> **要求**　在注释处补一行，让断言通过。然后注释掉补上的那行再运行，确认断言能检测到遗漏清理的情况。
