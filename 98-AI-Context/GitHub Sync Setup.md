# GitHub Sync Setup

当前状态：

- 本地 Git 已初始化。
- GitHub 远程仓库已配置：`https://github.com/snlkss/knowledge.git`
- 首次 Push 已完成：`master -> origin/master`
- GitHub CLI 未安装。
- 本地 Git 已配置 `http.version=HTTP/1.1`，用于避免 HTTPS 推送连接重置。

## 需要用户提供

已提供。

## 提供地址后要做的事

1. 日常修改后运行 `git status`。
2. 确认无误后提交。
3. 推送到 `origin/master`。
4. 确认 GitHub 页面能看到 Vault 文件。

## 注意

不要把 GitHub Token、密码、密钥写进知识库文件。
