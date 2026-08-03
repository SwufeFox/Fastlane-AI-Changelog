# Fastlane AI Changelog Generator

用 AI 根据 Git 提交记录，自动生成适合 **Google Play** 的用户向更新说明（What's New），并直接写入 Fastlane 的 Android metadata 目录。

支持 OpenAI、DeepSeek、硅基流动、OpenRouter、Groq 等所有兼容 OpenAI Chat Completions 接口的模型。

生成的文件路径示例：

```
fastlane/metadata/android/zh-CN/changelogs/128.txt
fastlane/metadata/android/en-US/changelogs/128.txt
fastlane/metadata/android/zh-CN/changelogs/default.txt
```

---

## 目录

- [功能特点](#功能特点)
- [工作原理](#工作原理)
- [前置要求](#前置要求)
- [Inputs 参数说明](#inputs-参数说明)
- [使用示例](#使用示例)
  - [基础用法（OpenAI）](#1-基础用法openai)
  - [DeepSeek 推荐示例](#2-deepseek-推荐示例)
  - [手动指定提交范围](#3-手动指定提交范围)
  - [生成后自动提交到仓库](#4-生成后自动提交到仓库)
  - [配合 Fastlane 上传到 Google Play](#5-配合-fastlane-上传到-google-play)
- [生成效果示例](#生成效果示例)
- [注意事项与最佳实践](#注意事项与最佳实践)
- [常见问题排查](#常见问题排查)
- [License](#license)

---

## 功能特点

- **完整提取 Git 信息**：不只拿 commit 标题，还会提取完整的 commit message（标题 + 正文）。
- **智能范围选择**：
  - 不指定范围时，自动查找「上一个 tag」到当前 HEAD 的所有提交。
  - 找不到 tag 时，自动回退到最近 N 条提交。
  - 支持调用方手动指定 `from_ref` 和 `to_ref`。
- **严格字符控制**：生成结果强制限制在指定字符数内（默认 450，留出 Google Play 500 字符余量）。
- **多语言支持**：一次生成后，可同时写入多个 locale 目录。
- **高度兼容**：支持任意 OpenAI 兼容接口（DeepSeek、硅基流动、OpenRouter、Groq、本地 Ollama 等）。
- **干净输出**：只输出纯文本 bullet points，无 markdown 标题、无代码块、无多余前后文。

---

## 工作原理

1. 检查是否为浅克隆，必要时尝试 `git fetch --unshallow`。
2. 确定提交范围：
   - 优先使用调用方传入的 `from_ref`。
   - 否则自动执行 `git describe --tags --abbrev=0 HEAD^` 找上一个 tag。
   - 都找不到则取最近 `commit_count_fallback` 条提交。
3. 使用 `git log --pretty=format:"%s%n%b"` 提取**完整 commit message**（排除 merge commit 可选）。
4. 构建精心设计的 Prompt，要求 AI 输出用户向、简洁的更新说明。
5. 调用 AI 接口生成内容。
6. 检查并截断超出字符限制的内容。
7. 写入 `fastlane/metadata/android/<locale>/changelogs/<version_code>.txt`。

---

## 前置要求

### 1. 必须完整克隆仓库

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0          # 必须！否则无法正确获取历史 tag 和完整提交
```

如果不设置 `fetch-depth: 0`，`git describe` 和跨 tag 的 `git log` 会失败或结果不准确。

### 2. 准备 API Key

把对应的 API Key 添加到仓库 Secrets 中，例如：

- `OPENAI_API_KEY`
- `DEEPSEEK_API_KEY`
- `SILICONFLOW_API_KEY` 等

---

## Inputs 参数说明

| 参数名 | 必填 | 默认值 | 说明 |
|--------|------|--------|------|
| `openai_api_key` | ✅ | - | API Key（支持 OpenAI / DeepSeek / 硅基流动等） |
| `api_base_url` | ❌ | `https://api.openai.com/v1` | API 基础地址，**必须以 `/v1` 结尾** |
| `model` | ❌ | `gpt-4o-mini` | 模型名称 |
| `version_code` | ❌ | `default` | 生成的文件名。对应 Android 的 `versionCode`，也可写 `default` |
| `locales` | ❌ | `zh-CN` | 要写入的语言，逗号分隔，例如 `zh-CN,en-US,ja-JP` |
| `max_characters` | ❌ | `450` | 最大字符数限制（Google Play 官方限制 500，建议 ≤450） |
| `from_ref` | ❌ | （空） | 起始点（tag、commit SHA、branch）。不填则自动找上一个 tag |
| `to_ref` | ❌ | `HEAD` | 结束点，通常保持默认即可 |
| `commit_count_fallback` | ❌ | `20` | 找不到上一个 tag 且未指定 `from_ref` 时，取最近多少条提交 |
| `exclude_merges` | ❌ | `true` | 是否排除 merge commit（推荐保持 `true`） |

---

## 使用示例

### 1. 基础用法（OpenAI）

```yaml
name: Generate Fastlane Changelog

on:
  workflow_dispatch:
  push:
    tags:
      - 'v*'

jobs:
  changelog:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Generate AI Changelog
        uses: ./.github/actions/fastlane-ai-changelog
        with:
          openai_api_key: ${{ secrets.OPENAI_API_KEY }}
          version_code: ${{ github.run_number }}
          locales: 'zh-CN,en-US'
          max_characters: '450'
```

### 2. DeepSeek 推荐示例

DeepSeek 的 `deepseek-chat` 中文表现优秀、价格极低，强烈推荐。

```yaml
- name: Generate AI Changelog (DeepSeek)
  uses: ./.github/actions/fastlane-ai-changelog
  with:
    openai_api_key: ${{ secrets.DEEPSEEK_API_KEY }}
    api_base_url: 'https://api.deepseek.com/v1'
    model: 'deepseek-chat'                 # 也可使用 deepseek-reasoner
    version_code: '128'
    locales: 'zh-CN,en-US'
    max_characters: '450'
```

> 如使用硅基流动，把 `api_base_url` 改成 `https://api.siliconflow.cn/v1`，模型填对应名称即可。

### 3. 手动指定提交范围

适合从某个具体版本到当前的情况：

```yaml
- name: Generate AI Changelog
  uses: ./.github/actions/fastlane-ai-changelog
  with:
    openai_api_key: ${{ secrets.DEEPSEEK_API_KEY }}
    api_base_url: 'https://api.deepseek.com/v1'
    model: 'deepseek-chat'
    version_code: '130'
    locales: 'zh-CN'
    from_ref: 'v1.4.2'      # 从上一个正式版 tag 开始
    to_ref: 'HEAD'
```

### 4. 生成后自动提交到仓库

```yaml
- name: Generate AI Changelog
  uses: ./.github/actions/fastlane-ai-changelog
  with:
    openai_api_key: ${{ secrets.DEEPSEEK_API_KEY }}
    api_base_url: 'https://api.deepseek.com/v1'
    model: 'deepseek-chat'
    version_code: ${{ github.run_number }}
    locales: 'zh-CN,en-US'

- name: Commit generated changelogs
  run: |
    git config user.name "github-actions[bot]"
    git config user.email "github-actions[bot]@users.noreply.github.com"
    git add fastlane/metadata/android/*/changelogs/
    git diff --staged --quiet || git commit -m "chore: update Fastlane changelogs via AI [skip ci]"
    git push
```

### 5. 配合 Fastlane 上传到 Google Play

在生成 changelog 后，直接调用 Fastlane 的 `upload_to_play_store`：

```yaml
- name: Upload to Google Play
  run: bundle exec fastlane supply
  env:
    # 你的 service account 等配置
```

Fastlane 会自动读取刚才生成的 `changelogs/<versionCode>.txt` 文件。

---

## 生成效果示例

**输入的 Git 提交**（完整 message）：

```
feat: 新增深色模式
支持跟随系统主题，并优化了夜间模式下的阅读体验

fix: 修复登录闪退
在部分 Android 13 设备上点击登录按钮会偶发崩溃

chore: 更新依赖库
```

**AI 最终生成的内容**（写入 txt 文件）：

```
- 新增深色模式支持，可跟随系统自动切换
- 修复部分机型登录时偶发闪退的问题
- 优化夜间模式下的阅读体验
```

---

## 注意事项与最佳实践

1. **字符限制**  
   Google Play 官方限制为 **500 字符**。建议设置 `max_characters: 450`，给 AI 留一点缓冲。

2. **version_code 的使用**  
   - 正式发布时，强烈建议传入真实的 `versionCode`（例如 `128`），这样 Play 控制台会正确关联到对应版本。
   - 仅测试时可使用 `default`。

3. **语言处理**  
   当前版本是「生成一份内容，然后复制到所有 locale」。  
   Prompt 会尽量根据 commit 语言自动选择中文或英文。如果需要真正的多语言分别生成，可后续扩展。

4. **模型推荐**  
   - 中文项目优先：`deepseek-chat`（性价比极高）
   - 需要更强推理：`deepseek-reasoner` 或 `gpt-4o`
   - 追求便宜：硅基流动的各类模型

5. **安全性**  
   API Key 一定要通过 GitHub Secrets 传入，不要硬编码在工作流文件中。

---

## 常见问题排查

| 问题 | 可能原因 | 解决方法 |
|------|----------|----------|
| `No commits found` | 浅克隆或范围错误 | 确认 `fetch-depth: 0`，检查 `from_ref` 是否正确 |
| AI 返回空内容 | API Key 错误 / 额度不足 / 模型名称错误 | 检查 Secrets、模型名、账户余额 |
| 生成内容超过字符限制 | AI 未严格遵守 | 已内置截断逻辑，可适当降低 `max_characters` |
| 中文效果不好 | 使用了不擅长中文的模型 | 换成 `deepseek-chat` |
| 文件没有生成 | 路径权限或 locale 写错 | 检查 `locales` 格式是否为 `zh-CN,en-US`（无空格或有空格均可） |

---

## License

MIT
