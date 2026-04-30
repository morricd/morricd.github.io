---
title: OpenClaw 从云端模型迁移到本地模型
description: 从 MiniMax-M1.5 迁移到 Ollama 本地模型，M1 Max Mac 跑 7B 模型完全够用，日常指令任务成本降到零。
author: morric
date: 2026-04-21 22:00:00 +0800
categories: [开发记录]
tags: [ai, ollama, openclaw, 工具]
pin: true
math: true
mermaid: true
---

平时工作用 Claude Code 处理复杂的开发任务，Claude Code 验证也是焦虑，还好我不是重度使用者，新的项目如果是完整的AI开发我会转移codex,我基本是AI上实现部分功能的代码片段，所以没有什么影响。OpenClaw 主要负责日常自动化指令——抓新闻、推代码、发提醒、跑脚本。这类任务不需要顶级模型，但如果一直走云端 API，费用积累下来也不少。

自己的 MacBook Pro M1 Max 内存够大，跑 7B 左右的本地模型完全没有问题，于是把 OpenClaw 的模型从 MiniMax 迁移到了本地 Ollama。日常指令任务成本直接降到零，响应速度也更快。

---

## 一、Ollama 是什么

Ollama 是一个在本地运行大语言模型的工具，支持 macOS、Linux 和 Windows。它把模型的下载、管理、运行全部封装好，对外提供一个兼容 OpenAI 格式的 HTTP 接口，和各种工具的对接非常方便。

核心特点：

- **本地运行**：数据不出机器，没有隐私泄露风险
- **OpenAI 兼容接口**：默认监听 `http://localhost:11434`，支持 `/v1/chat/completions`，可以直接替换云端 API
- **模型管理简单**：一条命令下载、运行、删除模型
- **Apple Silicon 优化**：利用 Metal GPU 加速，M 系列芯片跑模型效率很高

---

## 二、安装 Ollama

```bash
# 方式一：官网下载安装包（推荐）
# 访问 https://ollama.com 下载 macOS 安装包，拖入 Applications 即可

# 方式二：命令行安装
curl -fsSL https://ollama.com/install.sh | sh
```

安装完成后，Ollama 会在后台以服务形式运行，菜单栏出现图标即表示运行正常。

验证安装：

```bash
ollama --version
```

---

## 三、常用命令

```bash
# 下载并运行模型（首次会自动下载）
ollama run qwen2.5:7b

# 只下载不运行
ollama pull qwen2.5:7b

# 查看已安装的模型列表
ollama list

# 查看正在运行的模型
ollama ps

# 删除模型
ollama rm qwen2.5:7b

# 查看模型详细信息
ollama show qwen2.5:7b

# 停止运行中的模型
ollama stop qwen2.5:7b

# 查看 Ollama 服务状态
ollama serve

# 用 nohup 在终端后台启动
nohup ollama serve > /tmp/ollama.log 2>&1 &
这条命令会立即启动服务并放到后台，关闭终端也不会中断。
日志写入 /tmp/ollama.log，出问题时可以查看。

# 用 nohup 在终端后台启动的停止
pkill ollama

```

---

## 四、模型选择

针对 OpenClaw 日常指令任务（执行脚本、处理文件、抓取信息、简单问答），推荐以下几个模型：

### qwen2.5:7b（首选）

阿里巴巴出品，中文支持最好，对中文指令的理解和输出质量在同量级模型里最强。日常用中文给 OpenClaw 发指令，选这个准没错。

```bash
ollama pull qwen2.5:7b
```

内存占用约 5GB，M1 Max 跑起来非常流畅。

### deepseek-coder:6.7b（代码任务）

代码生成能力强，如果 OpenClaw 的任务涉及较多代码片段生成或脚本编写，可以用这个。

```bash
ollama pull deepseek-coder:6.7b
```

### llama3:8b（通用备选）

Meta 出品的通用模型，英文能力强，任务覆盖面广，作为备用模型不错。

```bash
ollama pull llama3:8b
```

**选择建议**：日常 OpenClaw 指令任务，`qwen2.5:7b` 足够；有代码任务时切换到 `deepseek-coder:6.7b`。不需要全部都装，按需下载。

---

## 五、模型存放位置

Mac 上 Ollama 的模型默认存储在：

```
~/.ollama/models/
```

每个模型包含两类文件：

```
~/.ollama/models/
├── blobs/          # 模型权重文件（体积大，几 GB 到几十 GB）
└── manifests/      # 模型元数据（很小）
```

查看模型占用的磁盘空间：

```bash
du -sh ~/.ollama/models/
```

如果磁盘空间紧张，可以把 `~/.ollama` 目录迁移到外接 SSD，然后用软链接指向新位置：

```bash
mv ~/.ollama /Volumes/外接SSD/ollama
ln -s /Volumes/外接SSD/ollama ~/.ollama
```

---

## 六、模型常驻内存吗？

默认情况下，模型**不会永久常驻内存**。Ollama 的行为是：

- 收到请求时将模型加载到内存
- **空闲 5 分钟后自动卸载**，释放内存

这意味着：
- 第一次请求响应稍慢（加载模型需要几秒）
- 之后的连续请求很快（模型已在内存中）
- 长时间不用时内存会自动释放

如果你希望模型一直保持加载状态（比如 OpenClaw 需要频繁调用），可以修改超时时间：

```bash
# 设置模型在内存中保持 1 小时后才卸载
OLLAMA_KEEP_ALIVE=1h ollama serve

# 永久保持（直到手动停止）
OLLAMA_KEEP_ALIVE=-1 ollama serve
```

或者在 macOS 的 LaunchAgent 配置中加入环境变量，让每次启动都生效：

```bash
# 编辑 Ollama 的 plist 配置
# 文件位置：~/Library/LaunchAgents/com.ollama.ollama.plist
```

---

## 七、内存 vs 磁盘

| 项目                         | 说明                                                       |
| ---------------------------- | ---------------------------------------------------------- |
| **磁盘占用**                 | 模型文件永久存储在 `~/.ollama/models/`，下载后一直占用磁盘 |
| **内存占用**                 | 只在运行时占用，空闲超时后自动释放                         |
| **qwen2.5:7b 磁盘**          | 约 4.7GB                                                   |
| **qwen2.5:7b 内存**          | 运行时约 5~6GB                                             |
| **deepseek-coder:6.7b 磁盘** | 约 3.8GB                                                   |
| **deepseek-coder:6.7b 内存** | 运行时约 4~5GB                                             |

M1 Max 标配 32GB 统一内存，同时跑一个 7B 模型 + 正常工作完全没有压力。

---

## 八、正确卸载模型

直接删文件可能残留元数据，推荐用命令卸载：

```bash
# 正确方式：通过命令删除
ollama rm qwen2.5:7b

# 验证是否已删除
ollama list
```

如果要彻底卸载 Ollama 本身：

```bash
# 停止服务
ollama stop

# 删除应用
rm -rf /usr/local/bin/ollama
rm -rf ~/.ollama

# macOS 还需要删除 LaunchAgent
rm ~/Library/LaunchAgents/com.ollama.ollama.plist
```

---

## 九、OpenClaw 配置切换到本地模型

OpenClaw 支持配置自定义模型接口，Ollama 提供了 OpenAI 兼容的 API，切换非常简单。

在 OpenClaw 的配置文件中修改模型设置：

```yaml
# ~/.openclaw/config.yaml 或通过 openclaw config 命令修改

model:
  provider: openai-compatible
  base_url: http://localhost:11434/v1
  api_key: ollama          # Ollama 不需要真实 key，随便填
  model: qwen2.5:7b        # 指定使用的本地模型
```

或者通过命令行配置：

```bash
openclaw config set model.provider openai-compatible
openclaw config set model.base_url http://localhost:11434/v1
openclaw config set model.api_key ollama
openclaw config set model.model qwen2.5:7b
```

配置完成后重启 OpenClaw Gateway：

```bash
openclaw gateway restart
```

发一条测试消息验证是否切换成功：

```
你好，现在用的是什么模型？
```

正常回复即表示本地模型已接入。

---

## 模型选择和参数设置
切换到本地模型后，有时候需要针对任务类型微调参数，让输出更稳定。
temperature：控制输出随机性。OpenClaw 执行的多是指令类任务（调用脚本、格式化输出、填充模板），不需要创意发散，建议设为 0.2～0.4，输出更确定、更可预测。如果任务涉及文案生成或头脑风暴，可以提高到 0.7 左右。
num_ctx（上下文长度）：qwen2.5:7b 默认上下文窗口是 4096 tokens，对于短指令任务完全够用。如果任务需要处理较长文档或多轮对话，可以在 Modelfile 里把 num_ctx 调到 8192，但内存占用会随之增加。
Modelfile 自定义：如果某个任务类型需要固定的系统提示词（比如让模型只输出 JSON），可以创建一个自定义模型：

```bash
# 创建 Modelfile
cat > Modelfile <<EOF
FROM qwen2.5:7b
PARAMETER temperature 0.3
SYSTEM "你是一个严格输出 JSON 的助手，不输出任何其他内容。"
EOF

# 构建自定义模型
ollama create openclaw-json -f Modelfile

# 在 OpenClaw 中使用
openclaw config set model.model openclaw-json
```

## 有些时候还是有点卡
7B 模型在 M1 Max 上整体流畅，但有几种情况会明显卡顿，记录一下：
首次加载：模型从磁盘加载到内存需要 3～8 秒，取决于磁盘速度。用 OLLAMA_KEEP_ALIVE 让模型常驻内存可以完全消除这个延迟，代价是一直占用 5～6 GB 内存。
长输出任务：让模型生成几百行代码或长篇文章时，生成速度大约是 20～40 tokens/秒，比云端明显慢。对于 OpenClaw 的指令任务问题不大，但如果偶尔用来处理复杂任务，需要耐心等一下。
并发请求：Ollama 默认单任务处理，OpenClaw 同时发出多条指令时会排队而非并发，响应时间会叠加。目前的做法是在 OpenClaw 侧控制并发数，不超过 2 个请求同时发出。
内存压力：同时开着多个吃内存的应用（Xcode、虚拟机、大型 Electron 应用）时，模型加载会触发内存压缩，速度会下降一些。这种情况基本靠关掉不用的应用解决，没有更好的办法。
总体来说，上述问题在日常自动化任务中极少触发，绝大多数情况下响应速度比云端更快，毕竟省掉了网络往返延迟。

## 总结

迁移到本地模型后，OpenClaw 的日常指令任务成本降到零，响应速度比云端更快——省掉了网络往返延迟。
不过目前还在测试阶段。本地模型跑纯问答确实很快，但 OpenClaw 的使用场景不止于此：工具调用、多步指令、格式化输出……这些场景下 7B 模型的生成速度有时仍然跟不上节奏。如果后续测试下来体验不够流畅，还是会迁回云端——本地化省下的费用，抵不过效率损耗的成本。
复杂的开发任务（架构分析、大型代码生成）继续留给 Claude Code，两者分工明确——本地模型负责高频低复杂度，云端模型处理低频高复杂度。分工是理想情况，实际用下来再做决定。