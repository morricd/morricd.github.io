---
title: Kotlin 与 Java 混合开发指南
author: morric
date: 2021-06-26 20:40:00 +0800
categories: [开发记录, Android]
tags: [android, kotlin]
---

Kotlin 与 Java 的互操作性是其最大优势之一，两者可以在同一项目中无缝共存。但混合开发时有一些细节需要特别注意，本文整理了属性读写、空安全、互操作注解、集合、I/O 等常见场景的处理方式。

> 在 Kotlin 中继承 Java 方法并调用时，建议将参数声明为**可空类型**，除非能确定 Java 方法的参数一定非空。Java 代码中若参数可能为 `null`，应使用 `@Nullable` 注解明确标注。
{: .prompt-tip }

---

## 一、属性读写

### Kotlin 调用 Java

Kotlin 能自动识别 Java 的 getter / setter，直接用属性语法访问即可，无需手动调用 `getXxx()` 或 `setXxx()`。

### Java 调用 Kotlin

Kotlin 的属性在编译后会生成对应的 getter / setter，Java 代码需要通过这些方法来访问。

例如 Kotlin 中声明：

```kotlin
var firstName: String = ""
```

编译后等价于 Java 中的：

```java
private String firstName;

public String getFirstName() {
    return firstName;
}

public void setFirstName(String firstName) {
    this.firstName = firstName;
}
```

> 特殊情况：若属性名以 `is` 开头（如 `isOpen`），其 getter 为 `isOpen()`，setter 为 `setOpen()`。

---

## 二、空安全

Kotlin 有完整的空安全机制，Java 没有。两者互调时需要特别注意：

- Kotlin 调用 Java 方法时，返回值类型是**平台类型**（既可能为空也可能非空），建议明确用可空类型 `String?` 接收；
- Java 调用 Kotlin 方法时，若 Kotlin 参数是非空类型，Java 传入 `null` 会在运行时抛出 `NullPointerException`。

推荐在 Java 代码中使用注解明确语义：

```java
// 明确表示参数可以为 null
public void setName(@Nullable String name) { ... }

// 明确表示参数不能为 null
public void setId(@NotNull String id) { ... }
```

---

## 三、Kotlin 提供给 Java 使用的常用注解

### @JvmField

将 Kotlin 属性编译为真正的 Java 字段，Java 可以直接访问，无需通过 getter / setter：

```kotlin
@JvmField
val TAG = "MyActivity"
```

### @JvmStatic

将 Kotlin 的 `companion object` 中的方法编译为真正的 Java 静态方法：

```kotlin
companion object {
    @JvmStatic
    fun newInstance(): MyFragment = MyFragment()
}
```

Java 调用时可以直接写 `MyFragment.newInstance()`，而不需要 `MyFragment.Companion.newInstance()`。

### @JvmOverloads

Kotlin 支持参数默认值，但 Java 不支持。`@JvmOverloads` 会自动为带默认值的参数生成多个重载方法，方便 Java 调用：

```kotlin
@JvmOverloads
fun testFun(a: String, b: Int = 0, c: String = "abc") {}
```

等价于 Java 中声明了三个方法：

```java
void testFun(String a)
void testFun(String a, int b)
void testFun(String a, int b, String c)
```

### @file:JvmName

修改 Kotlin 文件编译后生成的 Java 类名，避免默认的 `FileNameKt` 命名不够直观：

```kotlin
@file:JvmName("Utils")
package com.example
```

---

## 四、SAM（单一抽象方法）

Java 中只有一个抽象方法的接口（SAM 接口）可以用 lambda 表达式替代实例化。Kotlin 同样支持对 **Java SAM 接口**使用 lambda：

```kotlin
// 直接用 lambda 替代 Runnable 实例
Thread { println("running") }.start()
```

对于 Kotlin 自定义的函数式接口，需要使用 `fun interface` 关键字声明，或通过类型别名简化：

```kotlin
typealias Action = () -> Unit
```

---

## 五、正则表达式

推荐使用 Kotlin 的原始字符串（Raw String）定义正则，避免手动转义：

```kotlin
// 匹配 IP 地址，Raw String 无需转义反斜杠
val ipRegex = Regex("""\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}""")
```

使用 Kotlin 的 `Regex` API 提取匹配结果：

```kotlin
ipRegex.findAll(input)
    .toList()
    .flatMap(MatchResult::groupValues)
    .forEach(::println)
```

---

## 六、集合类

### Kotlin 集合的不可变性

Kotlin 中 `listOf`、`setOf`、`mapOf` 创建的集合是**只读的**，不能增删元素。需要增删操作时，使用对应的可变版本：

```kotlin
// 只读集合
val readOnlyList = listOf("1", "Hello", "Kotlin")

// 可变集合
val mutableList = mutableListOf("1", "Hello", "Kotlin")
mutableList.add("World")
mutableList.remove("1")
```

### 使用 Java 的 ArrayList

Kotlin 可以直接使用 `java.util.ArrayList`，并通过 `removeAt` 按下标删除元素：

```kotlin
val list = ArrayList<String>()
list.add("Hello")
list.add("Kotlin")
list.removeAt(0)  // 按下标删除
list.remove("Kotlin")  // 按值删除
```

> `removeAt` 是 Kotlin 对 Java `ArrayList` 的优化——当列表是 `Int` 类型时，`remove(0)` 会产生歧义（是删除下标 0 的元素，还是删除值为 0 的元素？）。`removeAt` 语义明确，推荐使用。

---

## 七、I/O 操作

### Java 的文件读取（传统方式）

```java
File file = new File("filename.txt");
BufferedReader bufferedReader = null;
try {
    bufferedReader = new BufferedReader(new FileReader(file));
    String line;
    while ((line = bufferedReader.readLine()) != null) {
        System.out.println(line);
    }
} catch (FileNotFoundException e) {
    e.printStackTrace();
} catch (IOException e) {
    e.printStackTrace();
} finally {
    try {
        if (bufferedReader != null) bufferedReader.close();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

### Kotlin 的文件读取

Kotlin 通过 `use` 扩展函数自动管理资源关闭，无需手动调用 `close()`：

```kotlin
val file = File("filename.txt")
BufferedReader(FileReader(file)).use { reader ->
    while (true) {
        val line = reader.readLine() ?: break
        println(line)
    }
}
```

对于小文件，Kotlin 提供了更简洁的扩展方法：

```kotlin
File("filename.txt").readLines().forEach(::println)
```

---

## 八、装箱与拆箱

Java 中有 `int`、`double`、`boolean` 等基本类型，以及对应的包装类型 `Integer`、`Double`、`Boolean`。在集合等泛型场景中需要使用包装类型，JVM 会自动完成装箱（基本类型 → 对象）和拆箱（对象 → 基本类型）的转换。

Kotlin 只有 `Int`、`Double`、`Boolean` 等对象类型，编译器会根据使用场景自动决定是否使用基本类型，开发者无需手动处理。

---

## 九、注解处理器（kapt）

Kotlin 使用 `kapt`（Kotlin Annotation Processing Tool）替代 Java 的 `apt` 来处理注解。以 Dagger2 为例：

**第一步：在模块的 `build.gradle` 中启用 kapt 插件**

```gradle
apply plugin: 'kotlin-kapt'
```

如果项目之前使用了 `android-apt` 插件，需要将其移除，并将所有 `apt` 配置替换为 `kapt`。

**第二步：配置注解处理器依赖**

```gradle
dependencies {
    kapt "com.google.dagger:dagger-compiler:$dagger_version"
}
```

**测试代码中使用注解处理器：**

- `androidTest` 使用：`kaptAndroidTest`
- `unitTest` 使用：`kaptTest`

> `kaptAndroidTest` 和 `kaptTest` 是 `kapt` 的扩展配置。如果只配置了 `kapt` 依赖，它会同时作用于生产代码和测试代码；如需仅在测试中使用，则使用对应的专属配置。
{: .prompt-tip }