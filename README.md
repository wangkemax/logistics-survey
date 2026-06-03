# 物流售前现场调研工具

> 售前体系中的「现场调研采集入口」——把现场的事记全记准，当场出图文调研报告。
> 测算（人数/成本/毛利/报价）由 Logistics Presale System 负责，本工具只管采集与出报告。

## 这是什么

一个移动端现场调研工具：现场边走边填、拍照记录、当场导出图文调研报告。报告可直接喂给 Logistics Presale System 的 NotebookLM 导入做后续测算。

## 定位边界

- ✅ 做：结构化字段采集、拍照 + AI 图注、图文报告导出（Markdown / ZIP）、本地自动保存
- ❌ 不做：人数/成本/毛利/报价测算、系统对接、云端账户（归 Presale System 或后续版本）

## 项目结构

```
logistics-survey/
├── README.md
├── CLAUDE.md            # 项目上下文与定位边界（先读）
├── docs/
│   ├── 立项说明.md      # 立项稿，方向与范围
│   └── v1.0-spec.md     # v1.0 开发规格与验收标准
└── tools/               # 调研工具 PWA
    ├── index.html       # 主程序（单文件，含全部逻辑）
    ├── manifest.webmanifest / sw.js / icon-*.png   # PWA 配套（可装、离线）
    ├── README.txt       # 部署与使用说明
    └── survey-pwa.zip   # 整套打包，便于一次性部署
```

## 状态

- ✅ 立项 — 见 `docs/立项说明.md`
- ✅ v1.0 工具 — 基于现场调研原型改造收口为通用采集器，见 `docs/v1.0-spec.md`

---

*售前/项目组*
