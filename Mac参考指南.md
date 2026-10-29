> 历史参考：人格和管理页面描述可能过时，请以 [部署流程](部署流程.md) 和当前代码为准。

# 给 DSH 接 QQ：从零复刻指南（macOS）

> 目标：让别人在自己的 Mac 上，把 DeepSeek Harness 接成 QQ 机器人。
> 全程约 20 分钟，其中一半时间在等扫码和 QQ 启动。
>
> 本机实测环境：macOS 26.6.2（Apple Silicon）· QQ 7.0.1 (52892) · NapCat 4.18.41 · dsh-onebot-qq 0.9.6 · DSH 桌面版。

---

## 0. 你需要准备

| 东西 | 说明 |
|---|---|
| 一个**小号 QQ** | 当机器人用。必须和日常号不同，否则「自己和自己聊天」会被插件忽略 |
| 日常 QQ | 用来和机器人聊 |
| QQ for Mac | 装到 `/Applications/QQ.app` |
| DeepSeek Harness 桌面版 | 插件宿主 |
| NapCat-Mac-Installer | 给 QQ 注入 NapCat（GitHub: NapNeko/NapCat-Mac-Installer） |

---

## 1. 装 NapCat（把 QQ 变成 OneBot 网关）

1. 下载并打开 **NapCatInstaller.app**，选最新 NapCat 版本安装。
2. 点「**切换程序入口 NapCat**」。
   - 如果系统提示需要授权：系统设置 → 隐私与安全性 → **App 管理** → 允许 NapCatInstaller。
3. 此时 `/Applications/QQ.app/Contents/Resources/app/package.json` 的 `main` 会被改成
   `~/Library/Containers/com.tencent.qq/Data/Documents/loadNapCat.js`。
4. **先不要启动 QQ**，等第 3 步 DSH 配好再一起启动。

---

## 2. 装 DSH 插件 `dsh-onebot-qq`

```sh
# DSH_HOME 默认 ~/.dsh，桌面版 profile 名是 desktop
dsh plugin --profile desktop add dsh-onebot-qq
```

或在 DSH 的 Plugins 页面里装。装完确认：

```sh
cat ~/.dsh/profiles/desktop/package.json
# dependencies 里应有 "dsh-onebot-qq": "^0.9.6"
# dsh.profile.bundles 里应有 "dsh-onebot-qq"
```

生成一个 token 备用：

```sh
openssl rand -hex 32
```

---

## 3. 写 DSH 侧配置

编辑 `~/.dsh/profiles/desktop/cordis.patch.yml`，在文件末尾追加：

```yaml
- id: dsh-onebot
  config:
    mode: reverse                       # NapCat 拨入
    host: 127.0.0.1
    port: 8644
    accessToken: "<上一步生成的 64 位 hex>"
    adminUsers: ["<你的日常QQ>"]
    dmPolicy: allowlist
    allowFrom: ["<你的日常QQ>"]
    groupPolicy: allowlist
    groupAllowFrom: []                  # 有群号后填
    requireMention: false
    allowAllUsers: false
    workspacePath: /Users/<你>/qq-workspace   # QQ 对话的工作目录
    interimMessages: false
```

重启 DSH（或改完等它热重载）。验证：

```sh
lsof -nP -iTCP:8644 -sTCP:LISTEN
# 应当有 DeepSeek 进程监听 127.0.0.1:8644
```

用错 token 握手会返回 `HTTP/1.1 401 unauthorized`，说明插件已生效。

---

## 4. 启动 QQ 并登录机器人号

**必须从普通终端启动**（不能在 DSH / 任何被沙箱的进程里启动，原因见「坑 1」）：

```sh
/Applications/QQ.app/Contents/MacOS/QQ --no-sandbox
```

- 首次会弹二维码：**用当机器人的小号扫**，手机确认登录。
- 终端日志出现 `[WebUi] WebUi Token: xxxx` 和 `http://127.0.0.1:6099/webui?token=...` 即 NapCat 起来了。
- 二维码也可从 `~/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/NapCat/cache/qrcode.png` 取。

登录成功标志：

```sh
lsof -nP -iTCP:6099 -sTCP:LISTEN     # NapCat WebUI
```

---

## 5. 给 NapCat 配「反向 WebSocket 客户端」

⚠️ **最容易配错的一步**：NapCat 真正读取的配置在
`~/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/NapCat/config/`
**不是** `~/Library/Containers/.../Documents/napcat/config/`（后者是安装器目录）。

登录后该目录会生成 `onebot11_<机器人QQ>.json`。两种改法：

### 方法 A：改文件后重启 QQ

把文件改成（保留其余字段）：

```json
{
  "network": {
    "httpServers": [], "httpSseServers": [], "httpClients": [],
    "websocketServers": [],
    "websocketClients": [
      {
        "name": "dsh",
        "enable": true,
        "url": "ws://127.0.0.1:8644/ws",
        "messagePostFormat": "array",
        "reportSelfMessage": false,
        "reconnectInterval": 3000,
        "token": "<与 accessToken 完全一致>",
        "debug": false,
        "heartInterval": 30000,
        "verifyCertificate": true
      }
    ],
    "plugins": []
  },
  "enableLocalFile2Url": true,
  "parseMultMsg": true
}
```

然后重启 QQ（`--no-sandbox`），登录用快速登录，不用再扫码。

### 方法 B：走 WebUI API（不用重启）

```sh
TOKEN="<终端里的 WebUi Token>"
HASH=$(printf '%s.napcat' "$TOKEN" | shasum -a 256 | cut -d' ' -f1)
CRED=$(curl -s -X POST http://127.0.0.1:6099/api/auth/login \
  -H 'Content-Type: application/json' -d "{\"hash\":\"$HASH\"}" \
  | python3 -c 'import json,sys;print(json.load(sys.stdin)["data"]["Credential"])')

# config 字段是「OneBot 配置的 JSON 字符串」
curl -s -X POST http://127.0.0.1:6099/api/OB11Config/SetConfig \
  -H "Authorization: Bearer $CRED" -H 'Content-Type: application/json' \
  -d "{\"config\": $(python3 -c 'import json;print(json.dumps(open("/tmp/onebot11.json").read()))')}"
```

> 注意路由是 `/api/OB11Config/SetConfig`，不是 `/api/OB11Config`。

成功后 NapCat 日志出现：

```
[Notice] [OneBot11] 配置变更后: WebSocket反向服务: ws://127.0.0.1:8644/ws, : 已启动
```

DSH 侧同时能看到 ESTABLISHED：

```sh
lsof -nP -iTCP:8644 | grep ESTABLISHED
```

---

## 6. 验证

1. 从日常号私聊机器人一句「你好」。
2. NapCat 日志应出现 `接收 <- 私聊 (你的号) 你好` 与 `发送 -> 私聊 ...`。
3. DSH 会为这个聊天建持久会话：
   `~/.dsh/sessions/<工作区编码>/onebot-private-<QQ号>/`
4. 群里说话（若配了群）同理，`groupPolicy=allowlist` 必须把群号写进 `groupAllowFrom`。

---

## 7. 四个必踩的坑（本机实测）

### 坑 1：从 DSH 沙箱里启动 QQ → 秒崩

症状：`EXC_BREAKPOINT` / `libsystem_secinit` / `SYSCALL_SET_USERLAND_PROFILE`。
原因：QQ 自带 App Sandbox，被外层沙箱嵌套后初始化失败。
**解决**：只在 Terminal / Finder 里启动，命令带 `--no-sandbox`。

### 坑 2：只签外层 → Helper 加载框架被拒

症状：`dlopen QQNT.framework ... different Team IDs`，或 `QQ Helper` 起不来。
原因：改过 `package.json` 后系统校验签名，外层被 ad-hoc 重签、嵌套框架仍是腾讯签名。
**解决**：整包深签（保留 entitlements）：

```sh
codesign --force --deep --sign - \
  --preserve-metadata=entitlements,flags,runtime /Applications/QQ.app
codesign --verify --deep --strict /Applications/QQ.app   # 应为 valid on disk
```

### 坑 3：NapCat 的 utility worker 被系统 SIGKILL（退出码 9）

症状：`Worker进程退出，退出码: 9`，连续 3 次后 NapCat 主进程退出。
原因：NapCat 在 utility 进程里 hook QQ 内核代码，ad-hoc + hardened runtime 禁止改可执行页（崩溃报告写 `CODESIGNING / Invalid Page`）。
**解决**：给四个 Helper 补 entitlements 再深签：

```sh
cat > /tmp/helper.ent.plist <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>com.apple.security.app-sandbox</key><true/>
  <key>com.apple.security.cs.allow-jit</key><true/>
  <key>com.apple.security.cs.allow-unsigned-executable-memory</key><true/>
  <key>com.apple.security.cs.disable-executable-page-protection</key><true/>
  <key>com.apple.security.cs.disable-library-validation</key><true/>
  <key>com.apple.security.inherit</key><true/>
</dict></plist>
EOF
for h in "QQ Helper.app" "QQ Helper (GPU).app" "QQ Helper (Plugin).app" "QQ Helper (Renderer).app"; do
  codesign --force --deep --sign - --entitlements /tmp/helper.ent.plist \
    --preserve-metadata=flags,runtime "/Applications/QQ.app/Contents/Frameworks/$h"
done
codesign --force --deep --sign - \
  --preserve-metadata=entitlements,flags,runtime /Applications/QQ.app
```

### 坑 4：配置写到了安装器目录

症状：`[Notice] [OneBot11] [network] 配置加载` 后面是空行、反向 WS 不启动。
原因：`Documents/napcat/config/` 只是安装器的工作目录，运行时读的是
`Library/Application Support/QQ/NapCat/config/onebot11_<uin>.json`。
**解决**：改真正的运行目录，见第 5 步。

---

## 8. 日常维护

| 事项 | 做法 |
|---|---|
| 关机重开 | 运行 `大肥鱼.command` → 选 1；或装菜单 4 的开机自启 |
| 看配置 / 改配置 | 控制台 http://127.0.0.1:8788 |
| QQ 自动更新后 | 入口会被还原、签名失效 → 重做第 1、7 步 |
| 消息不回 | 先看 `lsof -iTCP:8644` 有没有 ESTABLISHED，再看 NapCat 日志 |
| 机器人离线 | 重新扫码登录（`--no-sandbox` 启动后看终端二维码） |

---

## 9. 最小检查清单

```sh
lsof -nP -iTCP:8644 -sTCP:LISTEN      # 插件在听
lsof -nP -iTCP:8644 | grep ESTAB      # NapCat 已连上
lsof -nP -iTCP:6099 -sTCP:LISTEN      # NapCat WebUI 在
codesign --verify --deep --strict /Applications/QQ.app   # 签名完整
```

四项都过，QQ 通道就是通的。
