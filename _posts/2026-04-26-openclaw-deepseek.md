---
title: OpenClaw 本地模型太慢？切换到 DeepSeek V4 的实践记录
description: 本地跑不动，API 便宜又高效——记一次从 Ollama 本地模型到云端 DeepSeek V4 的平滑迁移。
author: morric
date: 2026-04-26 22:14:00 +0800
categories: [开发记录]
tags: [ai, ollama, openclaw, 工具]
pin: true
math: true
mermaid: true
---

最近在尝试 OpenClaw 这个 Agent 框架，一开始图省事用了本地 Ollama 跑的模型。结果发现——**太慢了**。

慢到什么程度呢？稍微复杂一点的任务直接 timeout，更别提多轮对话或者需要推理的步骤了。研究了一下 Agent 框架的基本门槛，大概需要 **32B+** 参数量的模型才能玩得转。可我的机器只有 32GB 内存，跑 32B 模型简直是强人所难（即使量化后也够呛）。

刚好 DeepSeek V4 发布了（没错，就是那个价格屠夫），我看了一眼定价——**依然很香**。于是决定把 OpenClaw 的模型后端从本地换成云端 DeepSeek V4。同时顺手整理了 Claude Code 的模型切换方案，以后想换什么模型就换什么模型，再也不用被本地硬件绑死了。

> 本文记录的就是这次迁移过程中的工具选择与配置方法。

---

## 核心工具：`cc switch`

`cc switch` 是一个轻量级的模型切换工具，主要解决了一个痛点：**Claude Code 这类工具默认只能用一个模型，想换模型就得改配置、重启，非常麻烦**。

有了 `cc switch`，你可以：
- 在多个云端模型（DeepSeek、Claude、GPT 等）之间实时切换
- 同时管理本地模型（通过 LiteLLM 代理）
- 不用停止正在运行的任务，动态生效

> 简单说：它就是 Claude Code 的“模型遥控器”。

---

![cc switch](/assets/images/2026/202604261.png)

## 让 Claude Code 访问本地模型：需要 LiteLLM

虽然我最终换成了云端 DeepSeek V4，但本地模型在某些离线场景下还是很有用的（比如断网调试、低成本测试）。为了让 Claude Code 能够调用 Ollama 里跑着的本地模型，需要一个**兼容 OpenAI API 格式的代理**——这里我用的是 **LiteLLM**。

### 1. 创建 LiteLLM 配置文件

```bash
# 创建配置目录（如果不存在）
mkdir -p ~/.litellm

# 写入配置文件
cat > ~/.litellm/litellm_config.yaml << 'EOF'
model_list:
  - model_name: my-local-model    # 这个名称会在 cc switch 中使用
    litellm_params:
      model: ollama/your_model_name   # 请将 your_model_name 替换为你实际的 Ollama 模型名，例如 qwen2.5-coder:7b
      api_base: http://localhost:11434
EOF

```

![litellm model](/assets/images/2026/202604261.png)

```bash
model_list:
  # 默认入口模型（Claude Code 会调用这个）
  - model_name: claude-code-default
    litellm_params:
      model: ollama/qwen2.5-coder:7b
      api_base: http://localhost:11434
      extra_body:
        think: false

  # 可选的第二个本地模型（DeepSeek-R1）
  - model_name: deepseek-local
    litellm_params:
      model: ollama/deepseek-r1:14b
      api_base: http://localhost:11434

  # 可选：作为备用或负载均衡的同一模型的不同实例
  - model_name: claude-code-default
    litellm_params:
      model: ollama/deepseek-r1:14b
      api_base: http://localhost:11434

litellm_settings:
  drop_params: true
  modify_params: true
  params_to_drop:
    - thinking
    - betas

router_settings:
  routing_strategy: "usage-based"        # 路由策略：基于使用量
  enable_loadbalancing: true             # 启用负载均衡
  num_retries: 2                         # 失败重试次数
  request_timeout: 60                    # 请求超时（秒）
  fallbacks: [{"claude-code-default": ["deepseek-local"]}]  # 降级策略

general_settings:
  master_key: "sk-your-secure-master-key"
  database_url: "postgresql://..."       # 可选，用于持久化
```

> 附注：cc switch 和 LiteLLM 的详细用法可以看各自的官方文档，本文只记录了打通本地 → 云端的核心链路。