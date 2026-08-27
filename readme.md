# Public Files

本项目用于托管一些公共资源文件（如通知图标、脚本等）。

## 使用方法

当你通过浏览器访问 GitHub 上的文件时，URL 是不可直接引用的预览链接。你需要将其转换为 **Raw** 链接才能在程序（如 Bark, Webhook）中使用。

### 转换规则

将 `github.com` 替换为 `raw.githubusercontent.com`，并去掉中间的 `/blob/`。

`https://raw.githubusercontent.com/kentxxq/public-files/main/[路径]/[文件名]`

> 如果你的网络不好，可以通过[免费的镜像站加速访问](https://demo.kentxxq.com/app/mirror)。

### 示例

- **浏览器链接**:  
  `https://github.com/kentxxq/public-files/blob/main/images/alert.png`
- **可直接引用地址 (Raw)**:  
  `https://raw.githubusercontent.com/kentxxq/public-files/main/images/alert.png`



## Skills 使用

本项目在 `.agents/skills/` 目录下托管了一些 Agent Skill，可通过 [skills](https://skills.sh/) 的 CLI 一键安装到你的 AI 编程助手。

### 可用 Skill 列表

| Skill 名称 | 说明 |
| --- | --- |
| `ken-release` | 发布更新流程：更新版本号、README、CHANGELOG 等文件，不做任何 git 操作 |
| `ken-commit-push` | 代码变更提交并推送到远端仓库 |
| `ken-apifox-mcp` | 调用 Apifox MCP 查询项目结构、定位接口、管理数据模型、创建/更新 HTTP 接口 |


### 安装全部 Skill

```text
vpx skills add kentxxq/public-files
```

### 按需安装单个 Skill

```text
vpx skills add kentxxq/public-files --skill <skill-name>

vpx skills add kentxxq/public-files --skill ken-commit-push --skill ken-release
```

