# 05 · 勾选复选框后，启用「次へ」

接着第 04 篇改。进入复选框步骤时，先看它有没有勾选。勾上就启用「次へ」，取消勾选就再次禁用。

### 1. 在 `index.ts` 扩充 `Content` 的 `rule`

把第 04 篇加的 `rule?: 'click';` 替换为这一行。

```ts
rule?: 'click' | { checked: boolean };
```

`|` 表示两种写法都可以。原来的点击步骤继续用 `'click'`。复选框步骤用 `{ checked: true }` 表示要求勾选，用 `{ checked: false }` 表示要求取消勾选。`?` 仍然允许普通步骤省略规则。

### 2. 在 `tips.ts` 的 `content` 数组里加一个复选框步骤

放在结束步骤之前。示例里的 `#agree` 要换成你页面上那个 `<input type="checkbox">` 的选择器，直接指向复选框本身。

```ts
{
  selector: '#agree',
  title: '確認',
  message: 'チェックを入れてください。',
  rule: { checked: true },
},
```

`rule.checked` 是**要求的状态**。`input.checked` 是**页面上实际的状态**。两者相等，就允许继续。

> **练习 1 · 配置一个要求取消勾选的步骤**
>
> ```html
> <label><input id="subscribe" type="checkbox" checked>订阅通知</label>
> ```
>
> ```ts
> const unsubscribeStep: Content = {
>   selector: '_____',
>   title: '通知设置',
>   message: '请取消订阅通知。',
>   rule: { checked: _____ },
> };
> ```
>
> **要求**　补全选择器和布尔值。使用正文的按钮逻辑时，进入步骤应当禁用「次へ」，取消勾选后才允许继续。

### 3. 在 `index.ts` 让监听随步骤结束

第 04 篇的点击只需监听一次。复选框可以来回勾选，所以要一直监听到离开这一步。

在 `start()` 顶部，把第 04 篇的 `const clickController = new AbortController();` 替换为：

```ts
let ruleController: AbortController | undefined;
```

这个变量保存当前步骤的控制器。刚开始还没有步骤，所以它可以是 `undefined`。

在 `cleanUp()` 里，把 `clickController.abort();` 替换为：

```ts
ruleController?.abort();
```

这样关闭引导时会撤掉当前步骤的监听。`?.` 表示已经创建了控制器才调用 `abort()`。

> **练习 2 · 切步时清理旧监听**
>
> ```ts
> let ruleController: AbortController | undefined;
>
> function beginStep(): AbortSignal {
>   // 在这里结束上一步的监听
>   ruleController = new AbortController();
>   return ruleController.signal;
> }
>
> const first = beginStep();
> const second = beginStep();
> console.assert(first.aborted && !second.aborted);
> ```
>
> **要求**　在注释处补一行。第一次进入时可以直接创建控制器；第二次进入时，旧信号应已中止，新信号应仍然有效。

### 4. 在 `proceedToStep(index)` 替换第 04 篇的按钮判断

找到 `currentStep = index;`。保留这行，把下面第 04 篇加的 `stepButton.disabled = ...` 和紧跟的 `if` 整段替换为：

```ts
ruleController?.abort();
ruleController = new AbortController();
const signal = ruleController.signal;
const rule = step.rule;

stepButton.disabled = rule !== undefined;

if (rule === 'click' && target !== null) {
  target.addEventListener('click', () => {
    stepButton.disabled = false;
  }, { once: true, signal });
}

if (typeof rule === 'object' && target instanceof HTMLInputElement && target.type === 'checkbox') {
  const updateButton = (): void => {
    stepButton.disabled = target.checked !== rule.checked;
  };

  updateButton();
  target.addEventListener('change', updateButton, { signal });
}
```

- 开头先撤掉上一步的监听，再给这一步创建控制器。这样上一步的复选框就无法改变当前步骤的按钮。
- `stepButton.disabled = rule !== undefined` 让普通步骤直接可用，有规则的步骤先禁用。点击分支沿用第 04 篇的行为。
- `typeof rule === 'object'` 选中形如 `{ checked: true }` 的规则。后面两个判断确认选到的是复选框。如果选择器写错，这一步会保持禁用。
- `target.checked !== rule.checked` 是“实际状态与要求不同”。不同就禁用，相同就启用。
- `updateButton()` 在进入步骤时立即检查一次，所以已经勾选的复选框也能正确显示按钮状态。
- `change` 会在用户勾选或取消勾选时触发。这里要响应每一次变化，因此监听持续到切步或关闭。

> **练习 3 · 修复「取消勾选后按钮仍然可用」**
>
> ```ts
> // target 是复选框，rule.checked 是本步要求的状态
> const updateButton = (): void => {
>   stepButton.disabled = target.checked !== rule.checked;
> };
>
> updateButton();
> target.addEventListener('change', updateButton, { once: true, signal });
> ```
>
> **要求**　只改最后一行的监听选项，保留切步时撤掉监听的能力。用 `checked: true` 的步骤连续试「勾选 → 取消 → 再勾选」，按钮应依次启用、禁用、启用。

### 5. 在页面上验证

1. 让复选框初始未勾选，进入这一步。「次へ」应禁用。勾上后应启用，取消勾选后应再次禁用。
2. 先勾好复选框，再进入这一步。「次へ」应立即可用。
3. 把配置改成 `rule: { checked: false }` 再试一次。这次未勾选时可以继续，勾上后按钮禁用。

> **练习 4 · 检查初始状态的四种组合**
>
> ```ts
> function isNextDisabled(actual: boolean, expected: boolean): boolean {
>   // 补上 return 语句
> }
>
> console.assert(isNextDisabled(true, true) === false);
> console.assert(isNextDisabled(false, true) === true);
> ```
>
> **要求**　补全函数，再为 `expected === false` 的情况写两条断言，分别覆盖初始已勾选和初始未勾选。四条断言都应通过。
