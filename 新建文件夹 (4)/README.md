# DeepSeek Harness Evidence Skill Pack

这是可复制/归档的 DeepSeek Harness skill pack，包含两个本地 skill：

- `evidence-grounded-answer-loop`：证据追踪、可比性分级、完成门、代码检查状态和科学数据审计。
- `deepseek-v41-flash-adapter`：DeepSeek-V4.1-Flash-1017 的 thinking、tool call、JSON、vision、长上下文和 smoke-test 适配。

## 当前安装位置

本机 DeepSeek Harness 的实际加载位置为：

```text
C:\Users\高茂源\.dsh\skills\
```

本目录是备份/便携副本；两份 skill 已同时安装到上述 Harness 用户 skill 根目录。

## 手动安装

在 PowerShell 中从本目录执行：

```powershell
Copy-Item -Recurse -Force .\evidence-grounded-answer-loop "$env:USERPROFILE\.dsh\skills\"
Copy-Item -Recurse -Force .\deepseek-v41-flash-adapter "$env:USERPROFILE\.dsh\skills\"
```

安装后新建 Harness 会话，使 skill catalog 重新扫描。API key 不包含在此目录中；请在 Harness 的 `Settings → Models` 中配置。

## 验证边界

- 文件和 frontmatter 已安装。
- DeepSeek Harness `0.2.0-rc.2` 已确认启用 `dsh-skill-filesystem` 和 `dsh-tool-skill`。
- 实际 API smoke test 仍需要 DeepSeek API key；密钥不保存到本目录。
