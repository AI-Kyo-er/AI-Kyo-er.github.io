---
title: "欸？うそ？ huggingface-cli: command not found?"
categories: [CS, AI]
tags:
- AI
date: 2025-10-28
---

## 场景
今天创建了一个新的 conda 虚拟环境，然后自然安装了 `huggingface_hub` 准备下载别人的模型。结果竟然报错 `huggingface-cli: command note found`.

这怎么搞得？终端重启了也不行，照理说 conda 虚拟环境中也不会存在环境变量的问题。

## 'huggingface-cli download' is deprecated!
竟然花了我半个小时，其间尝试用 `conda install -c huggingface huggingface_hub` 来排除依赖项未正常安装的问题。都无法解决。

### 意外之提示
于是准备放弃，安装其他环境去了，然后安装完 `Transformers` 之后 pip 给出警告称 Transformers needs huggingface_hub <= 0.34.0 while huggingface_hub == 1.0.0...

直接意识到这个船新版本的 huggingface_hub 弃用了 huggingface-cli，于是连夜安装回 0.34.0 版本。（可能太新了直接搜没人提到，并且这种基础的下载器我根本没往版本特性上想，所以上网搜索的时候也没有带版本号<不然应该早发现了>）

### 最后的最后
用 0.34.0 下载的时候，赫然显示着：
```
WARNING: huggingface-hub 1.0.0 does not provide the extra 'cli'
```
大白于天下，呜呼。