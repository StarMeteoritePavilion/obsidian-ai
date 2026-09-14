---
title: 你装的skill都是如何被Claude Code识别和加载的
source: https://www.bilibili.com/video/BV19bjN61EaK
author: 张司机在路上
created: 2026-06-15
tags:
  - AI
  - Claude Code
  - Anthropic
  - Skill
  - 渐进式披露
  - Prompt Caching
  - ToolSearch
  - 上下文工程
  - 驾驭工程
  - 应用工程
---

# 你装的skill都是如何被Claude Code识别和加载的

Claude Code 不会在会话开始时把每个 `SKILL.md` 的完整正文全部交给模型。Skill 采用渐进式披露：先暴露名称与描述，模型判断任务需要某项 Skill 后再索取完整说明，说明中引用的参考文档和脚本则按任务需要继续读取。

## 第一层：用名称和描述建立 Skill 清单

抓包中的 System Prompt 包含：

> The following skills are available for use with the Skill tool

其后列出当前可用的全部 Skill，每项只包含名称和一段描述。描述说明该 Skill 应在什么任务中使用，内容来自对应 `SKILL.md` 开头的 frontmatter。

案例使用 `document-skills:pdf` 分析 *Attention Is All You Need*，并解释其中的 Attention 公式。PDF Skill 的描述说明，只要用户要求读取、合并、拆分或填写 PDF，就应使用该 Skill。模型可以依靠这段简短描述完成初步选择，而不必先读取全部说明书。

`tools` 数组同时提供一个名为 `Skill` 的工具。它的定义说明：可用 Skill 已列在 System Prompt 中；任务匹配某项 Skill 时，模型必须先调用该工具，再执行后续工作。

## 第二层：点名加载 `SKILL.md`

模型判断任务需要 PDF Skill 后，先返回一次 `tool_use`，调用参数指定 `document-skills:pdf`。这次调用的作用是向 Claude Code 索取该 Skill 的完整说明。

下一次请求中，工具结果本身写着：

```text
Launching skill: document-skills:pdf
```

`SKILL.md` 正文则作为同一条用户消息中的独立 Text Block 加入上下文。Claude Code 从本地读取原文件后会做三处处理：

1. 去掉 frontmatter，因为名称和描述已经出现在 Skill 清单中；
2. 在开头增加 `Base directory for this skill:`，指明 Skill 所在目录；
3. 在末尾增加 `ARGUMENTS:`，附上调用时传入的参数。

Base directory 使模型能够解析 `SKILL.md` 中引用的相对路径。案例中的说明书要求：高级 PDF 功能读取 `REFERENCE.md`，表单处理读取 `FORMS.md`；它还引用了同目录中的第三方脚本。模型取得目录后，才能准确定位这些资源。

## 第三层：只读取任务需要的附属资源

完整加载 `SKILL.md` 不等于加载整个 Skill 目录。案例中的 `REFERENCE.md` 与 `FORMS.md` 合计接近三万字符，但当前任务只需抽取论文文字，因此模型没有读取这两份参考文档，而是直接按照主说明运行 `pdftotext`，提取论文第 3～5 页并解释其中公式。

这种分层把 Skill 内容拆成三种可见范围：

1. 会话开始时，模型只看到名称与描述；
2. 选中 Skill 后，模型看到处理过的 `SKILL.md` 正文；
3. 主说明引用的参考文件与脚本，仅在当前任务需要时读取或执行。

与普通工具调用相比，Skill 加载只多一次模型请求：第一轮选择并索取说明书，下一轮取得说明书后才按其中的命令工作。

## 与 ToolSearch 的相同点和区别

Skill 与 ToolSearch 都先提供能力目录，再让模型点名索取完整说明。两者的初始目录不同：Skill 清单包含名称和描述；Deferred Tool 清单只提供名称，没有 Description 与 JSON Schema。

两者加载内容的位置也不同。`SKILL.md` 正文追加在消息末尾，不改变此前的请求前缀，Prompt Cache 可以继续命中。ToolSearch 取得的工具定义需要进入靠前的 `tools` 数组；服务端依靠 `defer_loading` 标记在计算缓存前缀时跳过新增定义，并通过消息中的 `tool_reference` 让模型使用它。

更重要的区别在于最终执行内容。ToolSearch 暴露的是 Claude Code 官方工具定义及其实现；Skill 则把第三方编写的提示词、参考材料和脚本交给模型，并可能使用用户的 Shell、文件和密钥。因此，Skill 的扩展性更强，信任边界也更宽。

## 第三方 Skill 必须先审计

官方简介记录了两个教学样例：一个伪装成 AWS 配置工具，实际尝试打包外传 AWS 凭据、SSH 私钥、`.env` 和 Shell 历史；另一个声称执行“加密归档”，实际会原地加密项目文件、删除原件并留下勒索信。

这些风险不是 Skill 加载机制自动消除的。`SKILL.md` 会作为普通文本进入上下文，其中引用的脚本也可能被执行。安装第三方 Skill 前，应完整阅读 `SKILL.md`，继续检查它要求读取的参考文件、脚本、文件范围、网络访问和凭据使用，再决定是否运行。

Skill 的核心价值是按需披露：用很短的目录完成发现，用主说明指导任务，再按需读取附属资源。这个机制减少了初始上下文，却不会替代来源审查、权限控制和执行隔离。
