# LLM Radar

Python 服务以常驻 Docker 容器运行在飞牛 NAS，定时抓取模型报价和能力评测。SQLite 保存主数据，公开记录按序同步到 Cloudflare D1，Worker 和 React 提供查询。报价默认展示美元，支持人民币。开发顺序为 Python 采集服务、NAS 局域网管理页面、Cloudflare 公开查询与展示。

架构、抓取流程、同步、备份和容器部署见 [plan.md](plan.md)。目前处于方案阶段，尚未实现抓取器、镜像或网站。

开始开发时，先看 [项目架构](docs/architecture.md) 确认应用边界和部署方式，再按 [开发手册](docs/development.md) 实现第一条真实采集链路。手册包含管理 API、前后端联调样例和各阶段的验收条件。

模型报价优先使用同档官方价。官方没有可用报价时，页面可展示核准的第三方参考价，并标明出处、核实时间和回退原因。能力评测区分厂商自测、第三方机构和个人，按标准版本、指标和测试条件展示原始结果。新增标准可复用已有适配器和字段映射。详见 [来源如何登记和使用](plan.md#来源如何登记和使用)。

## Python 抓取方式

常驻服务按计划时间触发抓取，也可手动运行 `llm-radar run-once`。每个来源配置指定一个适配器，由适配器定义请求和分页方式；Fetcher 优先使用 JSON/API，没有稳定 API 时解析 HTML，确实依赖 JavaScript 的来源使用可选浏览器镜像。响应按页保存为本地压缩 raw 数据，Parser 再将完整页面转换为统一的模型和报价记录。

来源结果通过分页、字段、价格单位和覆盖范围校验后，才会应用到 SQLite 业务数据，并排入 D1 同步队列；失败或不完整的来源保留旧数据，不参与下架判断。D1 保存公开数据；SQLite 备份恢复来源配置、运行记录和待同步数据，NAS 环境里的密钥需另行安全备份。当前方案尚未列出首批厂商来源和具体 endpoint；它们会在开始实现时写入 `apps/llm-radar/config/default-sources.json` 并配套脱敏 fixture。完整流程见 [plan.md](plan.md)。

计划使用 GitHub Actions 自动构建并发布多架构镜像，NAS 从 GHCR 拉取运行：

```text
ghcr.io/lblbk/llm-radar:v0.1.0
```

以上是预定地址与示例版本，首次发布成功后才可拉取。发布说明会给出具体版本和 digest；支持 linux/amd64、linux/arm64，部署固定版本，不依赖 latest。完整流程见 [GitHub Actions 与 GHCR 分发](plan.md#十四github-actions-与-ghcr-分发)。
