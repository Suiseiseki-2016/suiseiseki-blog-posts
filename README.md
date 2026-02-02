# 博客文章仓库

本仓库用于存放博客的**文章源**，仅包含 Markdown 文件（`.md`）。

- **用途**：推送到 GitHub 后，作为博客后端的「远程文章仓库」；后端通过 Webhook 或定期同步拉取此处内容并更新博客。
- **格式**：每篇 `.md` 建议带 Front-matter（`title`、`slug`、`summary`、`category`、`published_at`），详见博客项目根目录的 [CONFIG.md](../CONFIG.md) 第五节。
