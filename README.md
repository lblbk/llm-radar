# LLM Radar

Python 抓取任务以 Docker 运行在 NAS，本地 SQLite 保存主数据；Cloudflare D1 和 KV 保存公开数据副本，TypeScript Worker 提供只读 API 与受 Access 保护的快照发布入口，React 展示模型与价格历史。默认美元展示，支持人民币。

架构、数据模型、同步发布协议、容器运行和实施顺序见 [plan.md](plan.md)。目前处于方案阶段，尚未实现抓取器、镜像或网站。

计划使用 GitHub Actions 自动构建并发布多架构镜像，NAS 从 GHCR 拉取运行：

```text
ghcr.io/lblbk/llm-radar:v0.1.0
```

以上是预定地址与示例版本，首次发布成功后才可拉取。发布说明会给出具体版本和 digest；支持 linux/amd64、linux/arm64，部署固定版本，不依赖 latest。完整流程见 [GitHub Actions 与 GHCR 分发](plan.md#十四github-actions-与-ghcr-分发)。
