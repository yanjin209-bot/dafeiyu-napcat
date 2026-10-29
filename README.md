# 大肥鱼：NapCat + DeepSeek Harness

这是从 macOS 实际项目提取的源码和复刻需求，适合学习和继续开发。请先阅读 [复刻需求](复刻需求.md) 和 [Mac 参考指南](Mac参考指南.md)。

QQ → NapCat（OneBot 11）→ dsh-onebot-qq → DeepSeek Harness → QQ。

## 使用前

- 安装对应系统的 QQ、NapCat、DeepSeek Harness 和插件；自行生成连接令牌并在两端配置，使用白名单。
- `harness-qq/qq-console` 是原项目的管理页面源码，`harness-qq/大肥鱼.command` 是 macOS 启动脚本。请在本机配置好服务，再双击脚本启动。脚本需要系统已安装 Node.js 或另行准备对应内置运行时，首次使用请先在终端检查配置。
- `harness-qq/workspace/AGENTS.md` 是已去除个人账号的人格模板；按部署者需要修改角色和白名单。次记忆留在本地。
- Windows 复刻需求已列出，但当前控制台和启动脚本依赖 macOS 的 zsh、lsof、QQ 路径和进程管理，**尚未适配或测试 Windows**；请先移植再运行。
- 本仓库不包含 Node 运行时、node_modules、QQ 登录数据、模型凭据、消息、日志或个人文件；复制源码后仍需安装依赖和配置账号。不要上传密钥或个人聊天记录。
- 提供的是脱敏源码而非开箱即用的完整安装包；QQ 图片发送与各平台插件兼容性请实际测试。
