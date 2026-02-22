---
title: '基于Claude的LifeSystem'
date: '2026-02-22T21:21:07+08:00'
lastmod: '2026-02-22T21:21:07+08:00'
keywords: ['claude']
categories: ['claude']
tags: ['claude']
author: '小十一狼'
---

最近在微博上看到爱可可-爱生活老师分享的一个[基于纯文本和 Claude Code 的个人生命操作系统](https://weibo.com/1402400261/5267766594766455?wm=3333_2001&from=10G2193010&sourcetype=weixin&rp_trans=dFZVU0RJSXZ0STdVaTI1REdtSVF3MzRoVGVtMzI4VXdVK0VteWpnMWhpbE51Um1Vc1pYTkZ4dElqSXZvM05oQ0RBOG1TcFkwYW93aWFrQ2hIVnRMTmJVMnFpSFlESDBPbFFhRjNwdzUxek4xU1BFNU91ci9ZamwwZlpZR0pEcjRDUFFDZFUxQW5SK2pNaERzbFZyc2xnPT0%3D&s_trans=Li1PdIkgUXm%2Bm6xZ3lyd8g%3D%3D_5267766594766455_s&s_channel=4)，日常协助个人做记录、做规划、提供反思、理清思路、培养习惯的一款自我提升项目，于是想自己也尝试一下，这里将初次尝试的步骤记录下来。

# 配置 Claude

安装 Claude Code：

```shell
npm install -g @anthropic-ai/claude-code
```

由于中国被限制访问，所以我们切换一下模型为 GLM，参考 [用 Claude Code 跑 GLM-5：安装 + 配置 + 上手](https://mp.weixin.qq.com/s/YuzQtQ_CvLsgNZ8vE2qrzw)，以下是我的 ~/.claude/settings.json 配置：

```json
{
    "env":
    {
        "ANTHROPIC_BASE_URL": "https://open.bigmodel.cn/api/anthropic",
        "ANTHROPIC_DEFAULT_HAIKU_MODEL": "glm-4.6V-Flash",
        "ANTHROPIC_DEFAULT_SONNET_MODEL": "glm-4.7-Flash",
        "ANTHROPIC_DEFAULT_OPUS_MODEL": "glm-5"
    },
    "permissions":
    {
        "allow": []
    }
}
```

在 [智普 BigModel](https://bigmodel.cn/usercenter/proj-mgmt/apikeys) 上注册账号后创建新的 API Key，并设置 ANTHROPIC_AUTH_TOKEN：

```shell
export ANTHROPIC_AUTH_TOKEN="替换为你的 API Key"
```

测试成功后，可写入 ~/.zshrc 或 ~/.bashrc，执行 source 生效。

# “开启” Life System

其实 [life-system](https://github.com/davidhariri/life-system) 仓库中的 README.md 步骤已经很明确了，照做还是很容易的。我这里也按步骤再叙述一下：

```shell
# 克隆项目
git clone git@github.com:davidhariri/life-system.git

# 本地新建自己名字命名的路径
# 切记替换这里的小写 yourname 和后文出现小写 yourname 的地方，为你自己的命名
mkdir -p ~/Documents/yourname/

# 替换 life-system 仓库下 CLAUDE.md 中 YOURNAME 占位的路径为上面新建的自己名字的命名
# 然后拷贝至 ~/.claude/CLAUDE.md，全局配置，适用于机器上所有项目，适合设置通用规则和常用工具链
cp CLAUDE.md ~/.claude/CLAUDE.md

# skills/morning 目录下的 SKILL.md 和 reference.md 也进行 YOURNAME 的占位替换
# 然后拷贝至 ~/.claude/skills/ 目录下
mkdir -p ~/.claude/skills
cp -R skills/morning ~/.claude/skills/morning

# 安装 journal.sh 脚本
# 脚本中的 JOURNAL_DIR 和 TEMPLATE 中的 YOURNAME 占位也进行替换
# 脚本中的 EDITOR_CMD 我修改成了 vim
mkdir -p ~/.scripts
cp scripts/journal.sh ~/.scripts/journal.sh
chmod +x ~/.scripts/journal.sh

# 添加 journal.sh 的别名 jrn
echo "alias jrn='~/.scripts/journal.sh'" >> ~/.zshrc
source ~/.zshrc

# 拷贝 plan.md、inbox.md journal/、reference/、people/、research/、templates/ 这些文件和目录到 ~/Documents/yourname 目录路径下
cp plan.md ~/Documents/yourname
cp -R journal/ ~/Documents/yourname/journal
cp -R reference/ ~/Documents/yourname/reference
cp -R people/ ~/Documents/yourname/people
cp -R research/ ~/Documents/yourname/research
cp -R templates/ ~/Documents/yourname/templates
cp inbox.md ~/Documents/yourname
```

进入 ~/Documents/yourname 目录，执行 claude 命令：

```shell
cd ~/Documents/yourname

claude
```

输入 morning 开启 life system：

![first-time-to-use-lifesystem](/基于Claude的LifeSystem/first-time-to-use-lifesystem.png)

接着持续沟通，最终生成了一个学习路径：

![roadmap-to-learn](/基于Claude的LifeSystem/roadmap-to-learn.png)

展示一下这个 roadmap 的内容：

```markdown
# Learning Roadmap: Kubernetes, Claude, and LLMs

**Created:** 2026-02-22
**Purpose:** Structured path to learn three critical technologies

---

## Phase 1: Foundations (Weeks 1-2)

### Kubernetes
- [ ] Core concepts: pods, services, deployments, controllers
- [ ] kubectl commands and workflow
- [ ] Local environment: minikube or kind
- [ ] Resources: k8s.io docs, "Kubernetes in Action"

### Claude API
- [ ] Anthropic API basics
- [ ] Prompt engineering fundamentals
- [ ] Python SDK (anthropic)
- [ ] Resources: Anthropic docs, Prompt library

### LLM Fundamentals
- [ ] What is a language model?
- [ ] Tokenization, context window
- [ ] Types: autoregressive, seq2seq, transformers
- [ ] Resources: Andrej Karpathy's YouTube, "Attention Is All You Need" paper

---

## Phase 2: Practical Integration (Weeks 3-4)

### Kubernetes
- [ ] Deploy a simple app to k8s
- [ ] ConfigMaps and Secrets
- [ ] Services and Ingress
- [ ] Basic health checks (liveness/readiness probes)

### Claude
- [ ] Build a basic chatbot CLI
- [ ] Temperature, top_p, max_tokens tuning
- [ ] System prompts
- [ ] Function calling

### LLM Applications
- [ ] RAG: Retrieval-Augmented Generation
- [ ] Simple prompt templates
- [ ] Processing structured vs unstructured text

---

## Phase 3: Advanced Kubernetes (Weeks 5-6)

- [ ] StatefulSets and DaemonSets
- [ ] ConfigMap/Secret best practices
- [ ] Horizontal Pod Autoscaling
- [ ] Custom Resources (CRDs)
- [ ] CI/CD with ArgoCD or Flux

---

## Phase 4: LLM Architecture & Claude Advanced (Weeks 7-8)

### LLMs
- [ ] Transformer architecture
- [ ] Attention mechanisms
- [ ] Fine-tuning basics (LoRA, PEFT)
- [ ] Evaluation metrics

### Claude
- [ ] Prompt caching (performance optimization)
- [ ] Batches API for bulk processing
- [ ] Tool use and function calling
- [ ] Claude for coding (Claude Code)

---

## Phase 5: Integration Projects (Weeks 9-10)

Build something that combines all three:

**Project Idea:** Deploy an LLM-powered application on Kubernetes

- [ ] Build a simple LLM service (FastAPI + Claude API)
- [ ] Containerize with Docker
- [ ] Deploy to Kubernetes cluster
- [ ] Add ingress, autoscaling, and monitoring
- [ ] Add functionality: chat interface, document processing

---

## Success Criteria

By end of Week 10, you should be able to:
- Deploy and manage applications on Kubernetes
- Build and deploy LLM-powered services
- Understand when to use Claude vs other models
- Have at least one integrated project

---

## Recommended Resources

### Kubernetes
- Official docs: https://kubernetes.io/docs
- Learning: k8straining.io
- Hands-on: k8s.io/tutorials

### Claude
- Anthropic docs: https://docs.anthropic.com
- Prompt library: https://docs.anthropic.com/claude/prompt-library

### LLMs
- Karpathy's channel: https://youtube.com/neo414
- Fast.ai: https://course.fast.ai
- LlamaIndex docs: https://docs.llamaindex.ai

---

## Prerequisites

- Basic Python knowledge
- Command line familiarity
- Understanding of APIs
- Git familiarity

---

## Notes

Add notes and questions as you progress. What tripped you up? What connected?
```

初步使用就到这里，目前还只是了解皮毛，后续按照这个 life system 的用法管理自己的目标，等得到一定体会和感悟时，再来和大家聊聊技巧和感受。
