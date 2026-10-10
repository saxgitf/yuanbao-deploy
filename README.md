# yuanbao-deploy

在 Linux 机器上部署「呆呆面板(daidai-panel) + Cloudflare Tunnel 公网入口 + 元宝保活定时任务」的 SKILL。

开工先**只读体检**：看这台机器上是否已有隧道/面板，按**隧道归属**（用 API 查隧道 UUID 在不在本账号）决定复用还是新建 —— 绝不打断别人正在服务的隧道。

## 使用

    帮我安装 https://gh-proxy.com/https://raw.githubusercontent.com/saxgitf/yuanbao-deploy/refs/heads/main/yuanbao-deploy-single/SKILL.md 并开始执行。

如遇拦截，打开侧边栏点击新旁路会话发送。

直连（不走加速）：`https://raw.githubusercontent.com/saxgitf/yuanbao-deploy/refs/heads/main/yuanbao-deploy-single/SKILL.md`

## 自包含

单文件版 `yuanbao-deploy-single/SKILL.md`（约 131 KB）里就是全部内容，不依赖任何外部文件：

- **附录 A**：完整安装脚本（2000+ 行）—— 拿到后先落盘到 `/opt/daidai-panel/data/scripts/yuanbao_install.sh`
- **附录 B**：参数表
- **附录 C**：排错手册

## 关于密码

脚本里**不写死任何密码**。面板管理员密码在运行时确定：

1. 环境变量 `PANEL_PASS=xxx` 指定了就用它；
2. 没指定就随机生成 16 位，存到本机 `/opt/yuanbao-install/.panel_pass`（权限 600），重跑时复用；
3. 安装结束时打印出来给你。

体检模式（`CF_SURVEY_ONLY=1`）全程只读，不生成也不落盘密码。

## 组件

| 组件 | 落点 | 作用 |
| --- | --- | --- |
| 呆呆面板 | `/opt/daidai-panel`，systemd `daidai-panel.service`，端口 5700/5701（脚本自动探测） | 跑定时任务的宿主 |
| Cloudflare Tunnel | `/usr/bin/cloudflared` | 给面板 HTTPS 公网入口 |
| 保活脚本 | `/opt/daidai-panel/data/scripts/ybbh_keepalive.py` | 每天给元宝发消息，防会话被回收 |

## 特性

- 只读体检先行（`CF_SURVEY_ONLY=1`），全程不动环境
- 默认复用现有隧道；换域名只追加 ingress，不重建
- 面板端口自动探测（进程实际监听 → `config.yaml` → 默认 5700），体检匹配与 CF 映射都用真实端口
- 有 CF 凭据时自动建隧道 / 写 DNS / 追加 ingress（支持 API Token 与 Global API Key 两种认证）
- 分档卸载：只停本机 / 连云端隧道+DNS 删 / 全清（破坏性操作需显式开关）
- 下载内置 gh-proxy.com 等加速镜像，失败回退备用镜像与直连

## 铁律

1. 绝不重建正在服务别人的隧道（归属只能用 API 查隧道 UUID 判定，域名后缀会骗人）
2. 换域名 ≠ 换隧道，加一条 ingress 就够
3. 凭据只经环境变量传入，不落文档、不写日志
