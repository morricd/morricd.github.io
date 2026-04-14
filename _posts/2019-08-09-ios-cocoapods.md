---
title: iOS CocoaPods 私有库搭建完整指南
description: >-
  基于码云 Gitee，从零搭建 CocoaPods 私有 Spec 仓库与私有库，涵盖 pod lib create 创建模板、
  podspec 配置与验证、发布到私有仓库、项目引入的完整流程，以及常见报错的解决方法。
author: morric
date: 2019-08-09 20:39:00 +0800
categories: [开发记录, iOS]
tags: [ios]
pin: true
---

当 iOS 项目规模增大、团队分工越来越细，组件化开发就成了绕不开的话题。CocoaPods 私有库是实现组件化的核心手段——它既能让各业务模块独立维护，又能像使用公开第三方库一样方便地集成进项目。

本文基于码云 Gitee，介绍从零搭建 CocoaPods 私有库的完整流程。

> 前提：本机已安装 CocoaPods。如未安装，请先参考官方文档完成安装。

---

## 一、搭建私有 Spec 仓库

私有库的整个体系由两个仓库组成：**Spec 仓库**（存放 podspec 索引）和**私有库仓库**（存放实际代码）。先从 Spec 仓库开始。

**第一步：在码云上新建一个仓库**，用于存放后续所有私有库的 `.podspec` 索引文件，相当于你自己的私有 CocoaPods 索引中心。

![新建 Spec 仓库](/assets/images/2019/2019080901.png)

**第二步：填写仓库信息并创建**，建议勾选初始化 README。

![创建仓库](/assets/images/2019/2019080902.png)

**第三步：将远程 Spec 仓库关联到本地**，执行：

```bash
pod repo add MorricSpec https://gitee.com/xxxx/MorricSpec.git
```

`MorricSpec` 是该仓库在本地的别名，地址替换为你自己的仓库地址。执行成功后可以看到：

```
Cloning spec repo `MorricSpec` from `https://gitee.com/xxxx/MorricSpec.git`
```

通过以下命令确认添加成功：

```bash
pod repo list
```

看到 `MorricSpec` 出现在列表中即可。

---

## 二、创建私有库

使用 `pod lib create` 命令创建私有库模板工程：

```bash
pod lib create MorricKit
```

执行后会有几个交互式问题，按实际需求选择：

```
What platform do you want to use?  >> iOS
What language do you want to use?  >> Swift / ObjC
Would you like to include a demo application with your library? >> Yes
Which testing frameworks will you use? >> None
Would you like to do view based testing? >> No
```

完成后 Xcode 会自动打开，生成的目录结构如下：

```
MorricKit/
├── MorricKit.podspec          # 库的描述配置文件
├── MorricKit/
│   └── Classes/               # 组件源码放这里
└── Example/                   # 示例工程
    └── Podfile
```

将组件源码放入 `MorricKit/Classes/` 目录，删除其中的占位文件 `ReplaceMe.swift`（或 `.m`）。

---

## 三、配置 podspec 文件

`.podspec` 是私有库的核心描述文件，打开 `MorricKit.podspec` 进行配置：

```ruby
Pod::Spec.new do |s|
  s.name             = 'MorricKit'
  s.version          = '0.1.0'
  s.summary          = '一句话描述你的库'
  s.description      = <<-DESC
    比 summary 更详细的描述，长度必须超过 summary，否则验证时会报 warning。
  DESC

  s.homepage         = 'https://gitee.com/xxxx/MorricKit'
  s.license          = { :type => 'MIT', :file => 'LICENSE' }
  s.author           = { 'morric' => 'your@email.com' }
  s.source           = { :git => 'https://gitee.com/xxxx/MorricKit.git', :tag => s.version.to_s }

  s.ios.deployment_target = '9.0'
  s.source_files = 'MorricKit/Classes/**/*'

  # 如果依赖了其他第三方库，在这里声明
  # s.dependency 'AFNetworking', '~> 2.3'
end
```

配置完成后，给代码打上对应的 Git tag：

```bash
git add .
git commit -m "初始化 MorricKit 0.1.0"
git tag 0.1.0
git push origin master --tags
```

---

## 四、关联远程私有仓库与私有库

在码云上再新建一个仓库用于存放私有库的实际代码（与前面的 Spec 仓库是两个独立仓库），然后将本地 MorricKit 工程与之关联：

```bash
git remote add origin https://gitee.com/xxxx/MorricKit.git
git push -u origin master
```

此后每次更新代码，修改 podspec 中的版本号，打新 tag，再重新推送即可。

---

## 五、验证并发布到私有 Spec 仓库

**先在本地验证 podspec：**

```bash
pod lib lint MorricKit.podspec
```

如果私有库本身依赖了其他私有库，需要通过 `--sources` 指定来源：

```bash
pod lib lint MorricKit.podspec --sources='https://gitee.com/xxxx/MorricSpec.git,https://github.com/CocoaPods/Specs'
```

验证通过后会看到：

```
MorricKit passed validation.
```

**再将 podspec 推送到私有 Spec 仓库：**

```bash
pod repo push MorricSpec MorricKit.podspec
```

推送成功后，可以在本地 `~/.cocoapods/repos/MorricSpec/` 目录下看到对应的 podspec 文件，说明发布完成。

---

## 六、项目引入私有库

在项目的 `Podfile` 中，需要同时声明私有 Spec 仓库地址和官方源（私有源放在前面）：

```ruby
source 'https://gitee.com/xxxx/MorricSpec.git'
source 'https://github.com/CocoaPods/Specs.git'

platform :ios, '9.0'

target 'YourApp' do
  use_frameworks!
  pod 'MorricKit', '~> 0.1.0'
end
```

执行 `pod install`，引入成功后即可在项目中正常使用。

---

## 七、搭建过程中遇到的问题

### podspec 验证失败

验证失败是搭建过程中最容易碰到的问题，以下是几种常见情况：

**① description 比 summary 短**

```
- WARN | description: The description is shorter than the summary.
```

将 `s.description` 的内容写得比 `s.summary` 更详细即可解决。

**② tag 与版本号不匹配**

```
- ERROR | [iOS] unknown: Unable to find a specification for...
```

确认已打好与 `s.version` 一致的 Git tag，且已推送到远程。

**③ 找不到依赖的私有库**

```
- ERROR | [iOS] unknown: Unable to find a specification for 'XXX'
```

验证时通过 `--sources` 参数将私有 Spec 仓库地址一并传入。

**④ 只有 warning 没有 error**

加上 `--allow-warnings` 跳过警告强制通过：

```bash
pod lib lint MorricKit.podspec --allow-warnings
pod repo push MorricSpec MorricKit.podspec --allow-warnings
```

---

## 总结

整个流程可以概括为五步：

1. 在 Gitee 创建 Spec 仓库，并通过 `pod repo add` 关联到本地
2. 用 `pod lib create` 生成私有库模板工程，编写组件代码
3. 配置 `.podspec` 文件，打 tag 并推送到远程
4. 通过 `pod lib lint` 验证，再用 `pod repo push` 发布到私有 Spec 仓库
5. 在项目 `Podfile` 中声明私有源，`pod install` 引入

后续每次迭代，只需更新代码 → 修改版本号 → 打新 tag → 重新 push 即可。