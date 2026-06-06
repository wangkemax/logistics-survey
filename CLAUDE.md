# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# 项目上下文（请先读这份）

## 这是什么

一个**移动端现场调研工具**（PWA）：售前/BD 去 3PL 仓储项目现场时，手机边走边填、拍照记录，当场导出图文调研报告。报告可直接喂给 Logistics Presale System 的 NotebookLM 导入做后续测算。

v1.0 已交付（2026-06-03）：原型已落地为 `tools/index.html` 单文件 PWA。

## 最重要：定位边界（不要越界）

本工具**只负责采集与出报告**。下面这条线不能模糊：

```
现场调研采集器（本工具）──出图文报告──► 人看 + 喂给 Logistics Presale System 做测算
```

### ✅ 要做
- 结构化字段采集（仓储货物、用工现状、瓶颈、空间结构、现场观察等客观事实）
- 拍照记录 + AI 图注（核心能力，重点打磨）
- 一键导出图文报告（Markdown 单文件 + 图文 ZIP）
- 本地自动保存（localStorage + IndexedDB），断网可用

### ❌ 绝对不要做（这些归 Logistics Presale System，不在本工具范围）
- **任何计算**：人数、用工成本、毛利、报价、投资回收期、ROI
- 与 Presale System 的接口对接（用户复用其 NotebookLM 导入能力，本工具只输出 NotebookLM 友好的报告即可）
- 云端账户、多人协作、服务端

> 商务/预算类字段（计费方式、预算、报价时间、回收期期望等）**只作纯文本记录**，绝不加任何运算逻辑。若发现自己在写计算函数，停下——那是另一个系统的职责。v1.0 已确认全工具无任何业务测算函数，新增改动须保持这一点。

## 技术约束

- **单文件 HTML**：核心程序就是 `tools/index.html` 一个文件——HTML + CSS + 原生 JS 全部内联，方便手机直接打开、加主屏、离线用。**所有逻辑改动都收在这个文件内**。
- **无构建步骤、无依赖**：不引入打包工具、不依赖 npm 运行时。无 package.json、无测试框架、无 lint 配置。验证靠浏览器手动走查。
- **移动端优先**：手机浏览器为主要场景，触控友好。
- **数据只存本地**：文字存 `localStorage`，照片存 `IndexedDB`；任何数据不上传服务器（除非用户主动点 AI，照片才发到其自配的 API）。
- **AI 图注**：调用用户自己在「设置」里填入的 OpenAI 兼容 API（key 仅存本机浏览器，不上传）；无网络时手填图注照常工作，不能阻塞调研。
- **UI 中文**，**提交信息用中文**，简明说明改了什么。

## 仓库结构

```
logistics-survey/
├── CLAUDE.md            # 本文件
├── README.md            # 项目总览
├── docs/
│   ├── 立项说明.md      # 立项稿，方向与范围
│   └── v1.0-spec.md     # v1.0 开发规格与验收标准（含完成状态表，改动前先看）
├── deploy/              # NAS / Docker 部署（nginx 静态 + Lucky 反代上 HTTPS）
│   ├── docker-compose.yml / Dockerfile / nginx.conf
│   └── README.md
├── tools/               # 调研工具 PWA（部署单位就是这个目录）
│   ├── index.html       # ★ 主程序，全部逻辑在此
│   ├── sw.js            # Service Worker（离线缓存）
│   ├── manifest.webmanifest / icon-192.png / icon-512.png  # PWA 配套
│   ├── README.txt       # 面向使用者的部署与使用说明
│   └── survey-pwa.zip   # 整套打包，便于一次性部署
└── .claude/launch.json  # 本地预览配置（python http.server，端口 8765）
```

## 代码架构（`tools/index.html`）

整个程序是一段内联脚本（约 640 行）。理解它的关键是这几条主线：

- **SCHEMA 驱动 UI 与报告**：顶部 `const SCHEMA` 是 7 大板块的声明式定义（`01 项目基础信息 … 07 结论与下一步`），每个字段有 `id / t（text|area|date|radio|multi）/ label / opts / hint`。UI 渲染、进度统计、报告生成全部从 SCHEMA 派生。**改字段从这里开始**——通用化时主要改 `opts` 覆盖汽配/工程机械/电商/医药/快消，别堆长尾。
- **关注点（custom / fp）**：每个板块下用户可动态增删「关注点」卡片（`state.__custom[si]`，每项有稳定 `id`，前缀 `fp`）。照片可绑到板块或具体关注点。
- **两套存储，分开持久化**：
  - 文字状态 `state` → `localStorage`，KEY = `logistics_survey_v1`。`load()` 会从 `LEGACY_KEYS`（`autoparts_survey_v3`）自动迁移旧数据——**改 SCHEMA / KEY 时务必保持向后兼容**，别让用户已填数据丢失。
  - 照片 → `IndexedDB`（库 `survey_photos_db`，store `photos`）。每条记录含压缩图 blob、可选原图 `orig`、`sec`（板块号，`null`=待归类）、`fpId`、`caption`、`aiDesc`、`ord`（重排序号）。
- **照片归类三入口**：板块内拍照（自动归该板块）、关注点 📷（绑 fpId）、全局 FAB 📷（落「待归类」inbox）。`▦` 全部照片浮层可改归属/图注/删除。`ord` 控制卡片与报告内顺序，`reorderPhoto` 实现 `‹ ›` 重排。
- **AI（可选、可降级）**：`aiCfg/aiReady/chatUrl/aiChat` 走用户配的 OpenAI 兼容 `/chat/completions`。`runAI` 生成单图中文图注并建议归类；`genInsight` 汇总「AI 综合洞察」。`stripThink()` 剥离推理模型的 `<think>…</think>`。无网/未配置时全部跳过，手填照常。模型须支持图片输入（vision）。
- **报告生成**：`buildMD(mode, imgFiles)` 是核心，`mode='embed'` 内嵌 base64 出单文件 `.md`，`mode='ref'` 用相对路径配合 ZIP。空板块/空字段不输出（NotebookLM 友好）。`exportMD` / `exportZIP` / `openPreview` / `copyMD` 是出口。
- **自包含 ZIP**：`makeZip` + `crc32` + `dosDT` 是手写的 store-only ZIP 编码器（无依赖），同样思路 `unzipStored` 用于备份恢复。
- **备份/恢复**：`doBackup(includeOrig)` 导出 `.zip`（meta `ver:4`，含 state + 照片元数据 + 可选原图），`doRestore` 还原。改照片数据结构时同步升 `ver` 并兼容旧备份。
- **离线 / PWA**：`sw.js` 缓存外壳（`CACHE='logistics-survey-v2'`，同源缓存优先+后台更新）。**改了需要立即对用户生效的外壳资源时，记得 bump `CACHE` 版本号**。

## 本地预览 / 验证

无构建、无测试框架。验证 = 浏览器手动走查（关键路径见下）。

```bash
# 起本地静态服务（.claude/launch.json 用的就是这条）
python -m http.server 8765 --directory tools
# 浏览器开 http://localhost:8765/  （桌面可用 DevTools 移动模拟）
```

> 注意：相机 / IndexedDB / Service Worker 需要**安全上下文**。`localhost` 可用；真机测试需 HTTPS（见 `deploy/README.md`）。

每次涉及核心路径的改动，走一遍验收：
1. 填字段 → 拍照/选图 → 写图注（含 AI）→ 调归类/重排 → 导出 Markdown / ZIP / 预览。
2. **关网络再走一遍**，确认离线可用、AI 图注优雅降级为手填。
3. 若动了 SCHEMA / 存储结构：用旧 KEY 数据或旧备份验证迁移不丢数据。

## 部署

- **手机直接用**：把 `tools/` 整目录放到任意 HTTPS 站点；或解压 `survey-pwa.zip`。
- **NAS / Docker**：见 `deploy/README.md`——nginx 静态服务（默认端口 `8327`），前面用 Lucky 等反代成 HTTPS。`tools/` 只读挂载。

## 约定

- 改动尽量内聚在 `tools/index.html` 单文件内；保持现有视觉风格的一致性，不要换主题。
- 数据结构（SCHEMA / 照片记录 / 备份格式）改动要兼容已存数据：做好 KEY 版本与迁移。
- 提交信息用中文，简明说明改了什么。
- 开发分支：`claude/...`；未经明确许可不要推到其他分支。
