# 06 · 选中指定的单选项后，启用「次へ」

接着第 05 篇改。假设页面上有「配送」和「自取」两个单选项，这一步要求选「自取」。选「自取」时启用「次へ」，改选「配送」时立即禁用。

### 1. 在 `index.ts` 给规则加一个可选字段

在 `type Content` 里，把第 05 篇的 `rule` 那一行替换为：

```ts
rule?: 'click' | { checked: boolean; radioSelector?: string };
```

`radioSelector` 指定要检查哪个单选框。`?` 表示可以省略它，所以第 05 篇的复选框仍然写 `rule: { checked: true }`。

> **练习 1 · 修正一条单选规则**
>
> 下面是练习用的规则配置。
>
> ```ts
> const pickupRule: Content['rule'] = {
>   radioSelector: '#pickup',
>   checked: 'true',
> };
> ```
>
> **要求**　只改一个值，让这段代码通过 TypeScript 类型检查，并保持「必须选中自取」这个条件。

### 2. 在 `tips.ts` 加一个单选步骤

假设两个选项的共同容器是 `#delivery-method`，其中「自取」的 `<input type="radio">` 是 `#pickup`。同组的两个 radio 使用相同的 `name`。把下面这项放在 `content` 数组的结束步骤之前，两个示例选择器换成页面实际的选择器。

```ts
{
  selector: '#delivery-method',
  title: '受け取り方法',
  message: '店頭受け取りを選んでください。',
  rule: { radioSelector: '#pickup', checked: true },
},
```

- `selector` 指向**整组选项的容器**。你的 `updateMask(target)` 会给整组挖孔，这样用户能点两个选项，也能点它们的标签。
- `radioSelector` 指向**容器里要检查的那个 input**。`checked: true` 表示必须选中它才能继续。

> **练习 2 · 为支付方式配置引导**
>
> ```html
> <div id="payment-method">
>   <label><input type="radio" name="pay" id="card">银行卡</label>
>   <label><input type="radio" name="pay" id="cash">现金</label>
> </div>
> ```
>
> ```ts
> const paymentStep: Content = {
>   title: '支付方式',
>   message: '请选择现金。',
>   selector: '_____',
>   rule: { radioSelector: '_____', checked: true },
> };
> ```
>
> **要求**　根据 HTML 填写两处选择器。遮罩的孔要露出两个选项及其标签，只有选中「现金」时才能继续。

### 3. 在 `proceedToStep(index)` 替换复选框判断

找到第 05 篇最后那个带有 `target.type === 'checkbox'` 的 `if`，把这个 `if` 整块换成下面这段。插入位置仍在点击规则的 `if` 之后。

```ts
if (typeof rule === 'object' && target !== null) {
  const input = rule.radioSelector === undefined
    ? target
    : target.querySelector(rule.radioSelector);

  if (input instanceof HTMLInputElement &&
      (input.type === 'checkbox' || input.type === 'radio')) {
    const updateButton = (): void => {
      stepButton.disabled = input.checked !== rule.checked;
    };

    updateButton();
    target.addEventListener('change', updateButton, { signal });
  }
}
```

- `const input = ...` 找出要读状态的输入框。复选框步骤直接用原来的 `target`；单选步骤在容器里找配置的 `#pickup`。
- `input instanceof HTMLInputElement` 和后面的类型判断确认找到了复选框或单选框。找不到时，第 05 篇开头设置的禁用状态会保留。
- `input.checked !== rule.checked` 比较实际状态和要求。`updateButton()` 先执行一次，所以进入步骤时已经选好「自取」也能直接继续。
- **把 `change` 监听加在 `target` 上。** 从「自取」改选「配送」时，发出 `change` 的是刚选中的「配送」。事件会冒泡到共同容器，回调再读取「自取」的 `input.checked`，就能及时禁用按钮。`signal` 继续负责在切步或关闭时撤掉监听。

> **练习 3 · 修复切换选项后仍可继续的问题**
>
> 沿用练习 2。`target` 是支付方式容器，`input` 是「现金」单选框。下面的练习代码能在选中「现金」时启用按钮，但改选「银行卡」后，按钮仍然可用。
>
> ```ts
> const updateButton = (): void => {
>   stepButton.disabled = !input.checked;
> };
>
> updateButton();
> input.addEventListener('change', updateButton, { signal });
> ```
>
> **要求**　只改最后一行的监听对象。先点「现金」，再点「银行卡」，确认「次へ」先启用、再禁用。

### 4. 在页面上验证

1. 先选「配送」再进入这一步，「次へ」应禁用。
2. 点「自取」旁边的标签，「次へ」应启用。此时改选「配送」，按钮应立即禁用。
3. 先选好「自取」再进入这一步，「次へ」应一开始就可用。

> **练习 4 · 给反向规则补两个测试**
>
> 这次要求「自取」处于未选中状态。下面的函数返回「次へ」的 `disabled` 值。
>
> ```ts
> const rule = { radioSelector: '#pickup', checked: false };
>
> function isNextDisabled(pickupChecked: boolean): boolean {
>   return pickupChecked !== rule.checked;
> }
>
> console.assert(isNextDisabled(true) === _____, '自取已选中');
> console.assert(isNextDisabled(false) === _____, '自取未选中');
> ```
>
> **要求**　把两处空白分别换成 `true` 或 `false`，让两条断言都通过。注意断言检查的是按钮是否**禁用**。
