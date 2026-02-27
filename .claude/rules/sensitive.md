# 敏感文件处理

- frontmatter 中 `status` 含 `SENSITIVE` 或 tags 含 `sensitive` 的文件，仅在明确需要时读取
- `*_credentials.md` 包含凭据信息，已被 .gitignore 排除，不要主动加载
- `CLAUDE.local.md` 包含连接信息（非密钥），每次 session 自动加载，无需手动读取
