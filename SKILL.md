---
name: add-cf-opcvip
description: 把本机固定端口服务注册为 Servy 长期服务，并在 Cloudflare（opcvip.net）添加子域名+隧道回源的完整操作手册
---

# add-cf-opcvip：本地服务 → Servy 长期化 → CF 子域通道

把本机任意固定端口服务做成「开机自启 + 崩溃自愈 + 公网域名直达」的完整流程。
本手册以 opcvip.net 账号（Google 登录 datobig18@gmail.com，gh 账号 tcflying）的实际操作为准，
换服务/换子域时只需替换 {服务名}、{端口}、{子域} 三个占位。

## 前置事实（2026-10-06 实测）

- 本机已有**远端管理隧道** `devspace-cloudflare-tunnel` 以 Servy 服务常驻（token-file 模式，
  进程命令行 `cloudflared tunnel run --token-file C:\stage\ocx-tunnel.token`）。
  **加公共主机名必须走 CF 面板**（远端管理的 ingress 配置存在云端，本地 config.yml 无效）。
- `cloudflared tunnel login` 的证书下载从 login.cloudflareaccess.org 拉取会持续 EOF（直连与
  7890 代理都试过），**不要走授权流，直接用面板**。
- Zero Trust 已并入主面板：入口是 `dash.cloudflare.com/{账号ID}/one/networks/connectors`，
  隧道的公共主机名在「**已发布应用程序路由**」标签（不叫 Public Hostname）。
- 本机 UAC 已设「从不通知」：`Start-Process -Verb RunAs` 静默提权，不会弹窗。
- Servy 9.9 已装（`C:\Program Files\Servy\servy-cli.exe`），install/status/start 需要管理员；
  配置库 `C:\ProgramData\Servy\db\Servy.db`（sqlite 可只读直查，参数 SERVY_ENC 加密）。
- **PS5.1 坑**：无 BOM 的 UTF-8 .ps1 会被当 ANSI 读，中文注释直接炸语法。
  修复：`[IO.File]::WriteAllText($p,$c,(New-Object Text.UTF8Encoding $true))` 补 BOM。

## 第一步：服务端口固定 + Servy 长期化

模板脚本：`G:\mmx-project\zcode web\deploy\zrelay\servy-install.ps1`（自提权，改占位即可复用）。
要点（每个服务五要素缺一不可）：

1. **端口写死在服务定义里**：exe/参数/env 三处显式钉死（如 `ZRELAY_PORT=4703`、`PORT=3030`），
   不依赖默认值漂移。
2. env 用 `--envVars "K=V;K2=V2"` 传递，值内 `"` 转义为 `\"`；含中文的 .ps1 必须带 BOM。
3. 安装：`servy-cli install -n {服务名} -p "C:\Program Files\nodejs\node.exe"
   --startupDir {cwd} --params {入口js} --envVars ... --startupType Automatic
   --enableHealth --recoveryAction RestartProcess --maxRestartAttempts 0`（0=无限重启）。
4. **端口交接顺序（关键，防空转）**：服务装好后其进程会因端口被手动实例占用而 EADDRINUSE
   退出，靠无限重启兜着；此时**杀掉手动实例**，服务在 ~10s 内自动重绑。
   验证归属：`netstat -ano | findstr :{端口}` 拿 PID → 查父进程应为 `Servy.Service.CLI.exe`。
5. 验证：`servy-cli status -n {服务名}` = Running；`curl 127.0.0.1:{端口}/健康检查路径` 200；
   开机自启 = StartMode Auto。

## 第二步：CF 添加子域名 + 通道回源

1. 浏览器打开 `https://dash.cloudflare.com/`（datobig18 Google 登录态在本机 Chrome
   Default profile）。
2. 进入 `dash.cloudflare.com/{账号ID}/one/networks/connectors`（账号ID 在隧道详情页可见；
   opcvip.net 账号为 `b0648b2dfb3456fceb1a7344019d029d`）。
3. 点目标隧道名（如 `devspace-opcvip-local`，状态=正常）→「**已发布应用程序路由**」标签。
4. 「+ 添加已发布应用程序路由」：
   - 子域名 = {子域}（如 `zcode1`）；域 = `opcvip.net`；路径留空（匹配所有）；
   - 服务类型 = **HTTP**；URL = `localhost:{端口}`（如 `localhost:3030`）；
   - 保存 → toast「设置已成功保存」，**DNS 自动配置**，无需手动建记录。
5. 改已有路由：点**主机名链接本身**直接进编辑页（行尾 ⋯ 菜单点击不可靠）。

## 自动化操作浏览器的纪律（Computer Use）

- CF 面板页 a11y 树会裁剪（400 元素）且窗口反复被最小化：**同一个工具调用单元内完成
  「还原窗口 → 动作 → 截屏」**，跨单元状态丢失。
- Chrome 136+ 默认 profile 禁 `--remote-debugging-port`；拷贝 profile 出副本会因
  app-bound 加密解不开 Cookie（登录态丢失）。**正解 = 直接操作主上真实 Chrome 窗口**。
- 键盘事件要求窗口在前台：先点一个页面元素把窗口带到前台，再 Ctrl+L 输 URL。
- 下拉选择：点开下拉后**键盘输入过滤词再点选项**，比坐标猜选项稳。

## 第三步：端到端验证清单

```
curl -s -o /dev/null -w "%{http_code}" https://{子域}.opcvip.net/          # 200（页面）
curl -s https://{子域}.opcvip.net/{健康检查路径}                          # 服务 JSON
curl -s -D - -o /dev/null -H "Accept-Encoding: gzip" https://{子域}.opcvip.net/{主JS}
# content-encoding: gzip + cache-control: public, max-age=31536000, immutable
```

浏览器真实打开：页面渲染、核心交互可用、无崩溃横幅。

## 已知坑位速查

| 坑 | 处置 |
| --- | --- |
| .ps1 中文注释炸语法 | 补 UTF-8 BOM |
| cloudflared login 证书 EOF | 弃授权流，走面板 |
| 行菜单 ⋯ 点不开 | 点主机名链接进编辑页 |
| 窗口最小化丢动作 | 同单元「还原→动作→截屏」 |
| 端口被手动实例占用 | 杀手动实例，服务自动重绑（无限重启兜底） |
| 版本门拦截（版本不一致） | 页面构建版本必须与桌面/链接 app_version 一致（如 3.14.3） |
| web 静态面无压缩/泄露 map | server 侧接 hono/compress + Vary；.map 一律 404；vite sourcemap:false |

## 本机当前已配好的对照表（2026-10-06）

| 域名 | 回源 | 服务 | Servy 服务名 |
| --- | --- | --- | --- |
| zcode1.opcvip.net | http://localhost:3030 | zcode Web 工作界面（@zcode/server） | zcode-web-server |
| （本地）4703 /m | http://localhost:4703 | zrelay 手机远控配对页 | zcode-zrelay |
