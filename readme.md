# Public Files

本项目用于托管一些公共资源文件（如通知图标、脚本等）。

## 使用方法

当你通过浏览器访问 GitHub 上的文件时，URL 是不可直接引用的预览链接。你需要将其转换为 **Raw** 链接才能在程序（如 Bark, Webhook）中使用。

### 转换规则

将 `github.com` 替换为 `raw.githubusercontent.com`，并去掉中间的 `/blob/`。

### 示例

- **浏览器链接**:  
  `https://github.com/kentxxq/public-files/blob/main/images/alert.png`
- **可直接引用地址 (Raw)**:  
  `https://raw.githubusercontent.com/kentxxq/public-files/main/images/alert.png`

---

你可以直接复制下面的代码块中的地址并根据需要修改路径：

```text
https://raw.githubusercontent.com/kentxxq/public-files/main/[路径]/[文件名]
```