# backup

个人项目备份仓库 · GitHub: [zerotwo-debug](https://github.com/zerotwo-debug)

## 用途

作为个人项目的备份落脚点：每个子目录对应一个独立项目，或按需在本仓库内维护单一项目结构。

## 约定

- **不提交任何敏感信息**：密钥、令牌、证书、生产环境配置一律不入库
- 依赖目录与构建产物不入库，规则见 `.gitignore`
- 单文件超过 50 MB 请改用 Git LFS 或对象存储
- 备份是"最后一道防线"，重要项目仍建议保留独立仓库与异地副本

## 常用命令

```bash
git add -A
git commit -m "backup: <项目名> <说明>"
git push
```
