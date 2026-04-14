---
title: iOS podspec 书写规范
author: morric
date: 2019-08-11 00:27:00 +0800
categories: [开发记录, iOS]
tags: [ios]
---

`.podspec` 文件是 CocoaPods 私有库的核心描述文件，它告诉 CocoaPods 这个库叫什么、源码在哪、依赖了什么。写好 podspec 是搭建私有库的基础，本文系统梳理各字段的含义与用法。

---

## 完整参数示例

```ruby
Pod::Spec.new do |spec|

  spec.name         = "BaseFramework"          # 库的名称
  spec.version      = "0.0.1"                  # 库的版本号
  spec.summary      = "BaseFramework 简介"      # 简短摘要
  spec.description  = <<-DESC
    BaseFramework 的详细描述，内容长度必须比 summary 长，否则验证时会报 warning。
  DESC

  spec.homepage     = "https://gitee.com/xxxx/BaseFramework"  # 库的主页
  spec.license      = "MIT"                                    # 开源协议
  spec.author       = { "morric" => "your@email.com" }        # 作者信息

  spec.ios.deployment_target = "9.0"           # 最低支持的 iOS 版本

  spec.source = {
    :git => "https://gitee.com/xxxx/BaseFramework.git",
    :tag => "#{spec.version}"                  # tag 与版本号保持一致
  }

  spec.source_files          = "Classes/**/*.{h,m,swift}"   # 引入的源码文件
  spec.exclude_files         = "Classes/Exclude"             # 排除的文件
  spec.public_header_files   = "Classes/**/*.h"              # 暴露的头文件（ObjC）

  spec.resources             = "Resources/*"                 # 资源文件
  spec.resource_bundles      = {
    'BaseFramework' => ['Resources/**/*.{png,xib,storyboard,json}']
  }                                                          # bundle 资源

  spec.frameworks            = "UIKit", "Foundation"         # 依赖的系统 framework
  spec.weak_frameworks       = "MetricKit", "VisionKit"      # 弱引用的系统 framework
  spec.vendored_frameworks   = []                            # 依赖的第三方 framework

  spec.libraries             = "iconv", "xml2"               # 依赖的系统 library
  spec.vendored_libraries    = []                            # 依赖的第三方 library

  spec.requires_arc          = true                          # 是否使用 ARC

  spec.dependency "AFNetworking", "~> 3.0"                   # 依赖的第三方库

end
```

---

## 基础字段

### name / version / summary / description

`name` 是库的唯一标识，必须与 `.podspec` 文件名保持一致。`version` 对应 Git 仓库的 tag，两者必须匹配，否则验证时会报错。

`description` 的内容长度必须比 `summary` 长，否则 `pod lib lint` 验证时会出现以下警告：

```
- WARN | description: The description is shorter than the summary.
```

### source

`source` 指定库的 Git 地址和对应的 tag：

```ruby
spec.source = { :git => "https://gitee.com/xxxx/BaseFramework.git", :tag => "#{spec.version}" }
```

`#{spec.version}` 会自动读取上面定义的 `version` 值，确保两者始终保持一致，避免手动维护时出错。

---

## source_files —— 源码文件

`source_files` 是 podspec 中最核心的字段，决定了哪些文件会被引入主工程。

```ruby
spec.source_files = "Classes/**/*.{h,m,swift}"
```

路径规则说明：

- `**` 匹配任意层级的子目录
- `*.{h,m}` 匹配所有 `.h` 和 `.m` 后缀的文件
- 如果是 Swift 库，则改为 `*.{swift}` 或 `*.{h,m,swift}`

如果需要排除某些文件，使用 `exclude_files`：

```ruby
spec.exclude_files = "Classes/Exclude"
```

---

## public_header_files —— 暴露的头文件

```ruby
spec.public_header_files = "Classes/**/*.h"
```

这个字段只在打包成 framework 时生效，用于指定哪些头文件对外可见。未列入其中的头文件会被标记为 `project` 级别，外部无法直接引用。

纯 Swift 库不需要配置此项。

---

## resources / resource_bundles —— 资源文件

### resources

```ruby
spec.resources = "Resources/*"

# 也可以直接引入 bundle 文件
spec.resource = 'MJRefresh/MJRefresh.bundle'
```

`resources` 用于引入图片、JSON、xib、bundle 等资源文件，这些文件会被直接复制到主工程的 bundle 中。

### resource_bundles

```ruby
spec.resource_bundles = {
  'BaseFramework' => ['Resources/**/*.{png,xib,storyboard,json}']
}
```

相比 `resources`，`resource_bundles` 会将资源打包进一个独立的 `.bundle` 文件，有效避免多个库之间的资源命名冲突，**推荐优先使用这种方式**。

---

## frameworks / weak_frameworks —— 系统库依赖

### 强引用（frameworks）

```ruby
spec.frameworks = 'UIKit', 'Foundation'
```

强引用表示必须链接该系统库，如果系统中不存在则编译报错。适用于 iOS 各版本均支持的系统库。

### 弱引用（weak_frameworks）

```ruby
spec.weak_framework  = 'MetricKit'              # 弱引用单个
spec.weak_frameworks = 'MetricKit', 'VisionKit' # 弱引用多个
```

弱引用表示运行时动态检查该库是否存在，不存在时不会崩溃。适用于高版本 iOS 才引入的系统库，需要在代码中配合版本判断使用：

```swift
if #available(iOS 13.0, *) {
  // 使用 MetricKit
}
```

---

## vendored_frameworks / vendored_libraries —— 第三方依赖

### vendored_frameworks

```ruby
spec.vendored_frameworks = 'Frameworks/Bugly.framework'
```

用于引入预编译的第三方 `.framework` 文件，路径相对于 podspec 所在目录。

### vendored_libraries

```ruby
spec.vendored_libraries = 'Libraries/libWeChatSDK.a'
```

用于引入预编译的第三方静态库 `.a` 文件。

---

## libraries —— 系统 library 依赖

```ruby
spec.libraries = "iconv", "xml2"
```

用于链接系统自带的 `.dylib` 或 `.tbd` 库，填写时去掉前缀 `lib`，例如 `libiconv.tbd` 写作 `iconv`。

---

## dependency —— 依赖其他 Pod

```ruby
spec.dependency "AFNetworking", "~> 3.0"
spec.dependency "SDWebImage"
```

声明该库依赖的第三方 Pod，版本号规则与 Podfile 一致：

- `~> 3.0` 表示 3.0 ≤ 版本 < 4.0
- `~> 3.1.2` 表示 3.1.2 ≤ 版本 < 3.2.0
- 不写版本号则默认使用最新版

---

## requires_arc

```ruby
spec.requires_arc = true
```

指定库是否使用 ARC（自动引用计数）。现代项目几乎都是 ARC，保持默认 `true` 即可。如果部分文件不支持 ARC，也可以针对单个文件关闭：

```ruby
spec.requires_arc = false
spec.source_files = 'Classes/Legacy/**/*.{h,m}'
```

---

## 常用字段速查

| 字段                  | 说明                      | 是否必填 |
| --------------------- | ------------------------- | :------: |
| `name`                | 库名称，需与文件名一致    |    ✅     |
| `version`             | 版本号，需与 Git tag 一致 |    ✅     |
| `summary`             | 简短摘要                  |    ✅     |
| `description`         | 详细描述，须长于 summary  |    ✅     |
| `homepage`            | 库的主页地址              |    ✅     |
| `license`             | 开源协议                  |    ✅     |
| `author`              | 作者信息                  |    ✅     |
| `source`              | Git 地址与 tag            |    ✅     |
| `source_files`        | 引入的源码文件路径        |    ✅     |
| `exclude_files`       | 排除的文件路径            |    ❌     |
| `public_header_files` | 暴露的头文件（ObjC）      |    ❌     |
| `resources`           | 资源文件                  |    ❌     |
| `resource_bundles`    | 打包为独立 bundle 的资源  |    ❌     |
| `frameworks`          | 强引用的系统 framework    |    ❌     |
| `weak_frameworks`     | 弱引用的系统 framework    |    ❌     |
| `vendored_frameworks` | 第三方预编译 framework    |    ❌     |
| `libraries`           | 系统 library              |    ❌     |
| `vendored_libraries`  | 第三方预编译静态库        |    ❌     |
| `dependency`          | 依赖的第三方 Pod          |    ❌     |
| `requires_arc`        | 是否使用 ARC              |    ❌     |