---
title: MVC、MVP、MVVM 架构演进与 Vue 中的 MVVM 实践
author: morric
date: 2020-07-12 22:17:00 +0800
categories: [开发记录, H5]
tags: [h5, vue]
---

MV* 系列框架中，M（Model）和 V（View）的基础职责始终不变：Model 负责存储数据，View 负责展示数据。三种框架的本质区别，在于 **M 与 V 之间数据流转方式**的不同。

---

## 一、MVC（Model-View-Controller）

### 简介

MVC 是三者中历史最悠久的架构模式，MVP 和 MVVM 都是在它的基础上演进而来。

MVC 的核心目标是将 M 和 V 的代码分离，两者之间的通信**必须经过 Controller**，是典型的单向通信模型。

- **Model**：数据模型及业务逻辑，与 View 无关，只与业务有关；
- **View**：视图层，负责与用户交互，实现数据的输入和输出；
- **Controller**：控制器，连接 Model 与 View，处理用户请求，将处理结果返回给 View。

### 框架图

![MVC 框架图](/assets/images/2020/2020071201.png)

各部分之间的通信是**单向的**，呈三角形结构。

![MVC 流程图](/assets/images/2020/2020071202.png)

需要注意的是，Controller 触发 View 时并不直接更新 View 中的数据，View 的数据更新是通过**监听 Model 的变化**自动完成的，与 Controller 无关。

### MVC 的问题

- 大量业务逻辑集中在 Controller 层，代码量膨胀，维护压力大；
- Controller 与 View 之间一一对应，View 无法复用，产生大量冗余代码；
- View 层本身已具备独立处理事件的能力，却没有被充分利用。

为解决上述问题，MVP 框架应运而生。

---

## 二、MVP（Model-View-Presenter）

### 简介

MVP 比 MVC 晚出现约 20 年，是从 MVC 演变而来。将 Controller 改名为 Presenter，同时改变了通信方向——**由单向变为双向**。

- **Model**：数据存储及业务逻辑；
- **View**：视图层，负责展示与用户交互；
- **Presenter**：表示器，连接 Model 与 View，负责业务逻辑处理，是两者之间唯一的通信中介。

### 框架图

![MVP 框架图](/assets/images/2020/2020071203.png)

各部分之间的通信是**双向的**。

![MVP 流程图](/assets/images/2020/2020071204.png)

与 MVC 最大的区别在于：**View 层不再直接访问 Model 层**，所有交互必须经过 Presenter 提供的接口。这使得 View 层得以解耦，可以抽离为独立组件，复用性大幅提升。

### MVP 的问题

- View 和 Model 都需要经过 Presenter，Presenter 层逐渐变得臃肿；
- 没有数据绑定机制，所有数据同步都需要在 Presenter 中**手动处理**，代码量仍然较大。

为了让 View 和 Model 的数据始终保持自动同步，MVVM 框架出现了。

---

## 三、MVVM（Model-View-ViewModel）

### 简介

MVVM 与 MVP 的最大区别在于引入了**双向数据绑定**：View 的变动自动反映到 ViewModel，Model 的变动也自动反映到 View，数据同步无需手动干预。

- **Model**：后端传递的数据，负责数据处理与业务逻辑；
- **View**：视图层，将 Model 的数据以特定方式展示；
- **ViewModel**：视图模型，MVVM 的核心，负责 View 与 Model 之间的**双向数据绑定**。ViewModel 将数据同步自动化，开发者只需声明 View 要展示 Model 中的哪部分数据即可，无需手动同步。

### 框架图

![MVVM 框架图](/assets/images/2020/2020071205.png)

MVVM 与 MVP 结构相似，区别在于第三层的实现方式：Presenter 通过**手动调用方法**同步数据；ViewModel 通过**双向绑定**自动同步数据。

![MVVM 流程图](/assets/images/2020/2020071206.png)

MVVM 框架中数据流转有两个方向：

- **Model → View**：通过数据绑定实现；
- **View → Model**：通过 DOM 事件监听实现。

两个方向同时存在，即**双向数据绑定**。双向绑定本质上是一个模板引擎，数据变化时实时重新渲染视图。

![双向绑定示意图](/assets/images/2020/2020071207.png)

### MVVM 数据绑定的三种实现方式

- **数据劫持**：通过 `Object.defineProperty`（Vue 2）或 `Proxy`（Vue 3）拦截属性的读写操作；
- **发布-订阅模式**：数据变化时通知所有订阅者更新；
- **脏值检查**：AngularJS 1.x 采用的方式，通过定期检查数据是否变化来触发更新。

**Vue.js 采用数据劫持 + 发布-订阅模式**，其内部有三个核心角色：

- **Observer**：数据监听器，监听数据变化，数据发生改变时通知 Watcher；
- **Compiler**：指令解析器，解析模板中的指令，绑定相应事件，负责更新视图；
- **Watcher**：订阅者，接收 Observer 的通知后，决定调用哪个 Compiler 执行视图更新。

完整流程：Observer 监听数据 → 数据变化时通知 Watcher → Watcher 调用 Compiler 更新视图，完成双向绑定闭环。

> 从 MVC 到 MVP 再到 MVVM，是一个持续解耦、降低手动同步负担的演进过程。三者最核心的差异，在于 Model 与 View 之间"第三层"的设计思路不同。
{: .prompt-tip }

---

## 四、MVVM 在 Vue 中的实践

Vue.js 是 MVVM 模式的典型实现，也是目前最流行的前端 MVVM 框架之一（类似的还有 AngularJS、React）。

### Model：数据模型

在 Vue 中，Model 对应组件实例的 `data` 对象：

```js
new Vue({
  data: {
    message: 'Hello, World!'
  }
})
```

`data` 中的属性会被 Vue 的 Observer 劫持，任何变化都会被自动追踪。

### View：视图模板

View 对应 HTML 模板部分，通过 Vue 的模板语法与数据绑定：

```html
<div id="app">
  <p>{{ message }}</p>
  <input v-model="message" />
</div>
```

- `{{ message }}`：Mustache 插值语法，将 Model 数据渲染到视图；
- `v-model`：双向绑定指令，视图变化会同步到 Model，Model 变化也会同步到视图。

### ViewModel：Vue 组件实例

ViewModel 对应 Vue 组件实例，是连接 View 和 Model 的桥梁，负责**响应式数据绑定**和**视图状态控制**：

```js
Vue.component('my-component', {
  data() {
    return {
      message: 'Hello, World!'
    }
  },
  template: `
    <div>
      <p>{{ message }}</p>
      <input v-model="message" />
    </div>
  `
})
```

### 工作流程

1. 用户在输入框中修改内容；
2. `v-model` 监听到输入事件，通过双向绑定将新值写入 `data`（Model）；
3. Observer 检测到 `data` 变化，通知 Watcher；
4. Watcher 调用 Compiler 重新渲染视图，页面自动更新。

整个过程无需开发者手动操作 DOM，也无需手动同步数据，这正是 MVVM 相比 MVC/MVP 最大的优势所在。

---

## 三种架构横向对比

| 维度                | MVC               | MVP              | MVVM                  |
| ------------------- | ----------------- | ---------------- | --------------------- |
| 通信方式            | 单向              | 双向             | 双向                  |
| V 与 M 是否直接通信 | 可以              | 不可以           | 不可以                |
| 数据同步方式        | 手动              | 手动             | 自动（双向绑定）      |
| 第三层职责          | 路由 + 逻辑       | 逻辑 + 手动同步  | 自动数据绑定          |
| View 复用性         | 差                | 较好             | 好                    |
| 典型代表            | Spring MVC、Rails | Android 早期开发 | Vue、AngularJS、React |