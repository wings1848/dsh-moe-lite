# 与其它皮肤共存吗？

**结论：不要和其它“整体皮肤”同时启用。** 一套皮肤会重写 DSH 的设计令牌并装饰同一批 DOM；
两套一起开，得到的不是叠加效果，而是混合配色、双重立绘和错位的侧栏排版。

本页具体回答上游 issue
[KurosawaGeeker/dsh-moe-lite#8](https://github.com/KurosawaGeeker/dsh-moe-lite/issues/8)：
`dsh-moe-clean` 和 `@smalltailqwq/dsh-client-ui-skin-maid-atelier`（深海女仆工坊）一起用会发生什么。

## 与 maid-atelier 的具体冲突

以 maid-atelier `0.1.11` 为准（`npm view @smalltailqwq/dsh-client-ui-skin-maid-atelier version`）。

### 1. 同一组设计令牌，两套值

两套皮肤都写 `--dsw-alias-*` 和 `--dsw-specific-*`：

- maid-atelier 写在 `body[data-dsh-maid-atelier]` 作用域下；
- `dsh-moe-clean` 写在 `html body` 上。

按 CSS 优先级，maid-atelier 的令牌会赢（它的作用域选择器更具体），但 `dsh-moe-clean`
对背景、侧栏高度等属性带了 `!important`，这些又会压过 maid-atelier 的普通声明。
结果不是“谁的皮肤生效”，而是逐条属性各赢一部分：深蓝蕾丝面板配白蓝渐变底、
金边 logo 框里换成 44px crest、缎带新建按钮同时带着两套 padding / border 规则，
是最常见的混合态。

### 2. 装饰同一批侧栏节点

| 节点 | `dsh-moe-clean` | maid-atelier |
| --- | --- | --- |
| `[class*="logoRow"]` | `height: 56px !important`，字标绝对居中 | 60px 高、金边装饰框、`margin: 8px` |
| `button[class*="brand"]` | 隐藏原生 identity，插入 44px stacked crest | 给 182px 原生字标上金色、加投影（仅 button 分支；macOS 的 span 品牌吃不到） |
| `button[class*="newSession"]` | `height: 34px !important` | 58px 高、缎带背景、衬线字体 |
| `[class*="panelRow"]` | 32px 行高、2px 间距 | 自己的导航行与侧栏配色 |
| 右侧/背景立绘 | 固定定位的女仆 + 侧栏底纹 | `[data-skin-chrome]` 角色舞台 + 宫殿背景 |

两边都靠类名前缀匹配，所以冲突不是“找不到节点”，而是“都找到了、都改了”。
`height` 与 `min-height` 的胜负也要分开看：`dsh-moe-clean` 的
`height: 56px / 34px !important` 不会把 maid-atelier 的
`min-height: 60px / 58px` 压小，最终高度仍取较大值；冲突主要体现在
padding、border-image、背景、字体和定位上，观感是两套装饰叠在一起，而不是单纯变矮。

### 3. 双重立绘

两套都会挂页面级人物层：`dsh-moe-clean` 是 `.dsh-moe-maid` / `.dsh-moe-whale`，
maid-atelier 是 `[data-skin-chrome=character-stage]` 与 `[data-maid-character]`。
同时启用会看到至少三位女仆、两层背景，窄窗口下互相遮挡。

### 4. DSH 版本下限不同

maid-atelier `0.1.11` 的 `peerDependencies` 是：

```text
@deepseek-ai/dsh >=0.2.2-rc.3 <0.2.3-0
```

`dsh-moe-clean` 的实测矩阵见 README 的兼容性表（当前到 `0.2.2-rc.1`）。
`peerDependenciesMeta` 里 `@deepseek-ai/dsh` 被标了 `optional: true`，但 DSH 的
`evaluatePluginCompatibility()`（`packages/boot/app-boot/src/plugin-compatibility.ts`）
不读 `optional`：在 `0.2.2-rc.1` 上它仍会把 maid-atelier 判为与运行时
`>=0.2.2-rc.3` 不匹配，安装/重启需要 `dsh plugin allow-version` 做精确版本豁免。
也就是说，rc.1 上两者根本轮不到“样式打架”，maid-atelier 自己就会被门禁拦下。

### 5. 卸载/切换的顺序依赖（推断，未实测）

两套插件都靠 effect 销毁器还原 DOM/CSS。按实现推断，同时启用后先卸载谁会影响还原结果：
皮肤 A 卸载时把共享节点恢复成“原生”，皮肤 B 仍以为自己接管着它；之后再卸载 B 才是最终态。
这不是数据损坏，但没有做过真实浏览器实测，只能作为“切换后可能需要再刷新一次”的风险提示。

## 怎么做

1. **同时只启用一套**。用「设置 → 插件」禁用另一套，或按各自文档的皮肤切换方式切换，然后重启一次 DSH。
2. maid-atelier 的 `cordis.patch.yml` 注释明确说皮肤之间由 `scripts/dsh-skin`（home 层禁用行）互斥管理；
   本主题没有接入那套皮肤管理器，不要假设它能被 maid-atelier 的“皮肤管理”开关接管。
3. 只想要双方的美术、不要双方布局：需要自己写补丁层逐条覆盖，属于定制，不属于本主题的支持范围。

如果已经同时启用并出现排版错乱：禁用其中一套插件 → 重启 DSH → 刷新页面即可恢复，
不需要重装。两套的 `apply()` 都有销毁器，不会留下永久性的 DOM 改动。

## 复核命令

```sh
# 两套皮肤声明的 DSH 版本范围
npm view @smalltailqwq/dsh-client-ui-skin-maid-atelier@0.1.11 peerDependencies

# 拉取 maid-atelier 并核对它改写的选择器
npm pack @smalltailqwq/dsh-client-ui-skin-maid-atelier@0.1.11
tar -xzf smalltailqwq-dsh-client-ui-skin-maid-atelier-0.1.11.tgz
grep -o 'body\[data-dsh-maid-atelier\][^{}]*logoRow[^{}]*{[^}]*}' package/lib/client.js
grep -o 'body\[data-dsh-maid-atelier\][^{}]*newSession[^{}]*{[^}]*}' package/lib/client.js
```

本仓库侧的对应规则在 `lib/client.js` 的 `CSS` 常量里（`logoRow`、`newSession`、
`panelRow`、`brand` 几段），可以用上面同样的 grep 方式核对。
