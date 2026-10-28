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
| `design` | 编写和维护设计文档，记录系统架构、模块职责、数据流、业务规则和关键决策 |
| `review` | 代码审查：检查代码与设计的一致性、性能与架构问题、代码质量与可维护性 |
| `test` | 设计与执行测试，验证功能行为、回归情况与测试缺口 |
| `refactor` | 在不改变外部行为的前提下重构代码，降低复杂度、去重与优化结构 |
| `audit` | 串联 design / review / test / refactor 做系统性项目巡检，默认只读 |
| `ui-explore` | 探索并对比多种真实代码的 UI 方案，改善布局、信息组织与交互体验 |
| `ken-commit-push` | 提交并推送代码；用户主动要求发版时，集成版本号递增、CHANGELOG/README 更新与打标签 |
| `ken-apifox-mcp` | 调用 Apifox MCP 查询项目结构、定位接口、管理数据模型、创建/更新 HTTP 接口 |


### 安装全部 Skill

```shell
vpx skills add kentxxq/public-files
```

### 按需安装单个 Skill

```shell
vpx skills add kentxxq/public-files --skill <skill-name>

# 全局用的
vpx skills add kentxxq/public-files -s ken-commit-push
vpx skills add kentxxq/public-files --skill ken-commit-push

vpx skills add kentxxq/public-files -s ken-commit-push -g -a zed
```

> 配合使用 -g 和 -a 。20260827实测codex app也会读取`~/.agents/skills/`目录下的Skill。
> -g 参数：安装到全局目录
> -a 参数：安装到特定agent。codex(~/.codex/skills/), zed(~/.agents/skills/)

### 常用

```shell
# ai项目用的
vpx skills add kentxxq/public-files -s design -s review -s test -s refactor -s ken-commit-push
```