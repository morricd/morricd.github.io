---
title: UniApp nvue 与 vue 的使用与区别
author: morric
date: 2020-06-18 22:55:00 +0800
categories: [开发记录, Hybird]
tags: [Hybird, vue, uniapp]
---

uni-app 中存在两种页面类型：`.vue` 和 `.nvue`。它们的渲染方式不同，适用场景也有差异。理解两者的区别，是做好 uni-app 开发架构决策的基础。

---

## vue 与 nvue 的核心区别

**vue 文件**基于 Vue.js 框架，走 WebView 渲染，完整支持 Vue 的模板语法、响应式数据绑定、计算属性、组件系统等特性，适合 Web 端和对性能要求不极端的移动端页面。

**nvue 文件**（native vue）走 Weex 原生渲染引擎，输出的是真正的原生 UI，性能更接近原生 App。nvue 支持大部分 uni-app 的 JS API，但并非所有 Vue 特性都可用，且 CSS 写法受限，只支持 flex 布局。

一个 App 中可以同时使用两种页面。例如首页性能要求高，使用 nvue；二级页面使用 vue，两者可以共存，`hello uni-app` 官方示例即采用这种混合方式。

uni-app 也提供了 nvue 和 vue 之间的通信机制，通过 `uni.$emit` 触发事件、`uni.$on` 监听事件来跨页面通信。

---

## 什么是 nvue？

### nvue 的定位

nvue（native vue）即**原生渲染**。uni-app 采用逻辑与渲染分离的架构，在 App 端提供了两套渲染引擎：

- **WebView 渲染**：对应 `.vue` 文件，与小程序渲染方式相同；
- **原生渲染**：对应 `.nvue` 文件，基于 Weex 引擎，直接调用系统原生组件。

两种渲染方式的组件写法和 JS 写法基本一致，主要差异在于 CSS —— 原生渲染只支持 flex 布局。

### 为什么选择 nvue？

**① 性能更好**

WebView 渲染在复杂列表、长滚动等场景下性能有瓶颈，nvue 通过原生渲染可以显著提升流畅度，尤其适合对滚动性能要求高的页面。

**② 弥补了 Weex 的不足**

原始 Weex 只是一个渲染器，缺少 Push SDK、蓝牙、文件系统等丰富的原生能力，导致前端开发者仍然高度依赖 iOS/Android 原生工程师协作，反而增加了成本。nvue 在 Weex 渲染基础上，结合 uni-app 的完整 API 体系和插件生态，让前端工程师可以独立完成完整 App 的开发。

**③ 调用原生模块的必要条件**

如果需要在页面中调用自定义原生模块或原生组件，必须使用 nvue，vue 页面无法正常调用原生扩展。

**④ uni-app 对 nvue 的额外增强**

uni-app 在原生 Weex 基础上做了诸多扩展：

- Android 端良好支持边框阴影；
- iOS 端支持高斯模糊；
- 可实现区域滚动长列表 + 左右拖动列表 + 吸顶的复杂排版效果；
- 优化了圆角边框的绘制性能。

> 在 App 端某些 vue 页面表现不佳的场景下，可以用 nvue 作为强化补充。
{: .prompt-tip }

---

## nvue 与 vue 的常见开发区别

基于原生渲染引擎，nvue 与 Web 开发存在一些重要差异，开发前需要重点了解。

### 显隐控制

nvue 页面控制元素显隐只能使用 `v-if`，**不支持 `v-show`**。这是因为原生渲染没有 CSS `display` 的概念，`v-show` 底层依赖 `display:none` 无法生效。

### 布局限制

nvue 只支持 **flex 布局**，不支持其他布局方式（如 `position: absolute` 的自由布局在部分场景有限制）。

页面开发前建议先梳理清楚纵向内容结构和横轴排布，再按 flex 思路设计界面。

默认布局方向为**竖排（column）**。如需修改，可在 `manifest.json → app-plus → nvue → flex-direction` 中配置，仅在 uni-app 模式下生效。

编译为 H5 或小程序时，uni-app 会自动将 nvue 页面默认布局对齐为 flex + 垂直方向，以兼容 WebView 渲染的默认值差异。

### 文字与字体

- 文字内容**必须写在 `<text>` 组件内**，不能直接写在 `<view>` 或 `<div>` 的文本区域，否则即使渲染出来也无法绑定 JS 变量；
- `<text>` 组件内不能换行书写内容，否则会出现无法消除的周边空白；
- **只有 `<text>` 标签**可以设置字体大小和字体颜色；
- 不支持在 `style` 中通过 `@font-face` 引入字体文件，字体图标需参考官方文档单独加载。

### 样式限制

- **不支持百分比布局**，如 `width: 100%` 无效；
- **不支持媒体查询**；
- **不支持复合样式简写**，如 `border: 1px solid #ccc` 需拆分为独立属性；
- **不支持背景图**（`background-image`），可用 `<image>` 组件配合层级 `z-index` 实现类似效果；
- CSS 选择器支持有限，**只支持 class 选择器**；
- `class` 绑定**只支持数组语法**，不支持对象语法。

### Android 端注意

- nvue 组件在 Android 端默认背景为**透明**，若不设置 `background-color`，可能出现组件重影问题；
- 在同一页面内大量使用圆角边框，尤其是多角样式不一致时，会造成 Android 端性能下降，应避免此类用法。

### 滚动与页面高度

原生渲染没有页面自动滚动的概念，内容超出屏幕高度不会自动出现滚动条。只有以下组件支持滚动：`list`、`waterfall`、`scroll-view`/`scroller`、`recycle-list`。

需要滚动的内容必须包裹在可滚动组件内。uni-app 编译为 App 模式时，会自动给页面外层套一个 `scroller`，但包含 `recycle-list` 的页面不会自动套。

nvue 切换横竖屏时可能引发样式异常，建议有 nvue 的页面**锁定屏幕方向**。

### 导航栏建议

关闭原生导航栏后，若用 nvue 模拟状态栏，在页面进入动画期间可能出现白屏或闪屏。强烈建议 nvue 页面**保留原生导航栏**，原生排版引擎绘制导航栏的速度远快于 nvue 渲染整个页面的速度。

### 全局变量与样式

- `App.vue` 中定义的**全局 JS 变量**在 nvue 页面不生效；`globalData` 和 `Vuex` 正常生效；
- `App.vue` 中定义的**全局 CSS** 对 nvue 和 vue 页面同时生效。若全局样式中含有 nvue 不支持的属性，编译时控制台会报警，建议将这些样式用条件编译包裹：

```css
/* #ifdef APP-PLUS-NVUE */
/* nvue 专属样式 */
/* #endif */
```

### 其他

- 目前 nvue 页面**不支持 TypeScript**；
- `bounce` 回弹效果在 nvue 中默认不支持，只有 `list`、`recycle-list`、`waterfall` 这几个列表组件有 bounce 效果。

---

## nvue 使用注意事项速查

| 特性                     | nvue 支持情况                     |
| ------------------------ | --------------------------------- |
| flex 布局                | ✅ 唯一支持的布局方式              |
| 百分比宽高               | ❌ 不支持                          |
| 媒体查询                 | ❌ 不支持                          |
| 背景图                   | ❌ 不支持，用 `<image>` + 层级替代 |
| `v-show`                 | ❌ 不支持，用 `v-if` 替代          |
| 复合样式简写             | ❌ 不支持，需拆分为独立属性        |
| `@font-face` 引入字体    | ❌ 不支持，需单独加载              |
| TypeScript               | ❌ 暂不支持                        |
| class 对象语法绑定       | ❌ 只支持数组语法                  |
| 文字直接写在 `<view>` 中 | ❌ 必须用 `<text>` 包裹            |
| 全局 JS 变量（App.vue）  | ❌ 不生效，用 `globalData` 或 Vuex |
| 全局 CSS（App.vue）      | ✅ 生效                            |
| Vuex / globalData        | ✅ 正常使用                        |
| uni.$emit / uni.$on      | ✅ 正常使用                        |
| 原生模块调用             | ✅ nvue 专属能力                   |