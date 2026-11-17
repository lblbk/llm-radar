# LLM Radar

Python 服务以常驻 Docker 容器运行在 NAS，容器内调度器定时抓取，本地 SQLite 保存主数据；抓取结果同步到 Cloudflare D1，TypeScript Worker 提供只读 API，React 展示模型与价格历史。后续可增加只在 NAS 局域网使用的配置界面。默认美元展示，支持人民币。

架构、抓取流程、同步、备份和容器部署见 [plan.md](plan.md)。目前处于方案阶段，尚未实现抓取器、镜像或网站。

## Python 抓取方式

常驻服务按计划时间触发抓取，也可手动运行 `llm-radar run-once`。每个来源配置指定一个适配器，由适配器定义请求和分页方式；Fetcher 优先使用 JSON/API，没有稳定 API 时解析 HTML，确实依赖 JavaScript 的来源使用可选浏览器镜像。响应按页保存为本地压缩 raw 数据，Parser 再将完整页面转换为统一的模型和报价记录。

来源结果通过分页、字段、价格单位和覆盖范围校验后，才会应用到 SQLite 业务数据，并排入 D1 同步队列；失败或不完整的来源保留旧数据，不参与下架判断。D1 保存公开数据；SQLite 备份恢复来源配置、运行记录和待同步数据，NAS 环境里的密钥需另行安全备份。当前方案尚未列出首批厂商来源和具体 endpoint；它们会在开始实现时写入 `apps/llm-radar/config/default-sources.json` 并配套脱敏 fixture。完整流程见 [plan.md](plan.md)。

计划使用 GitHub Actions 自动构建并发布多架构镜像，NAS 从 GHCR 拉取运行：

```text
ghcr.io/lblbk/llm-radar:v0.1.0
```

以上是预定地址与示例版本，首次发布成功后才可拉取。发布说明会给出具体版本和 digest；支持 linux/amd64、linux/arm64，部署固定版本，不依赖 latest。完整流程见 [GitHub Actions 与 GHCR 分发](plan.md#十四github-actions-与-ghcr-分发)。
