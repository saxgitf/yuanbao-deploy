---
name: yuanbao-deploy-single
description: "在 Linux 机器上部署「呆呆面板(daidai-panel) + Cloudflare Tunnel 公网入口 + 元宝保活定时任务」：开工先只读体检这台机器上是否已有隧道/面板，按隧道归属决定复用还是新建（绝不打断别人正在服务的隧道），有 API 凭据时自动建隧道、写 DNS、追加 ingress、装面板、落保活脚本并注册面板定时任务，最后按判据验收；也支持分档卸载。全部能力封装在 scripts/yuanbao_install.sh，AI 只负责按本手册调度，不要自己重写安装逻辑。本版为单文件自包含版：安装脚本完整内嵌在附录 A，参数表与排错手册在附录 B/C，装到机器上只要这一个文件，不需要再单独下载任何东西。触发词：装面板、元宝保活、建隧道、daidai-panel、批量装机、卸载 cloudflared、面板上公网。"
version: 1.0.0
license: MIT
metadata:
  tags: [deploy, cloudflare, tunnel, daidai-panel, keepalive, uninstall, cron]
---

# 元宝保活一键部署手册

## 一、这套东西是什么

在一台干净的 Linux 机器上搭出「面板有公网入口 + 元宝会话每天自动保活」：

| 组件 | 落点 | 作用 |
| --- | --- | --- |
| 呆呆面板 | `/opt/daidai-panel`，systemd `daidai-panel.service`，端口默认 `5700` | 跑定时任务的宿主，保活脚本挂在它上面 |
| Cloudflare Tunnel | `/usr/bin/cloudflared`，systemd `cloudflared` | 给面板一个 HTTPS 公网入口（本手册的重点与主要风险点） |
| 保活脚本 | `/opt/daidai-panel/data/scripts/ybbh_keepalive.py` | 每天给元宝发一条消息，防止会话容器被回收 |
| 面板定时任务 | 面板内 cron 任务（名含「保活」） | 每天定时触发保活脚本 |

**已经验证过的真实环境特征**（写手册时实测，不是推测）：

- 一台机器上常常**已经跑着别人的隧道**（token 远程托管模式），绑着正在服务的域名。重建隧道 = 那些域名当场断线，且不可自动恢复。
- 换域名**不需要**重建隧道，追加一条 ingress 即可。
- 隧道归属的唯一可靠判据是**用 API 查 UUID 在不在这个账号的隧道列表里**；域名后缀、本机记录都可能骗人。

> **单文件版说明**：本文件自包含 —— 安装脚本（2000+ 行）完整内嵌在**附录 A**，参数表在**附录 B**，排错手册在**附录 C**。
> 拿到本文件后**第一步是步骤 0**：把附录 A 的内容原样落盘成 `/opt/daidai-panel/data/scripts/yuanbao_install.sh`。

## 二、AI 必须遵守的行事规则

1. **先体检，后动手。** 任何安装动作之前先跑 `CF_SURVEY_ONLY=1`（步骤 1）。体检全程只读。
2. **绝不以「域名后缀」判断能不能覆盖。** 不要用「域名里有没有某个后缀」决定重建。要判断归属就用 API 查 UUID（`CF_TUNNEL_MODE=auto`），查不到归属就按复用处理。
3. **默认复用，不打断服务。** `CF_TUNNEL_MODE` 默认 `reuse`：有可用隧道就原样保留，只按需追加域名。只有用户明确要求换隧道，才用 `new`。
4. **复现用户意图，不擅自扩大范围。** 卸载、删云端隧道、删面板目录都是破坏性操作，必须用户明确表态或显式给对应开关。
5. **凭据四条铁律**：真实 Token / Key / 邮箱**绝不写进 SKILL.md、日志、记忆**；只经环境变量传入；需要确认存在时只输出长度/布尔值，不输出值；含凭据的文件一律 600，且不要用文件读取工具打开。
6. **每条命令都要真跑过**，报告时说清「跑了什么、看到什么」。跑不了就直说，不要编造输出。

## 三、决策树：体检结果 → 该怎么跑

体检（步骤 1）输出里看这三行，按下表选参数：

| 体检看到 | 判定 | 该怎么跑 |
| --- | --- | --- |
| `本次【复用】` + 域名已指向 `:5700` | 现成可用 | 直接正式安装，无需额外参数 |
| `本次【复用】` + `现有隧道上没有指向面板端口 :5700 的域名` | 需要加映射 | 给 `CF_DOMAINS=<新子域>`（有 API 凭据会自动加，没给则按输出里的后台步骤手动加） |
| `cloudflared 未安装` / `没有进程在跑` | 空机 | 直接正式安装，脚本会新建隧道 |
| 用户**明确要求**换掉现有隧道 | 换隧道 | `CF_TUNNEL_MODE=new`（会打印断线警告，必须先跟用户确认） |
| 用户要求清理 | 卸载 | 见步骤 5 |

`auto` 模式（`CF_TUNNEL_MODE=auto`）：用 API 查现有隧道 UUID 在不在本账号 → 在就允许重建，不在就强制复用。**有 API 凭据时推荐用它代替人眼判断。**

## 四、步骤

### 步骤 0：把内嵌脚本落盘（单文件版专属，必做）

本手册自包含，脚本不依赖外部下载（脚本**内部**另有 gh-proxy.com 等加速镜像，用于下载面板与 cloudflared）。先把附录 A 原样写成文件：

```bash
mkdir -p /opt/daidai-panel/data/scripts
S=/opt/daidai-panel/data/scripts/yuanbao_install.sh
cat > "$S" <<'YUANBAO_INSTALL_EOF'
#  ← 把「附录 A」围栏内的全部内容原样粘到这里（一行不差）
YUANBAO_INSTALL_EOF
chmod +x "$S"
wc -l "$S"                  # 2000+ 行才对，行数明显少 = 复制截断
bash -n "$S" && echo "语法OK"   # 语法不过就别往下跑
```

**① 🤖 AI 执行**：从附录 A 开头围栏的下一行开始复制，到结尾围栏的上一行结束。

**② 💬 对用户说**

> 这个手册自带完整安装脚本，我先把脚本落到机器上，然后照手册跑。

**③ 🙏 用户手动做**：无。

**④ ✅ 做完的标志**：`wc -l` 显示 2000+ 行，`bash -n` 输出 `语法OK`。

**⑤ 🛠️ 常见失败与处理**

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 只有几百行 | 复制被截断 | 重新完整复制附录 A |
| `bash -n` 报语法错 | 多复制了围栏行 / 混入中文引号 / CRLF | 检查首尾；`sed -i 's/\r$//' "$S"` 去 CRLF |

### 步骤 1：只读体检（必做，第一个动作）

**① 🤖 AI 执行**

```bash
S=/opt/daidai-panel/data/scripts/yuanbao_install.sh   # 脚本路径按实际上传位置改
ls -la "$S"; wc -l "$S"          # 完整脚本 2000+ 行，行数明显不对说明上传失败
bash -n "$S" && echo "语法OK"     # 语法不过就不要往下跑
CF_SURVEY_ONLY=1 bash "$S" 2>&1 | tail -90
```

不是 root 就加 `sudo -E`（`-E` 保留环境变量）。

**② 💬 对用户说**

> 开工前我先只读体检一遍这台机器：有没有面板、有没有已经在跑的隧道、绑了哪些域名、分别指向本机哪个端口。全程不装东西、不重启服务。看完我告诉你是「直接复用」还是「需要新建」，再问你要不要继续。

**③ 🙋 用户手动做**

无。

**④ ✅ 做完的标志**

输出末尾出现 `===== 体检结束（全程只读，未做任何改动）=====`，且能看到：

- `隧道 UUID（从运行中的 token 解出）: xxxxxxxx-...`（token 模式）或配置里的 tunnel id
- `已绑定域名 (N 个): ...` + `域名 → 本机服务:` 配对
- 决策行：`=> 检测到可用隧道...本次【复用】` 或 `=> 现有隧道不满足复用条件...将新建隧道`

**⑤ 🛠️ 常见失败与处理**

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `wc -l` 只有几百行 | 上传截断 | 重新传；校验行数 |
| `bash -n` 报错 | 传输时换行/编码损坏 | 重传；或 `sed -i 's/\r$//'` 去 CRLF |
| 决策行说「未绑定域名」但机器明明在用 | 远程托管(token)模式，本地 `config.yml` 不生效 | 正常；真正生效的是 Cloudflare 后台的 Public Hostname，以体检里的 UUID 与 DNS 探测为准 |
| 体检卡住不动 | 脚本在等交互输入 | 体检阶段不该有交互；检查是否误设了别的开关 |

### 步骤 2：Cloudflare 凭据（需要自动建隧道/加域名时才要）

**① 🤖 AI 执行**

两种给法，二选一（**不要两种混用**）：

```bash
# A. API Token（推荐，权限可控）—— 邮箱必须留空，否则会 403
export CF_API_TOKEN='<用户给的 Token>'
# export CF_EMAIL=''          # 留空或不设

# B. Global API Key —— 必须同时给邮箱，否则 403
export CF_EMAIL='<账号邮箱>'
export CF_API_TOKEN='<Global API Key>'

# Account ID 可以不给，脚本会自动探测（该凭据下只有一个账号时直接选定）
# export CF_ACCOUNT_ID='<32位hex>'

# 验证凭据是否与认证方式匹配（只看 success，不打印值）
python3 - <<'PY'
import os, urllib.request, urllib.error, json
t = os.environ.get('CF_API_TOKEN', ''); e = os.environ.get('CF_EMAIL', '')
h = {'Content-Type': 'application/json'}
if e: h.update({'X-Auth-Email': e, 'X-Auth-Key': t})
else:  h.update({'Authorization': 'Bearer ' + t})
try:
    d = json.load(urllib.request.urlopen(
        urllib.request.Request('https://api.cloudflare.com/client/v4/user', headers=h), timeout=25))
    print('认证成功' if d.get('success') else '认证失败: %s' % d.get('errors'))
except urllib.error.HTTPError as ex: print('HTTP', ex.code)
PY
```

也可以在脚本跑到步骤 6 时让它**交互式提示输入**（输入不回显）：只给 `CF_API_TOKEN` 环境变量就跳过询问；都不给且是交互终端，脚本会问邮箱和 Token。非交互环境（面板任务、管道执行）不会问，直接走「后台手动操作」提示。

**② 💬 对用户说**

> 要自动建隧道 / 加域名映射，需要 Cloudflare 凭据，两种给法：
>
> **A. API Token（推荐）**：打开 https://dash.cloudflare.com/profile/api-tokens → Create Token，权限给 `Account:Cloudflare Tunnel:Edit` + `Zone:DNS:Edit`。**给了 Token 就不要填邮箱。**
>
> **B. Global API Key**：https://dash.cloudflare.com/profile/api-tokens → Global API Key → View。**必须连邮箱一起给**，它是账号全权密钥。
>
> 通过安全页面发我（别直接贴聊天）；或者你什么都不给，我在跑的时候用不回显的方式问你，也可以直接回车跳过——跳过的话我会给你后台手动操作的步骤。

**③ 🙋 用户手动做**

- 生成 API Token（路径 A）或取 Global API Key + 邮箱（路径 B），通过安全页面发给 AI；或等脚本交互询问时输入。

**④ ✅ 做完的标志**

上面的验证脚本输出 `认证成功`。失败（`Invalid access token` / 403）几乎都是**认证方式与凭据类型不匹配**：`cfk_` 前缀或 37 位 hex = Global API Key 要配邮箱；无前缀随机串 = API Token 不能配邮箱。

**⑤ 🛠️ 常见失败与处理**

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `9109 Invalid access token` | Token 类型与认证方式不匹配 / 已吊销 | 按上面区分类型；仍失败让用户去后台重建 |
| 有 Token 却填了邮箱 | 走了 X-Auth-Key 分支 | 清空 `CF_EMAIL` |
| `自动获取账号列表失败` | Token 没有 Account 读权限 | 显式给 `CF_ACCOUNT_ID` |
| 多个账号不知道选哪个 | 凭据下有多个账号 | 脚本会列出来；交互时手选，非交互就显式给 `CF_ACCOUNT_ID` |

### 步骤 3：正式安装

**① 🤖 AI 执行**

```bash
# 最常用：有可用隧道就复用（默认），指定给面板用的域名
CF_DOMAINS=yb1.<用户域名> bash "$S" 2>&1 | tail -120

# 空机 / 确认要新建隧道：有凭据时推荐 auto（按归属判断能不能重建）
CF_TUNNEL_MODE=auto CF_DOMAINS=yb1.<用户域名> bash "$S" 2>&1 | tail -120

# 明确要重建（会打断现有隧道，必须先跟用户确认）
CF_TUNNEL_MODE=new CF_DOMAIN_TPL='yb{N}.<用户域名>' bash "$S"

# 只装面板不碰隧道：不给任何 CF 凭据即可
bash "$S"
```

非 root 加 `sudo -E`。

**② 💬 对用户说**

> 按体检结果，这台机器走【复用现有隧道 / 新建隧道】，我用域名 `<域名>` 给面板做公网入口。面板装好、隧道通了之后，我会把保活脚本放进面板并注册一个每天跑的定时任务。中间不用你操作，我跑完把验收结果发你。

**③ 🙋 用户手动做**

无（除非脚本输出「需到 Cloudflare 后台补 Public Hostname」，那就照它给的路径补一条）。

**④ ✅ 做完的标志**

全部满足才算完成，缺一项先修不要往下走：

```bash
systemctl is-active daidai-panel              # 期望 active
systemctl is-active cloudflared              # 期望 active
curl -s -o /dev/null -w '%{http_code}\n' --max-time 10 https://<面板域名>/   # 期望 200/30x
ls -l /opt/daidai-panel/data/scripts/ybbh_keepalive.py                        # 存在
grep -c '' /var/log/yuanbao_install.log                                       # 有安装日志
```

判据：面板和隧道都 active + 域名 HTTPS 通 + 保活脚本落盘。**只有域名返回 200 还不够**——要确认它打到的是面板端口（脚本已做：不会拿指向别的服务端口的域名来验收）。

**面板密码从哪来**：脚本里**不写死密码**，运行时确定 —— 没给 `PANEL_PASS` 就随机生成 16 位，打印在安装末尾的「面板账号密码」行，并存到 `/opt/yuanbao-install/.panel_pass`（权限 600，重跑时复用）。要告诉用户密码就**从脚本输出里读**，不要自己编一个默认密码。想固定密码就 `PANEL_PASS=xxx` 显式给。体检模式（`CF_SURVEY_ONLY=1`）全程只读，不生成也不落盘密码。

**⑤ 🛠️ 常见失败与处理**

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 复用模式下域名没生效 | 隧道不在本账号，脚本加不了映射 | 按输出提示去对应账号后台加 Public Hostname；或换用本账号域名 + `new`/`auto` |
| 隧道起不来 | token 错 / 配置冲突 | `journalctl -u cloudflared -n 30 --no-pager` |
| 面板 502 | ingress 指向端口不对 | 脚本会自动探测真实端口（进程实际监听 → `config.yaml` → 默认 5700，5701 也能认）；也可 `PANEL_PORT=xxxx` 显式覆盖 |
| 域名不通但隧道 active | DNS CNAME 没生效或没写 | 查 DNS 是否指向 `<tunnel>.cfargotunnel.com` |
| 安装中途 `die` 退出 | 前置条件不满足（非 root / 缺依赖） | 看 `/var/log/yuanbao_install.log` 尾部 |

### 步骤 4：验收保活真的能发消息

**① 🤖 AI 执行**

```bash
cd /opt/daidai-panel/data/scripts && timeout 300 python3 ybbh_keepalive.py; echo "退出码=$?"
tail -25 ~/.openclaw/yuanbao_keepalive.log
```

**② 💬 对用户说**

> 装完了，我手动跑一次保活验证它真的能把消息发出去（不是只看进程在不在）。

**③ 🙋 用户手动做**

无。

**④ ✅ 做完的标志**

三条同时对上才算成功：

| 输出 | 期望 |
| --- | --- |
| `选中会话: agent:main:yuanbao:direct:...` | 必须是 `direct`，出现 `group` 说明选错会话 |
| `USER_ID:    rHn2tZN...` | **首字母必须大写**；全是小写说明用的是旧版脚本 |
| `gateway: send ok=True` + `Bot 回复 N 字` | 有回复字数 = 真的送达了 |

**⑤ 🛠️ 常见失败与处理**

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `USER_ID` 全小写 | 脚本版本旧 | 换成新版（本文件附录 A 里那份） |
| 选中 `group` 会话 | 会话选择逻辑旧 | 同上 |
| `send ok=False` | openclaw gateway 未就绪 | 检查 openclaw 服务状态 |
| 输出「跳过」 | 距上次成功 < 20 小时的防打扰窗口 | 设计如此；要立刻验证先清 state 或调小 `MIN_INTERVAL_H` |

### 步骤 5：卸载（用户明确要求才做）

**① 🤖 AI 执行**

```bash
# 档 1（默认，可逆）：只停本机 cloudflared，备份配置，不碰云端、不碰面板
UNINSTALL=1 bash "$S"

# 档 2：连云端隧道 + DNS 一起删（不可恢复；隧道不属于本账号时自动跳过云端删除）
export CF_EMAIL='<邮箱>'; export CF_API_TOKEN='<Key>'
UNINSTALL=1 UNINSTALL_PURGE=1 bash "$S"

# 档 3：全清（+停面板 +删保活任务 +删含凭据的 cf.conf）；非交互必给 UNINSTALL_YES=1
UNINSTALL=1 UNINSTALL_SCOPE=all UNINSTALL_PURGE=1 UNINSTALL_YES=1 bash "$S"
```

`UNINSTALL_SCOPE`：`cf`（默认）/ `panel` / `keepalive` / `all`。

**② 💬 对用户说**

> 卸载分档，你选：
>
> - **只停本机隧道服务**（推荐先做这个）：可恢复，云端隧道和域名都保留；
> - **连云端隧道和 DNS 一起删**：不可恢复，而且只有隧道属于你这个账号才会真删（别人的会跳过并提示你去对应后台删）；
> - **全清**：再把面板停掉、保活任务删掉、含凭据的配置文件删掉。
>
> 注意：Cloudflare 后台删隧道时提示的「在每台计算机上跑 `cloudflared service uninstall`」说的就是档 1——**云端删了不代表机器上的进程会停**，必须本机跑一次。

**③ 🙋 用户手动做**

- 选档并确认；档 2/档 3 涉及不可恢复删除，必须明确同意。

**④ ✅ 做完的标志**

```bash
pgrep -f cloudflared >/dev/null && echo "还在跑" || echo "已停"
systemctl is-active cloudflared 2>/dev/null || echo "服务已停"
ls /etc/cloudflared 2>/dev/null || echo "配置已清空"
ls -d /opt/yuanbao-install/uninstall-backup-* 2>/dev/null   # 备份在，可回滚
```

**⑤ 🛠️ 常见失败与处理**

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 卸完还有 cloudflared 进程 | 机器上装了多个隧道实例 | 脚本会提示；`pgrep -af cloudflared` 看是哪个 |
| 云端没删 | 隧道不在本账号 / 没给凭据 | 去对应 CF 账号后台删；DNS 残留用 `cmd_deldns` 或后台删 |
| 非交互跑卸载没反应 | 没给 `UNINSTALL_YES=1` | 脚本会直接中止（安全设计），补上参数 |

## 五、排错速查

完整参数表见**附录 B**，隧道/域名问题的详细排查见**附录 C**（单文件版已把它们内嵌进来，无需外部文件）。

先跑这三条拿现场，再对症下药：

```bash
tail -40 /var/log/yuanbao_install.log          # 安装日志
systemctl status daidai-panel --no-pager | head -8
journalctl -u cloudflared -n 30 --no-pager     # 隧道日志
```

| 症状 | 最可能的原因 | 处理 |
| --- | --- | --- |
| 面板域名 502/超时 | 隧道连不上边缘或源站端口不对 | 看 `journalctl -u cloudflared`；核对 `PANEL_PORT` |
| 域名解析不到 | DNS 没写 / 没生效 | 查 CNAME 是否指向 `<tunnel-id>.cfargotunnel.com` |
| 装完面板任务不跑 | 任务是禁用状态 | 面板里确认任务 `status=1`；脚本建任务时会补调 enable |
| 保活脚本报找不到会话 | openclaw 会话格式变了 | 看脚本里会话选择逻辑，选 `direct` 而非 `group` |
| 重跑脚本后多出垃圾隧道 | 没复用本机安装记录 | 用 `auto`/`reuse`；安装记录存在 `/opt/yuanbao-install/.install_state` |

## 六、铁律（违反必出事）

1. **绝不重建正在服务别人的隧道。** 任何覆盖动作前先确认归属（API 查 UUID）。重建不可自动恢复：旧 token 解绑、域名当场断线、要重配 Public Hostname 并等 DNS。
2. **换域名 ≠ 换隧道。** 加一条 ingress 就够；`cmd_appendconfig` 会保留原有所有规则（区别于会整体覆盖的 `cmd_setconfig`）。
3. **token 远程托管模式下本地 `config.yml` 不生效。** 别拿本地配置当事实，真正生效的是 Cloudflare 后台的 Public Hostname。
4. **验收只看「消息真的发出去」。** 进程在、HTTP 200 都不算数——保活要有 Bot 回复字数，面板要确认域名打到面板端口。
5. **凭据不落文档。** Token / Key / 邮箱只存在于环境变量和 600 权限的文件里，不写进 SKILL.md、日志、记忆；报告时只说「认证成功 / 失败」，不回显值。
6. **破坏性操作要显式开关。** `UNINSTALL_PURGE=1`、`CF_TUNNEL_MODE=new`、`UNINSTALL_YES=1` 都是刻意设计的门槛，不要为了方便替用户省掉。

---

## 附录 A：安装脚本（原样落盘到 `/opt/daidai-panel/data/scripts/yuanbao_install.sh`）

> **AI 注意**：以下围栏内是**完整安装脚本**。按步骤 0 原样写入文件 —— **不要修改、不要"优化"、不要自己重写安装逻辑**。
> 脚本已内置 `gh-proxy.com` 等 GitHub 加速镜像（首选 gh-proxy，失败依次回退备用镜像、直连），下载面板与 cloudflared 时自动生效，不需要额外配置。

```bash
#!/usr/bin/env bash
# ==============================================================================
# 元宝派 · 龙虾一键安装脚本 (Ubuntu x86_64)
#   1. 安装呆呆面板(daidai-panel) 最新版 linux-amd64
#   2. 初始化面板账号（密码随机生成并在末尾打印；也可 PANEL_PASS=xxx 指定）
#   3. 写入元宝保活 py 脚本 + 每天执行一次的计划任务
#   4. systemd + @reboot 双重保活面板开机自启
#   5. 安装 cloudflared
#   6. 创建 Cloudflare Tunnel (yb-dd1 / yb-dd2 ...) 并映射公网域名 + 安装开机服务
#   7. 检测域名是否可访问，失败明确指出卡在第几步
#
# 用法:
#   sudo bash yuanbao_install.sh                      # 交互式输入域名
#   CF_DOMAINS=dd.your.com sudo -E bash yuanbao_install.sh
#   CF_DOMAINS=dd1.a.com,dd2.a.com sudo -E bash yuanbao_install.sh      # 一个隧道挂多个域名
#   CF_TUNNEL_TOKEN=eyJ... CF_DOMAINS=dd.your.com sudo -E bash yuanbao_install.sh  # 远程托管隧道模式
#
#   多台机器批量上线（推荐，全程自动化，每台自动领编号/隧道/token/域名）:
#   export CF_API_TOKEN=xxxxx CF_ACCOUNT_ID=xxxxx CF_DOMAIN_TPL='yb{N}.4g.cc.cd'
#   sudo -E bash yuanbao_install.sh      # 每台机器跑同一条命令即可
#
#  其它可选变量:
#   CF_SKIP_NUMS=3,7              已占用的编号，脚本自动跳过（默认跳过 3）
#   LOCAL_TARBALL=/path/x.tar.gz  用本地离线包，跳过下载
#   CF_TUNNEL_MODE=new            强制重建隧道（默认 reuse：已有可用隧道就复用，不破坏环境）
#   CF_TUNNEL_MODE=auto           看归属：隧道是本脚本建的才重建，否则只追加不覆盖
#   STEPS=1,3,4                   只执行指定步骤（自选安装，默认全部 1-7，见菜单）
#   CF_SURVEY_ONLY=1              只体检隧道/域名，不装任何东西，摸清现网环境首选
#
# 卸载（UNINSTALL=1 走卸载分支，不装任何东西）：
#   UNINSTALL=1 UNINSTALL_SCOPE=cf bash yuanbao_install.sh
#     scope: cf(默认,只停本机cloudflared) | panel(停面板) | keepalive(删保活) | all
#     UNINSTALL_PURGE=1  连云端隧道+DNS 一起删（不可恢复，且只删归属本账号的）
#     UNINSTALL_YES=1    不询问直接执行（非交互环境必填，否则直接中止）
#     PANEL_TOKEN=xxx    scope 含 keepalive 时用它删面板定时任务
#   CF_EMAIL=you@example.com      用 Global API Key 时填邮箱（CF_API_TOKEN 里填 Key）
#                                 用 API Token 时留空 → 走 Bearer 认证
#   CF_ACCOUNT_ID=xxxxx           不填会自动探测；该凭据下只有一个账号时直接选定
#   交互式执行且没给 CF 凭据时，脚本会提示输入邮箱 + Token（输入不回显）
#   PANEL_PASS=xxx                指定面板密码；不给则随机生成，存 /opt/yuanbao-install/.panel_pass（600）
#                                 脚本里不写死任何密码 —— 脚本会随 skill 公开分发
#
#  重跑安全：脚本会记住本机已领到的编号/隧道并复用，不会重复创建隧道；
#           已完成的步骤自动跳过，安装日志在 /var/log/yuanbao_install.log
# ==============================================================================
set -uo pipefail

# ------------------------------ 配置区 ------------------------------
PANEL_DIR="/opt/daidai-panel"
PANEL_PORT="${PANEL_PORT:-5700}"     # 可用环境变量覆盖：PANEL_PORT=5701 sudo -E bash ...
PANEL_USER="admin"
# 面板密码：没给就随机生成，落盘在 $PANEL_PASS_FILE（权限 600），重跑时复用。
# 绝不写死在脚本里 —— 脚本会随 skill 公开分发，写死等于把每台机器的面板密码公开。
PANEL_PASS="${PANEL_PASS:-}"
KEEPALIVE_SCRIPT="yuanbao_keepalive.py"
TASK_NAME="元宝自动保活"
TASK_CRON="0 9 * * *"            # 每天 09:00 执行一次
HELPER_DIR="/opt/yuanbao-install"
PANEL_PASS_FILE="$HELPER_DIR/.panel_pass"   # 随机密码存放处（600）；卸载时可一并清掉
# 面板 API 基址。体检阶段就会按实际端口修正（见 detect_panel_port）；这里是兜底默认值
BASE_URL="http://127.0.0.1:$PANEL_PORT"

# GitHub 加速镜像（按优先级尝试，最后直连）
GH_MIRRORS=(
  "https://gh-proxy.com/"
  "https://git.166628.xyz/"
  ""
)
RELEASE_URL="https://github.com/linzixuanzz/daidai-panel/releases/latest/download/daidai-linux-amd64.tar.gz"
CF_DEB_URL="https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb"

# 域名：支持多个（逗号或空格分隔）。CF_DOMAINS 优先，CF_DOMAIN 保留兼容
CF_DOMAINS="${CF_DOMAINS:-${CF_DOMAIN:-}}"
CF_TUNNEL_TOKEN="${CF_TUNNEL_TOKEN:-}"

# ---- 多台机器批量上线（模式 C：Cloudflare API 全自动）----
# CF_API_TOKEN   : Cloudflare API Token，权限需要 Account:Cloudflare Tunnel:Edit + Zone:DNS:Edit
#                  用 Global API Key 时这里填 Key，并同时给 CF_EMAIL（走 X-Auth-Key 认证）
#                  交互式执行且没给凭据时，脚本会在步骤 6 提示输入
# CF_ACCOUNT_ID  : Cloudflare Account ID（后台右侧栏可复制）
# CF_ZONE_ID     : 可选，域名所在 Zone ID，不填会自动探测
# CF_DOMAIN_TPL  : 域名模板，{N} 会被替换成机器编号，如 yb{N}.4g.cc.cd → yb1.4g.cc.cd
#                  多个用逗号分隔：yb{N}.a.com,dd{N}.a.com
# CF_SKIP_NUMS   : 已占用的机器编号，逗号分隔，脚本会跳过它们。默认跳过 3（dd3 已在使用）
# CF_MACHINE_NUM : 手动指定本机编号（给了就不再自动分配，适合机器少时精确控制）
# CF 配置建议写在独立文件里，别硬编码进脚本：
#   CF_API_TOKEN 能建隧道 + 改 DNS，脚本一旦外传等于把整个 CF 账号交出去。
#   建一次即可，之后所有机器都自动读取：
#     mkdir -p /etc/yuanbao
#     cat > /etc/yuanbao/cf.conf <<'EOF'
#     export CF_API_TOKEN=xxxxx
#     export CF_ACCOUNT_ID=xxxxx
#     export CF_DOMAIN_TPL='yb{N}.4g.cc.cd'
#     # export CF_EMAIL=you@example.com   # 仅用 Global API Key 时才需要
#     EOF
#     chmod 600 /etc/yuanbao/cf.conf
# 环境变量优先级更高（临时换账号时用 export 覆盖即可）
CF_CONF="${CF_CONF:-/etc/yuanbao/cf.conf}"
[ -r "$CF_CONF" ] && . "$CF_CONF" 2>/dev/null

CF_SKIP_NUMS="${CF_SKIP_NUMS:-3}"
CF_MACHINE_NUM="${CF_MACHINE_NUM:-}"
CF_API_TOKEN="${CF_API_TOKEN:-}"
CF_ACCOUNT_ID="${CF_ACCOUNT_ID:-}"
CF_ZONE_ID="${CF_ZONE_ID:-}"
# Cloudflare 邮箱：用 Global API Key（X-Auth-Key）时必填；用 API Token 时留空
CF_EMAIL="${CF_EMAIL:-}"
CF_DOMAIN_TPL="${CF_DOMAIN_TPL:-}"
TUNNEL_PREFIX="yb-dd"

# 自选安装：本次要执行哪些步骤，逗号分隔（1~7），all=全部，不给则交互菜单选择
STEPS="${STEPS:-}"

# --------------------------- 卸载 ---------------------------
# 现取现用：卸载时可能压根没跑过安装流程，HELPER_DIR 里没有内嵌的两个 py，
# 所以直接从本脚本源码里抽出来，避免为了卸载先装一遍依赖。
ensure_helper() {
  mkdir -p "$HELPER_DIR"
  [ -f "$HELPER_DIR/cf_api.py" ] && [ -f "$HELPER_DIR/panel_api.py" ] && return 0
  python3 - "$0" "$HELPER_DIR" <<'PYX'
import sys, os
src = open(sys.argv[1], encoding='utf-8', errors='replace').read()
out = sys.argv[2]
for name, tag in (('cf_api.py', 'CFEOF'), ('panel_api.py', 'PYEOF')):
    dst = os.path.join(out, name)
    if os.path.exists(dst):
        continue
    key = 'cat > "$HELPER_DIR/%s"' % name
    i = src.find(key)
    if i < 0:
        continue
    a = src.find(tag, i)
    if a < 0:
        continue
    a += len(tag) + 1
    b = src.find(tag, a)
    if b < 0:
        continue
    open(dst, 'w', encoding='utf-8').write(src[a:b])
PYX
  chmod +x "$HELPER_DIR"/*.py 2>/dev/null || true
}

# 卸载主流程：默认只停本机服务（可恢复）；
# 删云端隧道 / DNS / 面板目录必须 UNINSTALL_PURGE=1，且隧道归属本账号才动手
run_uninstall() {
  local scope="${UNINSTALL_SCOPE:-cf}"
  local purge="${UNINSTALL_PURGE:-0}"
  ensure_helper

  echo
  echo -e "${C_G}===== 卸载模式 =====${C_0}"
  echo "  范围: $scope      清云端数据: $purge"
  echo

  # 1) 先体检，看清这台机器上有什么
  [ -f "$STATE_FILE" ] && . "$STATE_FILE" 2>/dev/null || true
  survey_tunnel

  # 2) 有凭据就查归属：不是本账号的隧道，云端一律不碰
  local owned="unknown"
  if [ "$purge" = "1" ] && [ -n "$CF_API_TOKEN" ] && [ -n "$EXIST_TUNNEL_ID" ]; then
    if [ -z "$CF_ACCOUNT_ID" ]; then
      _acc="$(python3 "$HELPER_DIR/cf_api.py" "$CF_API_TOKEN" accounts 2>&1 | head -n1)"
      case "$_acc" in ERR*) warn "账号列表获取失败: $_acc";; *) CF_ACCOUNT_ID="${_acc%% *}"; ok "Account: $_acc";; esac
    fi
    [ -n "$CF_ACCOUNT_ID" ] \
      && owned="$(python3 "$HELPER_DIR/cf_api.py" "$CF_API_TOKEN" tunnelowned "$CF_ACCOUNT_ID" "$EXIST_TUNNEL_ID" 2>&1)"
  fi

  # 3) 列出将要做什么，等确认
  echo
  echo "  将要执行："
  case "$scope" in
    cf|all)
      echo "    · 停止 + 禁用 cloudflared 服务，移除 systemd unit"
      echo "    · 备份并清空 /etc/cloudflared"
      [ "$purge" = "1" ] && echo "    · [云端] 删除隧道 ${EXIST_TUNNEL_ID:-未知} 及其 DNS 记录（不可恢复）"
      ;;
  esac
  case "$scope" in
    panel|all)
      echo "    · 停止 + 禁用 daidai-panel 服务"
      [ "$purge" = "1" ] && echo "    · [危险] 删除 $PANEL_DIR 整个目录（不可恢复）"
      ;;
  esac
  case "$scope" in
    keepalive|all)
      echo "    · 移走保活脚本，删除面板里的保活定时任务"
      ;;
  esac
  echo "    · 清理安装记录 $STATE_FILE"
  [ "$purge" = "1" ] && echo "    · 删除 $CF_CONF（含 API 凭据）"
  echo
  if [ "${UNINSTALL_YES:-0}" != "1" ]; then
    if [ ! -t 0 ]; then
      die "非交互环境必须显式给 UNINSTALL_YES=1 才会执行，已中止"
    fi
    read -r -p "  确认执行？输入 yes 继续（其它任意键取消）: " _c
    [ "$_c" = "yes" ] || { echo "  已取消，未做任何改动"; return 0; }
  fi

  local backup="$HELPER_DIR/uninstall-backup-$(date +%Y%m%d-%H%M%S)"
  mkdir -p "$backup"

  # 4) cloudflared
  case "$scope" in
    cf|all)
      log "[cf] 停止 cloudflared"
      systemctl stop cloudflared 2>/dev/null || true
      pkill -f cloudflared 2>/dev/null || true
      command -v cloudflared >/dev/null 2>&1 && cloudflared service uninstall 2>&1 | sed 's/^/      /'
      systemctl disable cloudflared 2>/dev/null || true
      rm -f /etc/systemd/system/cloudflared.service 2>/dev/null || true
      systemctl daemon-reload 2>/dev/null || true
      if [ -d /etc/cloudflared ]; then
        cp -a /etc/cloudflared "$backup/cloudflared" 2>/dev/null || true
        rm -rf /etc/cloudflared
        ok "[cf] 已清空 /etc/cloudflared（备份在 $backup/cloudflared）"
      fi
      if [ "$purge" = "1" ] && [ -n "$EXIST_TUNNEL_ID" ]; then
        if [ -z "$CF_API_TOKEN" ] || [ -z "$CF_ACCOUNT_ID" ] || [ "$owned" = "NO" ] || [ "$owned" = "unknown" ]; then
          warn "[cf] 隧道 $EXIST_TUNNEL_ID 不在本账号 / 缺凭据 → 云端未删，去对应账号后台删"
        else
          for _d in "${EXIST_DOMAINS[@]}"; do
            _z="$(python3 "$HELPER_DIR/cf_api.py" "$CF_API_TOKEN" findzone "$_d" 2>&1 | head -n1)"
            case "$_z" in ERR*) continue;; esac
            printf '      DNS %s: ' "$_d"
            python3 "$HELPER_DIR/cf_api.py" "$CF_API_TOKEN" deldns "${_z%% *}" "$_d" 2>&1
          done
          printf '      隧道: '
          python3 "$HELPER_DIR/cf_api.py" "$CF_API_TOKEN" deltunnel "$CF_ACCOUNT_ID" "$EXIST_TUNNEL_ID" 2>&1
        fi
      fi
      pgrep -f cloudflared >/dev/null 2>&1 \
        && warn "[cf] 仍有 cloudflared 进程在跑（可能还有别的隧道实例）" \
        || ok "[cf] 本机已无 cloudflared 进程"
      ;;
  esac

  # 5) 面板
  case "$scope" in
    panel|all)
      systemctl stop daidai-panel 2>/dev/null || true
      systemctl disable daidai-panel 2>/dev/null || true
      rm -f /etc/systemd/system/daidai-panel.service 2>/dev/null || true
      systemctl daemon-reload 2>/dev/null || true
      ok "[panel] 服务已停止并禁用"
      if [ "$purge" = "1" ] && [ -d "$PANEL_DIR" ]; then
        rm -rf "$PANEL_DIR"
        warn "[panel] 已删除 $PANEL_DIR（不可恢复，数据目录也没了）"
      fi
      ;;
  esac

  # 6) 保活任务
  case "$scope" in
    keepalive|all)
      local ks="$PANEL_DIR/data/scripts/ybbh_keepalive.py"
      if [ -f "$ks" ]; then mv "$ks" "$backup/" 2>/dev/null; ok "[keepalive] 脚本已移走: $ks"; fi
      if [ -n "${PANEL_TOKEN:-}" ]; then
        _tasks="$(python3 "$HELPER_DIR/panel_api.py" "$PANEL_TOKEN" listtasks 2>&1)"
        _tid="$(printf '%s\n' "$_tasks" | grep -E '保活|keepalive' | awk '{print $1}' | head -n1)"
        if [ -n "$_tid" ]; then
          printf '      '
          python3 "$HELPER_DIR/panel_api.py" "$PANEL_TOKEN" deltask "$_tid" 2>&1
        else
          warn "[keepalive] 面板里没找到保活任务"
        fi
      else
        warn "[keepalive] 未给 PANEL_TOKEN，面板定时任务请手动删（面板 → 定时任务 → 删除）"
      fi
      ;;
  esac

  # 7) 收尾
  [ -f "$STATE_FILE" ] && { cp -a "$STATE_FILE" "$backup/" 2>/dev/null; rm -f "$STATE_FILE"; ok "已清理安装记录（备份在 $backup）"; }
  if [ "$purge" = "1" ] && [ -f "$CF_CONF" ]; then rm -f "$CF_CONF"; ok "已删除 $CF_CONF"; fi

  echo
  echo -e "${C_G}===== 卸载完成 =====${C_0}"
  echo "  备份: $backup"
  echo "  日志: $LOG_FILE"
  echo
  echo "  提示：云端隧道如果上面没删掉，记得去 Cloudflare 后台删，否则会一直显示离线"
  return 0
}

# 机器上已经有隧道时怎么处理：
#   reuse = 探测到「隧道在跑 + 已绑定域名」就原样复用，不新建、不卸载、不改映射（默认，最安全）
#   auto  = 看归属：隧道 UUID 与本机安装记录对得上（本脚本建的）才允许重建，否则同 reuse
#   new   = 无条件重建隧道并替换现有域名映射（明确要换隧道时才用）
# 注意：不要用「域名后缀」判断隧道归属 —— 换域名只需追加一条 ingress，
#       重建隧道会打断它上面所有正在服务的域名，且不可自动恢复
CF_TUNNEL_MODE="${CF_TUNNEL_MODE:-reuse}"
# =1 时只做隧道/域名体检就退出，不安装任何东西（先摸清测试环境再决定）
CF_SURVEY_ONLY="${CF_SURVEY_ONLY:-0}"

# 运行时文件
LOG_FILE="/var/log/yuanbao_install.log"      # 完整安装日志
LOCK_FILE="/var/lock/yuanbao_install.lock"   # 防重复/并发运行
STATE_FILE="$HELPER_DIR/.install_state"      # 记录本机已领编号与隧道，重跑时复用
SUMMARY_FILE="$HELPER_DIR/install_summary.txt"

# 离线安装：若已手动下载好 tar.gz，用 LOCAL_TARBALL=/path/xxx.tar.gz 指定，跳过下载
LOCAL_TARBALL="${LOCAL_TARBALL:-}"

STEP=0
FAIL_REASON=""

# ------------------------------ 通用函数 ------------------------------
C_G='\033[32m'; C_R='\033[31m'; C_Y='\033[33m'; C_B='\033[36m'; C_0='\033[0m'
log()  { echo -e "${C_B}[$(date '+%H:%M:%S')]${C_0} $*"; }
ok()   { echo -e "${C_G}  ✔ $*${C_0}"; }
warn() { echo -e "${C_Y}  ! $*${C_0}"; }
err()  { echo -e "${C_R}  ✘ $*${C_0}"; }
step() { STEP=$1; echo; echo -e "${C_G}===== 步骤 $1: $2 =====${C_0}"; }

die() {
  echo
  echo -e "${C_R}############################################################${C_0}"
  echo -e "${C_R}  安装中断：失败发生在【步骤 $STEP】${C_0}"
  echo -e "${C_R}  原因：$1${C_0}"
  echo -e "${C_R}############################################################${C_0}"
  echo -e "${C_Y}  排查建议：${C_0}"
  echo "    · 面板日志: journalctl -u daidai-panel -n 100 --no-pager  或  tail -n 100 $PANEL_DIR/data/panel.log"
  echo "    · 隧道日志: journalctl -u cloudflared -n 100 --no-pager"
  echo "    · 修好后重新执行本脚本即可（已完成的步骤会自动跳过）"
  exit 1
}

# 自选安装：判断某一步本次是否执行（STEPS=all 或包含该编号）
should_run() {
  case ",${STEPS:-all}," in
    *",all,"*|*",$1,"*) return 0;;
  esac
  return 1
}

# ---------- Cloudflare 凭据交互输入 ----------
# 只在确实要调 CF API、且没给凭据时才问；非交互环境直接跳过（末尾会给后台手动步骤）
ask_cf_creds() {
  [ -t 0 ] || return 1
  echo
  echo -e "${C_G}----- Cloudflare 凭据 -----${C_0}"
  echo "  自动建隧道 / 追加域名映射需要 CF API 权限，两种给法："
  echo
  echo "  1) API Token（推荐，权限可控）—— 邮箱留空，直接回车"
  echo "     https://dash.cloudflare.com/profile/api-tokens → Create Token"
  echo "     权限: Account:Cloudflare Tunnel:Edit + Zone:DNS:Edit"
  echo
  echo "  2) Global API Key —— 填邮箱 + Key（权限等同账号全权，临时用）"
  echo "     https://dash.cloudflare.com/profile/api-tokens → Global API Key → View"
  echo
  echo "  （两项都直接回车 = 跳过，末尾会给你后台手动操作的步骤）"
  echo
  read -r -p "邮箱（用 API Token 就留空）: " CF_EMAIL
  read -r -s -p "API Token / Global API Key（不回显）: " CF_API_TOKEN
  echo
  [ -n "$CF_API_TOKEN" ] || { warn "未输入凭据，本次不自动配置 CF"; return 1; }
  export CF_EMAIL CF_API_TOKEN
  if [ -n "$CF_EMAIL" ]; then
    ok "已读入: Global API Key（邮箱 $CF_EMAIL）"
  else
    ok "已读入: API Token（长度 ${#CF_API_TOKEN}）"
  fi
  return 0
}


# --------------------------- 面板端口探测（只读）---------------------------
# 不同版本的呆呆面板默认端口不一样（5700 / 5701 都见过），必须按实际情况定。
# 关键：要在「体检」之前定准 —— 体检拿端口去匹配隧道上已存在的 ingress 规则
# （哪条域名指向面板端口），用默认端口去匹配会把 :5701 的现成映射判成没有，
# 明明能复用却要求用户另外给域名。
detect_panel_port() {
  local live cfg
  # 1) 已经在跑的面板进程：它实际监听的端口最可信
  live="$(ss -ltnp 2>/dev/null | grep -iE 'daidai|dumb' \
          | grep -oE ':[0-9]{2,5}[[:space:]]' | grep -oE '[0-9]+' | head -n1)"
  if [ -n "$live" ] && [ "$live" -ne "$PANEL_PORT" ] 2>/dev/null; then
    warn "面板实际监听 $live（脚本默认 $PANEL_PORT），以实际端口为准"
    PANEL_PORT="$live"
  fi
  # 2) 配置文件里写了端口，以配置为准（面板没启动时只有这里能读到）
  if [ -f "$PANEL_DIR/config.yaml" ]; then
    cfg="$(grep -iE '^\s*(port|panel_port|listen_port)\s*:' "$PANEL_DIR/config.yaml" \
           | head -n1 | grep -oE '[0-9]{2,5}' | head -n1)"
    if [ -n "$cfg" ] && [ "$cfg" -ne "$PANEL_PORT" ] 2>/dev/null; then
      warn "面板配置端口为 $cfg，改用该端口（当前 $PANEL_PORT）"
      PANEL_PORT="$cfg"
    fi
  fi
  BASE_URL="http://127.0.0.1:$PANEL_PORT"
}

# --------------------------- 隧道 / 域名 体检（只读）---------------------------
# 只探测，不安装、不修改、不重启任何服务。
# 产出全局量：CF_RUNNING / CF_MODE / EXIST_TUNNEL_ID / EXIST_TUNNEL_NAME / EXIST_DOMAINS[]
survey_tunnel() {
  CF_RUNNING=0; CF_MODE="none"
  EXIST_TUNNEL_ID=""; EXIST_TUNNEL_NAME=""
  EXIST_DOMAINS=(); EXIST_DOMAIN_SERVICES=()
  echo
  echo -e "${C_G}----- 隧道 / 域名 体检（只读，不改动任何配置）-----${C_0}"

  if ! command -v cloudflared >/dev/null 2>&1; then
    warn "cloudflared 未安装 → 没有隧道，步骤 5/6 会新建"
    return 0
  fi
  ok "cloudflared 已安装: $(cloudflared --version 2>/dev/null | head -n1)"

  # 1) 到底有没有在跑
  local pid=""
  if systemctl is-active --quiet cloudflared 2>/dev/null; then
    CF_RUNNING=1; ok "cloudflared 服务: 运行中 (systemd: active)"
  fi
  pid="$(pgrep -f 'cloudflared' 2>/dev/null | head -n1)"
  if [ -n "$pid" ]; then
    CF_RUNNING=1; ok "cloudflared 进程: 在跑 (pid $pid)"
  else
    warn "cloudflared 已安装，但当前没有进程在跑"
  fi

  # 2) 运行模式：--token（远程托管）还是本地命名隧道
  local cmdline=""
  [ -n "$pid" ] && cmdline="$(tr '\0' ' ' < /proc/$pid/cmdline 2>/dev/null)"
  if [ -z "$cmdline" ] && [ "$HAS_SYSTEMD" -eq 1 ]; then
    cmdline="$(systemctl cat cloudflared 2>/dev/null | grep -m1 'ExecStart' || true)"
  fi
  case "$cmdline" in
    *--token*)    CF_MODE="token"; ok "运行模式: 远程托管隧道 (token)";;
    *tunnel\ run*|*tunnel*) CF_MODE="named"; ok "运行模式: 本地命名隧道";;
    *) warn "运行模式: 未能识别（进程可能刚启动或已退出）";;
  esac

  # 2b) 远程托管模式：从 --token 里解出隧道 UUID（最权威；本地 config.yml 可能压根没被加载）
  local tk=""
  tk="$(printf '%s' "$cmdline" | grep -oE -- '--token[[:space:]]+[^[:space:]]+' | awk '{print $NF}')"
  if [ -n "$tk" ]; then
    local payload pad tid_tk
    payload="$(printf '%s' "$tk" | cut -d. -f2)"
    pad=$(( (4 - ${#payload} % 4) % 4 ))
    [ "$pad" -gt 0 ] && payload="${payload}$(printf '=%.0s' $(seq 1 "$pad"))"
    tid_tk="$(printf '%s' "$payload" | tr '_-' '/+' | base64 -d 2>/dev/null \
              | grep -oE '[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}' | head -n1)"
    if [ -n "$tid_tk" ]; then
      EXIST_TUNNEL_ID="$tid_tk"
      ok "隧道 UUID（从运行中的 token 解出）: $tid_tk"
      warn "远程托管模式下本地 config.yml 不生效，真正生效的是 Cloudflare 后台的 Public Hostname"
    fi
  fi

  # 3) 本地配置文件 → 已绑定域名（ingress hostname → service）
  local cfg=""
  for f in /etc/cloudflared/config.yml /etc/cloudflared/config.yaml; do
    [ -f "$f" ] && cfg="$f" && break
  done
  if [ -n "$cfg" ]; then
    ok "配置文件: $cfg"
    [ -n "$EXIST_TUNNEL_ID" ] \
      || EXIST_TUNNEL_ID="$(grep -m1 -E '^[[:space:]]*tunnel:' "$cfg" | sed -E 's/^[^:]*:[[:space:]]*//')"
    mapfile -t EXIST_DOMAINS < <(grep -oE 'hostname:[[:space:]]*[^[:space:]]+' "$cfg" \
                                 | sed -E 's/^hostname:[[:space:]]*//' | sort -u)
    [ -n "$EXIST_TUNNEL_ID" ] && echo "      tunnel     : $EXIST_TUNNEL_ID"
    if [ ${#EXIST_DOMAINS[@]} -gt 0 ]; then
      ok "已绑定域名 (${#EXIST_DOMAINS[@]} 个): ${EXIST_DOMAINS[*]}"
    else
      warn "配置里没有任何 hostname → 隧道没绑域名"
    fi
    # hostname → service 配对，看清每个域名打到本机哪个端口（格式: 域名|service）
    mapfile -t EXIST_DOMAIN_SERVICES < <(awk '/hostname:[[:space:]]*[^[:space:]]+/{h=$NF}
                  /service:[[:space:]]*[^[:space:]]+/{if(h!=""){print h "|" $NF; h=""}}' "$cfg")
    if [ ${#EXIST_DOMAIN_SERVICES[@]} -gt 0 ]; then
      echo "      域名 → 本机服务:"
      for _p in "${EXIST_DOMAIN_SERVICES[@]}"; do
        echo "        · ${_p%%|*}  →  ${_p#*|}"
      done
    fi
  else
    warn "未找到 /etc/cloudflared/config.yml（远程托管模式通常没有本地配置，属正常）"
  fi

  # 4) 本地凭据文件（一个 json = 一个隧道）
  local creds
  creds="$(ls /etc/cloudflared/*.json /root/.cloudflared/*.json "${HOME:-/root}"/.cloudflared/*.json 2>/dev/null \
           | grep -v 'cert.pem' | head -n5)"
  if [ -n "$creds" ]; then
    echo "      隧道凭据文件:"
    printf '%s\n' "$creds" | while read -r c; do echo "      · $(basename "$c")"; done
  fi

  # 5) 远端隧道列表（只有本地有 cert.pem 时才能查）
  if [ -f /root/.cloudflared/cert.pem ] || [ -f "${HOME:-/root}/.cloudflared/cert.pem" ]; then
    local tl; tl="$(cloudflared tunnel list 2>/dev/null)"
    if [ -n "$tl" ]; then
      echo "      --- cloudflared tunnel list ---"
      printf '%s\n' "$tl" | sed 's/^/      /'
      [ -n "$EXIST_TUNNEL_ID" ] \
        && EXIST_TUNNEL_NAME="$(printf '%s\n' "$tl" | awk -v id="$EXIST_TUNNEL_ID" '$1==id {print $2; exit}')"
    fi
  else
    warn "无 cert.pem，查不到远端隧道列表（远程托管模式属正常）"
  fi

  # 6) 已绑定域名的 DNS 实际解析情况
  if [ ${#EXIST_DOMAINS[@]} -gt 0 ]; then
    echo "      --- 域名解析 ---"
    for d in "${EXIST_DOMAINS[@]}"; do
      local ip
      ip="$(python3 -c 'import socket,sys
try: print(socket.gethostbyname(sys.argv[1]))
except Exception: print("")' "$d" 2>/dev/null)"
      echo "      · $d  →  ${ip:-未解析（DNS 记录不存在或未生效）}"
    done
  fi
  return 0
}
# ------------------------------------------------------------------------------

# 带镜像回退的下载：dl <目标路径> <原始URL>
dl() {
  local dest="$1" url="$2" m
  for m in "${GH_MIRRORS[@]}"; do
    local full="${m}${url}"
    [ -z "$m" ] && full="$url"
    log "下载: $full"
    if curl -fsSL --connect-timeout 15 --max-time 300 --retry 2 -o "$dest" "$full"; then
      if [ -s "$dest" ]; then ok "下载成功"; return 0; fi
    fi
    warn "该镜像失败，换下一个"
    rm -f "$dest"
  done
  return 1
}

require_root() {
  [ "$(id -u)" -eq 0 ] || { echo "请用 root 执行: sudo bash $0"; exit 1; }
}

# 从 GitHub API 解析真实资产下载链接（作者改名 / 资产不是固定名字时的兜底）
resolve_asset_url() {
  local api="https://api.github.com/repos/linzixuanzz/daidai-panel/releases/latest"
  local m full asset
  for m in "" "${GH_MIRRORS[@]}"; do
    full="$api"; [ -n "$m" ] && full="${m}${api}"
    asset="$(curl -fsSL --connect-timeout 10 --max-time 30 "$full" 2>/dev/null | python3 -c '
import sys, json
try:
    j = json.load(sys.stdin)
except Exception:
    sys.exit(0)
for a in j.get("assets", []):
    n = a.get("name", "").lower()
    if "linux" in n and ("amd64" in n or "x86_64" in n) and n.endswith((".tar.gz", ".tgz")) and "exe" not in n:
        print(a.get("browser_download_url", "")); break
' 2>/dev/null)"
    if [ -n "$asset" ]; then echo "$asset"; return 0; fi
  done
  echo ""
}

# ------------------------------ 0. 环境准备 ------------------------------
step 0 "环境检查与依赖安装"
require_root
mkdir -p "$(dirname "$LOG_FILE")" "$HELPER_DIR"
exec > >(tee -a "$LOG_FILE") 2>&1          # 全部输出同时落盘，便于事后排查
exec 9>"$LOCK_FILE"
flock -n 9 || die "已有另一个安装进程在运行（锁: $LOCK_FILE）"

# --- 架构 / 系统 ---
ARCH="$(uname -m)"
case "$ARCH" in
  x86_64|amd64) ok "架构检查通过: $ARCH";;
  aarch64|arm64) die "当前为 ARM64 ($ARCH)，本脚本固定资产为 linux-amd64，需换 daidai-linux-arm64 资产";;
  *) die "不支持的架构: $ARCH（本脚本仅适配 x86_64）";;
esac
command -v apt-get >/dev/null 2>&1 || die "本脚本面向 Debian/Ubuntu（依赖 apt-get），当前系统不支持"
[ "$(lsb_release -is 2>/dev/null)" = "Ubuntu" ] \
  || warn "系统不是 Ubuntu（$(lsb_release -ds 2>/dev/null || echo unknown)），未充分测试，继续尝试"

# --- systemd 是否可用（Docker / LXC 容器里常没有，后面要换守护方式）---
HAS_SYSTEMD=0
if [ -d /run/systemd/system ] && command -v systemctl >/dev/null 2>&1; then HAS_SYSTEMD=1; fi
[ "$HAS_SYSTEMD" -eq 1 ] && ok "systemd 可用" \
  || warn "未检测到 systemd（容器环境），面板与隧道将改用 nohup + cron 方式守护"

export DEBIAN_FRONTEND=noninteractive
if ! command -v curl >/dev/null 2>&1; then
  apt-get update -qq && apt-get install -y -qq curl
fi
apt-get install -y -qq python3 tar ca-certificates cron procps 2>/dev/null || true
systemctl enable --now cron 2>/dev/null || { service cron start 2>/dev/null; (crond 2>/dev/null &); }
command -v crontab >/dev/null 2>&1 || warn "未安装 cron/crontab，计划任务将不会执行"
timedatectl set-timezone Asia/Shanghai 2>/dev/null || true

# --- 磁盘空间 ---
AVAIL_KB="$(df -P /opt 2>/dev/null | awk 'NR==2{print $4}')"
if [ -n "$AVAIL_KB" ] && [ "$AVAIL_KB" -lt 524288 ]; then
  warn "/opt 剩余空间不足 512MB（现 $(( AVAIL_KB / 1024 ))MB），安装可能失败"
fi

# --- 端口占用 ---
if command -v ss >/dev/null 2>&1 && ss -ltn 2>/dev/null | grep -qE ":${PANEL_PORT}[[:space:]]"; then
  warn "端口 $PANEL_PORT 已被占用，若不是呆呆面板自身请先释放"
fi

# --- 时间偏差（cloudflared 证书校验严格，时间不准会直接连不上，是高频坑）---
timedatectl set-ntp on 2>/dev/null || true
SKEW="$(python3 - <<'PY' 2>/dev/null
import time, urllib.request, email.utils
try:
    r = urllib.request.urlopen('https://www.cloudflare.com', timeout=8)
    d = r.headers.get('Date')
    print(int(abs(email.utils.parsedate_to_datetime(d).timestamp() - time.time())) if d else 0)
except Exception:
    print(0)
PY
)"
SKEW="${SKEW:-0}"
if [ "$SKEW" -gt 300 ] 2>/dev/null; then
  warn "系统时间与网络时间相差 ${SKEW} 秒！cloudflared 会因证书时间校验失败而连不上"
  apt-get install -y -qq systemd-timesyncd chrony 2>/dev/null || true
  systemctl restart systemd-timesyncd 2>/dev/null || true
  chronyd -q 'server time.cloudflare.com iburst' 2>/dev/null || ntpdate -u time.cloudflare.com 2>/dev/null || true
  warn "已尝试自动校时；若仍失败请手动执行 timedatectl set-ntp on 后重跑"
fi

# --- 防火墙出站（cloudflared 需要出站 443 / 7844）---
if command -v ufw >/dev/null 2>&1 && ufw status 2>/dev/null | grep -q '^Status: active'; then
  warn "UFW 已启用，请确认允许出站 443/7844，否则隧道连不上（ufw allow out 443/tcp）"
fi

ok "基础依赖就绪 ($(uname -m) / $(lsb_release -ds 2>/dev/null || echo Ubuntu))"

if [ -z "$CF_DOMAINS" ] && [ -z "$CF_DOMAIN_TPL" ]; then
  if [ "$CF_SURVEY_ONLY" = "1" ]; then
    # 体检模式不需要域名：目的就是看机器上现有什么，不该反过来要求输入
    warn "体检模式：未指定域名，跳过（本次只检测现有隧道与域名）"
  else
    echo "域名怎么给？（二选一）"
    echo "  1) 固定域名：dd.your.com（多域名逗号分隔）"
    echo "  2) 域名模板：yb{N}.4g.cc.cd —— {N} 会自动换成机器编号，适合多台批量上线"
    read -r -p "请输入(直接回车表示用模板方式): " CF_DOMAIN_TPL
    [ -n "$CF_DOMAIN_TPL" ] || die "未提供域名或域名模板"
  fi
fi
# 归一化成数组：逗号/空格分隔
IFS=', ' read -r -a DOMAIN_LIST <<< "$CF_DOMAINS"
DOMAIN_LIST=($(printf '%s\n' "${DOMAIN_LIST[@]}" | sed '/^$/d' | awk '!seen[$0]++'))
if [ ${#DOMAIN_LIST[@]} -gt 0 ]; then
  ok "待映射域名 (${#DOMAIN_LIST[@]} 个): ${DOMAIN_LIST[*]}"
else
  ok "使用域名模板: $CF_DOMAIN_TPL（编号在步骤 6 确定后替换）"
fi

# ---- 开工前先摸清这台机器有没有现成的隧道 / 域名，避免一上来就把人家的隧道干掉 ----
# 先读本机安装记录，后面据此判断现有隧道归谁
if [ -f "$STATE_FILE" ]; then . "$STATE_FILE" 2>/dev/null || true; fi
# 端口必须在体检「之前」定准：体检要用它匹配隧道上指向面板端口的现有域名，
# 拿默认值去匹配会把 :5701 的现成映射漏判成「没有可用域名」
detect_panel_port
survey_tunnel

# ---------- 面板密码 ----------
# 优先级：环境变量 PANEL_PASS > 本机上次记录（.panel_pass，600）> 随机生成。
# 随机生成后落盘复用，否则重跑一次脚本密码就变一次，已初始化的面板就登不进去了。
# 体检模式全程只读：不生成、不落盘密码
if [ "${CF_SURVEY_ONLY:-0}" = "1" ]; then
  PANEL_PASS="${PANEL_PASS:-}"
else
  if [ -n "${PANEL_PASS:-}" ]; then
    ok "面板密码: 来自 PANEL_PASS（长度 ${#PANEL_PASS}）"
  elif [ -s "$PANEL_PASS_FILE" ]; then
    PANEL_PASS="$(head -n1 "$PANEL_PASS_FILE" 2>/dev/null | tr -d '\r\n')"
    [ -n "$PANEL_PASS" ] && ok "面板密码: 沿用本机记录（长度 ${#PANEL_PASS}）"
  fi
  if [ -z "${PANEL_PASS:-}" ]; then
    PANEL_PASS="$(tr -dc 'A-Za-z0-9' < /dev/urandom 2>/dev/null | head -c 16)"
    [ ${#PANEL_PASS} -ge 12 ] || PANEL_PASS="$(od -An -N16 -tx1 /dev/urandom 2>/dev/null | tr -d ' \n')"
    [ ${#PANEL_PASS} -ge 12 ] || PANEL_PASS="$(date +%s%N 2>/dev/null | md5sum 2>/dev/null | head -c 16)"
    [ ${#PANEL_PASS} -ge 12 ] || die "随机密码生成失败，请用 PANEL_PASS=xxx 显式指定"
    ok "面板密码: 已随机生成（长度 ${#PANEL_PASS}）"
  fi
  # 存一份供重跑复用（600，只本机可读）
  mkdir -p "$HELPER_DIR" 2>/dev/null
  umask 077
  printf '%s\n' "$PANEL_PASS" > "$PANEL_PASS_FILE" 2>/dev/null
  chmod 600 "$PANEL_PASS_FILE" 2>/dev/null
  umask 022
fi

# 现有隧道是不是本脚本建的：token 模式没有 cert.pem、查不到隧道名，UUID 是唯一可靠判据
TUNNEL_IS_OURS=0
if [ -n "${SAVED_TID:-}" ] && [ -n "$EXIST_TUNNEL_ID" ] && [ "$SAVED_TID" = "$EXIST_TUNNEL_ID" ]; then
  TUNNEL_IS_OURS=1
fi

CF_REUSE_EXISTING=0
if [ "$CF_RUNNING" -eq 1 ] && [ ${#EXIST_DOMAINS[@]} -gt 0 ] \
   && { [ "$CF_TUNNEL_MODE" = "reuse" ] \
        || { [ "$CF_TUNNEL_MODE" = "auto" ] && [ "$TUNNEL_IS_OURS" -eq 0 ]; }; }; then
  CF_REUSE_EXISTING=1
  ok "=> 检测到可用隧道（UUID ${EXIST_TUNNEL_ID:-未知}）：本次【复用】，不新建、不卸载、不改现有映射"
  # 没指定域名时，只沿用「已指向面板端口」的现有域名。
  # 不能拿指向别的服务（如 20447）的域名来做面板连通性检测——那会 200 但根本不是面板。
  if [ ${#DOMAIN_LIST[@]} -eq 0 ]; then
    _USABLE=()
    for _pair in "${EXIST_DOMAIN_SERVICES[@]}"; do
      case "${_pair#*|}" in *":$PANEL_PORT") _USABLE+=("${_pair%%|*}");; esac
    done
    if [ ${#_USABLE[@]} -gt 0 ]; then
      DOMAIN_LIST=("${_USABLE[@]}")
      ok "=> 未指定 CF_DOMAINS，沿用现有已指向 :$PANEL_PORT 的域名: ${DOMAIN_LIST[*]}"
    else
      warn "=> 现有隧道上没有指向面板端口 :$PANEL_PORT 的域名"
      warn "   请用 CF_DOMAINS=<新域名> 指定，例如 CF_DOMAINS=yb1.poc.dpdns.org"
      warn "   给了 CF_API_TOKEN+CF_ACCOUNT_ID 时脚本会自动追加到现有隧道；否则需到后台手动加"
    fi
  fi
  # 只给了模板（yb{N}.xxx）时需要个编号才能渲染；优先手动指定 > 本机上次编号 > 1
  if [ ${#DOMAIN_LIST[@]} -eq 0 ] && [ -n "$CF_DOMAIN_TPL" ]; then
    [ -f "$STATE_FILE" ] && . "$STATE_FILE" 2>/dev/null || true
    _NUM="${CF_MACHINE_NUM:-${SAVED_NUM:-1}}"
    IFS=',' read -r -a _TPLS <<< "$CF_DOMAIN_TPL"
    DOMAIN_LIST=()
    for _t in "${_TPLS[@]}"; do
      _t="$(echo "$_t" | tr -d ' ')"
      [ -n "$_t" ] && DOMAIN_LIST+=("${_t//\{N\}/$_NUM}")
    done
    ok "=> 复用模式：按模板渲染域名（编号 $_NUM）: ${DOMAIN_LIST[*]}"
  fi
elif [ "$CF_TUNNEL_MODE" = "new" ]; then
  warn "=> CF_TUNNEL_MODE=new：步骤 6 将重建隧道并替换现有域名映射"
  [ "$CF_RUNNING" -eq 1 ] && warn "   现有隧道正在跑，重建会让它上面所有域名立刻断线"
elif [ "$CF_TUNNEL_MODE" = "auto" ] && [ "$TUNNEL_IS_OURS" -eq 1 ]; then
  warn "=> auto：隧道 ${EXIST_TUNNEL_ID} 是本脚本创建的 → 允许重建（需 CF_API_TOKEN 或 cert.pem）"
else
  warn "=> 现有隧道不满足复用条件（未运行 或 未绑定域名），步骤 6 将新建隧道"
fi

# 卸载模式：不装任何东西，按 UNINSTALL_SCOPE 清理本机（+可选云端）
# 放在体检之后：survey_tunnel 要用 step 0 探测到的 HAS_SYSTEMD，且 root/日志已就绪
if [ "${UNINSTALL:-0}" = "1" ]; then
  run_uninstall
  exit $?
fi

# 只体检不安装
if [ "$CF_SURVEY_ONLY" = "1" ]; then
  echo
  echo -e "${C_G}===== 体检结束（全程只读，未做任何改动）=====${C_0}"
  exit 0
fi


# ---------- 自选安装：选择本次要执行的步骤 ----------
# 非交互时用环境变量指定：STEPS="1,3,4" 或 STEPS=all
if [ -z "${STEPS:-}" ]; then
  if [ -t 0 ]; then
    echo
    echo -e "${C_G}----- 选择本次要执行的步骤 -----${C_0}"
    echo "  1) 安装呆呆面板"
    echo "  2) 面板开机自启 (systemd + @reboot 双重保活)"
    echo "  3) 初始化面板账号 admin"
    echo "  4) 写入元宝保活脚本 + 建每天执行的计划任务"
    echo "  5) 安装 cloudflared"
    echo "  6) 创建/复用 Cloudflare 隧道并映射域名"
    echo "  7) 检测公网域名是否可访问"
    echo
    echo "  直接回车 = 全部执行；也可写 1,3,4 只跑指定步骤"
    read -r -p "请选择: " STEPS
    case "$STEPS" in a|A|"") STEPS="all";; esac
  else
    STEPS="all"      # 非交互环境默认全跑，避免卡在 read
  fi
fi
case "$STEPS" in
  all) ok "本次执行: 全部步骤 1-7";;
  *)   ok "本次执行步骤: $STEPS"
       case ",$STEPS," in
         *",4,"*) case ",$STEPS," in *",3,"*) ;; *) warn "步骤 4 会用 admin 账号登录建任务；若面板尚未初始化，请先跑步骤 3";; esac;;
       esac
       case ",$STEPS," in
         *",7,"*) case ",$STEPS," in *",6,"*) ;; *) warn "步骤 7 只做域名检测，未选 6 时请确保隧道与域名已就绪";; esac;;
       esac
       ;;
esac
# ==============================================================================
run_step1() {
step 1 "安装呆呆面板 (daidai-linux-amd64 最新版)"
# ==============================================================================
mkdir -p "$PANEL_DIR" "$HELPER_DIR" /tmp/dd-install
cd /tmp/dd-install || die "无法进入临时目录"

# 不同版本二进制名不一样：daidai-server / daidai-linux-amd64 / dumb-panel …
# 只认 daidai-server 会在已装机器上误判成「未安装」，进而覆盖掉现有面板和数据
EXISTING_BIN="$(find "$PANEL_DIR" -maxdepth 1 -type f -perm -u+x \
                 \( -name 'daidai*' -o -name 'dumb*' \) ! -name '*.tar.gz' 2>/dev/null | head -n1)"
if [ -n "$EXISTING_BIN" ]; then
  ok "检测到已安装（$(basename "$EXISTING_BIN")），跳过下载（如需重装请先删除 $PANEL_DIR）"
else
  if [ -n "$LOCAL_TARBALL" ] && [ -f "$LOCAL_TARBALL" ]; then
    ok "使用本地离线包: $LOCAL_TARBALL"
    cp -f "$LOCAL_TARBALL" daidai.tar.gz || die "复制本地包失败"
  else
    dl "daidai.tar.gz" "$RELEASE_URL"
    if [ ! -s daidai.tar.gz ]; then
      warn "固定资产名下载失败，尝试从 GitHub API 解析真实资产名（作者可能改了文件名）"
      ASSET_URL="$(resolve_asset_url)"
      [ -n "$ASSET_URL" ] || die "所有镜像均下载失败，且 GitHub API 也解析不到资产。可手动下载后改用 LOCAL_TARBALL=/路径/xxx.tar.gz 重跑"
      log "解析到资产: $ASSET_URL"
      dl "daidai.tar.gz" "$ASSET_URL"
      [ -s daidai.tar.gz ] || die "下载解析到的资产仍失败: $ASSET_URL"
    fi
  fi
  tar -tzf daidai.tar.gz >/dev/null 2>&1 || die "下载到的文件不是有效 tar.gz（可能被镜像返回了 HTML 错误页）"
  rm -rf extract && mkdir extract
  tar -xzf daidai.tar.gz -C extract || die "解压失败"
  # 定位二进制所在目录，抹掉顶层目录（排除 Windows 资产）
  # 注意：新版资产二进制名是 daidai-linux-amd64，只认 daidai-server 会找不到
  SRC_DIR="$(find extract -type f \( -name 'daidai*' -o -name 'dumb*' \) \
             ! -name '*.exe' ! -name '*.dll' ! -name '*.tar.gz' ! -name '*.md' \
             -printf '%h\n' | head -n1)"
  [ -n "$SRC_DIR" ] || die "压缩包内未找到面板二进制（daidai* / dumb*），请检查 release 资产是否正确"
  cp -rf "$SRC_DIR"/. "$PANEL_DIR"/ || die "复制文件到 $PANEL_DIR 失败"
  chmod +x "$PANEL_DIR"/daidai-server "$PANEL_DIR"/ddp 2>/dev/null
  ok "面板文件已安装到 $PANEL_DIR"
fi

# 统一软链成 daidai-server，后面所有步骤只认这一个名字
REAL_BIN="$(find "$PANEL_DIR" -maxdepth 1 -type f -perm -u+x \
             \( -name 'daidai*' -o -name 'dumb*' \) ! -name '*.tar.gz' 2>/dev/null | head -n1)"
if [ -n "$REAL_BIN" ] && [ "$(basename "$REAL_BIN")" != "daidai-server" ]; then
  ln -sf "$(basename "$REAL_BIN")" "$PANEL_DIR/daidai-server" 2>/dev/null \
    && ok "已把 $(basename "$REAL_BIN") 软链为 daidai-server"
fi

ls -1 "$PANEL_DIR" | sed 's/^/      - /'
"$PANEL_DIR"/ddp version 2>/dev/null | head -n2 || warn "ddp version 无法执行（不影响后续）"
}

# ==============================================================================
run_step2() {
step 2 "启动面板并配置开机自启动 (systemd + @reboot 双重保活)"
# ==============================================================================
# 端口以实际为准（同体检阶段那个探测函数；这里再跑一次，面板可能刚装上、端口此时才确定）
detect_panel_port

# 面板可能已被 ddp service install 装成别的 service 名，先停掉避免两个进程抢端口
for _s in daidai ddp dumb-panel dumbpanel; do
  systemctl stop "$_s" 2>/dev/null && warn "已停止可能冲突的旧服务: $_s"
done

if [ "$HAS_SYSTEMD" -eq 1 ]; then
cat > /etc/systemd/system/daidai-panel.service <<EOF
[Unit]
Description=Daidai Panel (呆呆面板)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=$PANEL_DIR
ExecStart=$PANEL_DIR/daidai-server
Restart=always
RestartSec=5
KillSignal=SIGINT
LimitNOFILE=65535
Environment=TZ=Asia/Shanghai

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable daidai-panel.service >/dev/null 2>&1
systemctl restart daidai-panel.service
ok "systemd 服务已创建并设为开机自启"
# 面板自带的服务安装命令，成功则让它接管（失败忽略）
"$PANEL_DIR"/ddp service install >/dev/null 2>&1 && ok "ddp service install 成功" || true
else
  log "无 systemd，改用后台进程方式启动面板"
  pkill -f "$PANEL_DIR/daidai-server" 2>/dev/null || true
  sleep 1
  ( cd "$PANEL_DIR" && nohup ./daidai-server >> /var/log/daidai-panel.log 2>&1 & )
  ok "面板已后台启动（日志: /var/log/daidai-panel.log）"
fi

# 额外一层：@reboot 保活（面板进程若被 kill，5 分钟内自动拉起）
# 同一个脚本同时兼容 systemd 与容器（无 systemd）两种环境
cat > /usr/local/bin/daidai-keepalive.sh <<KEEPEOF
#!/usr/bin/env bash
# 呆呆面板存活守护（由一键安装脚本生成）
DIR="$PANEL_DIR"
PORT="$PANEL_PORT"
HAS_SD=0
[ -d /run/systemd/system ] && command -v systemctl >/dev/null 2>&1 && HAS_SD=1

have_unit() { [ "\$HAS_SD" -eq 1 ] && systemctl list-unit-files 2>/dev/null | grep -q '^daidai-panel'; }
start_panel() {
  if have_unit; then
    systemctl restart daidai-panel
  else
    pgrep -f "\$DIR/daidai-server" >/dev/null 2>&1 \\
      || ( cd "\$DIR" && nohup ./daidai-server >> /var/log/daidai-panel.log 2>&1 & )
  fi
}
if have_unit; then
  systemctl is-active --quiet daidai-panel || systemctl restart daidai-panel
else
  pgrep -f "\$DIR/daidai-server" >/dev/null 2>&1 || start_panel
fi
curl -s -o /dev/null --max-time 5 --noproxy '*' "http://127.0.0.1:\$PORT/api/auth/check-init" 2>/dev/null || start_panel
KEEPEOF
chmod +x /usr/local/bin/daidai-keepalive.sh
( crontab -l 2>/dev/null | grep -v 'daidai-keepalive.sh'; \
  echo "@reboot sleep 20 && /usr/local/bin/daidai-keepalive.sh"; \
  echo "*/5 * * * * /usr/local/bin/daidai-keepalive.sh" ) | crontab -
ok "已加入 crontab 开机自启 + 每 5 分钟存活检查"

# 等待面板就绪：用真实存在的免认证接口探测（面板没有 /api/health）
panel_alive() {
  local p code
  for p in /api/auth/check-init /api/health /api/system/public-version /; do
    code="$(curl -s -o /dev/null -w '%{http_code}' --max-time 3 --noproxy '*' \
            "http://127.0.0.1:$PANEL_PORT$p" 2>/dev/null)"
    if [ -n "$code" ] && [ "$code" != "000" ]; then return 0; fi
  done
  return 1
}
log "等待面板监听 127.0.0.1:$PANEL_PORT ..."
PANEL_UP=0
for i in $(seq 1 60); do
  if panel_alive; then
    PANEL_UP=1; ok "面板已就绪（等待 $((i*2))s）"
    PANEL_VER="$(curl -s --max-time 5 --noproxy '*' \
                 "http://127.0.0.1:$PANEL_PORT/api/system/public-version" 2>/dev/null \
                 | grep -oE '"version":"[^"]+"' | head -n1 | cut -d'"' -f4)"
    [ -n "$PANEL_VER" ] && ok "面板版本: $PANEL_VER"
    break
  fi
  sleep 2
done
if [ "$PANEL_UP" -ne 1 ]; then
  err "面板 120 秒内未启动，诊断信息："
  echo "    · 当前监听端口: $(ss -ltnp 2>/dev/null | grep -i daidai || echo '未发现 daidai 相关监听')"
  if [ "$HAS_SYSTEMD" -eq 1 ]; then
    journalctl -u daidai-panel -n 30 --no-pager 2>/dev/null
  else
    tail -n 30 /var/log/daidai-panel.log 2>/dev/null
  fi
  die "面板启动失败，请确认 $PANEL_DIR/daidai-server 可执行且端口 $PANEL_PORT 未被占用"
fi
BASE_URL="http://127.0.0.1:$PANEL_PORT"
}

# ==============================================================================
run_step3() {
step 3 "初始化面板账号（密码随机生成或 PANEL_PASS 指定）"
# ==============================================================================
cat > "$HELPER_DIR/panel_api.py" <<'PYEOF'
#!/usr/bin/env python3
"""呆呆面板 API 交互助手：自动探测可用的路由与字段组合"""
import json, sys, urllib.request, urllib.error

BASE = sys.argv[1]

def req(method, path, data=None, token=None, timeout=30):
    url = BASE + path
    body = json.dumps(data).encode() if data is not None else None
    r = urllib.request.Request(url, data=body, method=method)
    r.add_header('Content-Type', 'application/json')
    if token:
        r.add_header('Authorization', 'Bearer ' + token)
    try:
        with urllib.request.urlopen(r, timeout=timeout) as resp:
            raw = resp.read().decode('utf-8', 'ignore')
            return resp.getcode(), raw
    except urllib.error.HTTPError as e:
        return e.code, e.read().decode('utf-8', 'ignore')
    except Exception as e:
        return 0, '{"error":"%s"}' % e

def good(code, raw):
    if code == 0 or code >= 300:
        return False
    try:
        j = json.loads(raw)
    except Exception:
        return 200 <= code < 300
    c = j.get('code')
    if c is None:
        return True
    return str(c) in ('0', '200', '201', 'success')

def find_token(obj):
    if isinstance(obj, dict):
        for k in ('token', 'access_token', 'accessToken', 'jwt', 'id_token'):
            v = obj.get(k)
            if isinstance(v, str) and len(v) > 20:
                return v
        for v in obj.values():
            t = find_token(v)
            if t: return t
    elif isinstance(obj, list):
        for v in obj:
            t = find_token(v)
            if t: return t
    return None

def cmd_checkinit():
    for p in ['/api/auth/check-init', '/api/v1/auth/check-init']:
        code, raw = req('GET', p)
        if good(code, raw):
            try:
                j = json.loads(raw)
                v = j.get('need_init')
                if v is None:
                    v = (j.get('data') or {}).get('need_init')
                print("need_init=%s" % (v is True or str(v).lower() == 'true'))
                return 0
            except Exception:
                pass
    print("need_init=unknown"); return 0

def cmd_init(user, pwd):
    payloads = [
        {"username": user, "password": pwd},
        {"username": user, "password": pwd, "confirmPassword": pwd},
    ]
    paths = ['/api/auth/init', '/api/v1/auth/init']
    for p in paths:
        for d in payloads:
            code, raw = req('POST', p, d)
            if good(code, raw):
                print("OK %s %s" % (p, json.dumps(d, ensure_ascii=False))); return 0
    print("FAIL"); return 1

def cmd_login(user, pwd):
    for p in ['/api/auth/login', '/api/v1/auth/login']:
        for d in ({"username": user, "password": pwd}, {"name": user, "password": pwd}):
            code, raw = req('POST', p, d)
            if good(code, raw):
                try:
                    t = find_token(json.loads(raw))
                except Exception:
                    t = None
                if t:
                    print(t); return 0
    print(""); return 1

def cmd_savescript(token, relpath, content):
    payloads = [
        {"path": relpath, "content": content},
        {"path": relpath, "content": content, "overwrite": True},
    ]
    for p in ['/api/scripts/content', '/api/v1/scripts/content']:
        for d in payloads:
            for m in ('PUT', 'POST'):
                code, raw = req(m, p, d, token)
                if good(code, raw):
                    print("OK %s %s" % (m, p)); return 0
    print("FAIL"); return 1

def cmd_createtask(token, name, script, cron, pyver='3.12'):
    # command 格式各版本不一样，两种都要试：
    #   v2.x（实测 2.3.2）: "task ybbh_keepalive.py"
    #   v3.x              : 完整 shell 命令 "python3 /abs/path/xxx.py"
    # status: 1 = 启用（模型里 0 是禁用，不传可能建出来是禁用状态、永远不会跑）
    # allow_multiple_instances: false 配合脚本内的跳过窗口，避免并发重复发送
    import os as _os
    _bn = _os.path.basename(script)
    base = {"name": name, "command": "python3 " + script,
            "task_type": "cron", "cron_expression": cron,
            "python_version": pyver, "timeout": 0,
            "success_exit_codes": "0", "max_retries": 0,
            "retry_interval": 60, "notify_on_failure": True,
            "notify_on_success": False,
            "status": 1, "allow_multiple_instances": False}
    payloads = [
        dict(base, command="task " + _bn),                 # v2.x：task 脚本名
        dict(base, command="task " + script),              # v2.x 变体：task 带路径
        base,                                              # v3.x：完整 shell 命令
        dict(base, python_version=""),                     # 空 = 交给面板默认解释器
        dict(base, command=script),                        # 面板自动补解释器
        {k: v for k, v in base.items() if k != "python_version"},
    ]
    for p in ['/api/tasks', '/api/v1/tasks']:
        for d in payloads:
            code, raw = req('POST', p, d, token)
            if not good(code, raw):
                continue
            # 建出来可能是禁用状态（status!=1），补一次启用，否则任务永远不跑
            tid, st = None, None
            try:
                j = json.loads(raw)
                obj = j.get('data', j) if isinstance(j.get('data', j), dict) else j
                tid, st = obj.get('id'), obj.get('status')
            except Exception:
                pass
            enabled = ''
            if tid is not None and st != 1:
                for ep in ['%s/%s/enable' % (p, tid), '/api/v1/tasks/%s/enable' % tid]:
                    c2, r2 = req('PUT', ep, {}, token)
                    if good(c2, r2):
                        enabled = '（已补调用 enable 启用）'
                        break
            print("OK %s %s%s" % (p, json.dumps(d, ensure_ascii=False), enabled))
            return 0
    print("FAIL")
    return 1

def cmd_listtasks(token):
    for p in ['/api/tasks', '/api/v1/tasks']:
        code, raw = req('GET', p, None, token)
        if good(code, raw):
            try:
                j = json.loads(raw)
                data = j.get('data', j)
                items = data.get('items', data.get('list', data)) if isinstance(data, dict) else data
                if isinstance(items, list):
                    for t in items:
                        if isinstance(t, dict):
                            print("%s\t%s" % (t.get('id'), t.get('name')))
            except Exception:
                pass
            return 0
    return 1

def cmd_deltask(token, taskid):
    """删除面板定时任务（卸载保活时用）。"""
    for p in ['/api/tasks/%s' % taskid, '/api/v1/tasks/%s' % taskid]:
        code, raw = req('DELETE', p, None, token)
        if good(code, raw):
            print("OK %s" % p); return 0
    print("FAIL"); return 1

if __name__ == '__main__':
    fn = sys.argv[2]
    globals()['cmd_' + fn](*sys.argv[3:])
PYEOF
chmod +x "$HELPER_DIR/panel_api.py"
sed -i 's/\r$//' "$HELPER_DIR/panel_api.py"

NEED_INIT="$(python3 "$HELPER_DIR/panel_api.py" "$BASE_URL" checkinit 2>/dev/null | tail -n1)"
if [ "$NEED_INIT" = "need_init=True" ] || [ "$NEED_INIT" = "need_init=unknown" ]; then
  INIT_OUT="$(python3 "$HELPER_DIR/panel_api.py" "$BASE_URL" init "$PANEL_USER" "$PANEL_PASS" 2>/dev/null | tail -n1)"
  [ "${INIT_OUT:0:2}" = "OK" ] && ok "管理员已创建: $PANEL_USER ($INIT_OUT)" \
    || warn "初始化接口未命中（可能已初始化），继续尝试登录"
else
  ok "面板已初始化，跳过创建管理员"
fi

TOKEN="$(python3 "$HELPER_DIR/panel_api.py" "$BASE_URL" login "$PANEL_USER" "$PANEL_PASS" 2>/dev/null | tail -n1)"
if [ ${#TOKEN} -lt 20 ]; then
  # 登录失败 → 用 ddp 把已有账号密码重置为指定密码
  warn "登录失败，尝试用 ddp 重置已有账号密码"
  EXIST_USER="$("$PANEL_DIR"/ddp list-users 2>/dev/null | awk 'NR>1 && $1!="" {print $1; exit}')"
  [ -n "$EXIST_USER" ] || EXIST_USER="admin"
  # 不同版本 ddp 改密码的子命令名不一致，逐个试
  RESET_OK=0
  for _sub in "reset-password" "reset-pwd" "resetpasswd" "user reset-password" "passwd"; do
    if "$PANEL_DIR"/ddp $_sub "$EXIST_USER" "$PANEL_PASS" >/dev/null 2>&1; then
      RESET_OK=1; ok "已用 ddp $_sub 重置 $EXIST_USER 的密码"; break
    fi
  done
  [ "$RESET_OK" -eq 1 ] \
    || die "无法初始化/重置面板账号（账号不存在且 init 接口未命中）。请手动打开 http://服务器IP:$PANEL_PORT 完成初始化后重跑本脚本。"
  if [ "$HAS_SYSTEMD" -eq 1 ]; then
    systemctl restart daidai-panel; sleep 8
  else
    pkill -f "$PANEL_DIR/daidai-server" 2>/dev/null || true; sleep 2
    ( cd "$PANEL_DIR" && nohup ./daidai-server >> /var/log/daidai-panel.log 2>&1 & ); sleep 8
  fi
  TOKEN="$(python3 "$HELPER_DIR/panel_api.py" "$BASE_URL" login "$EXIST_USER" "$PANEL_PASS" 2>/dev/null | tail -n1)"
  [ ${#TOKEN} -ge 20 ] || TOKEN="$(python3 "$HELPER_DIR/panel_api.py" "$BASE_URL" login "$PANEL_USER" "$PANEL_PASS" 2>/dev/null | tail -n1)"
fi
[ ${#TOKEN} -ge 20 ] && ok "面板账号可用，已拿到登录令牌" \
  || die "面板账号初始化/登录失败（账号: $PANEL_USER）。请确认面板版本与 API 路径，或手动网页初始化后重跑。"
echo "$TOKEN" > "$HELPER_DIR/.token"
}

# ==============================================================================
run_step4() {
step 4 "写入元宝保活脚本并创建每天执行一次的计划任务"
# ==============================================================================
# 确定面板脚本目录：优先读配置，其次找已存在的常见数据目录
DATA_DIR="$PANEL_DIR/Dumb-Panel"
if [ -f "$PANEL_DIR/config.yaml" ]; then
  _dd="$(grep -iE '^\s*data_dir' "$PANEL_DIR/config.yaml" | head -n1 | sed -E 's/.*:\s*//; s/["'"'"']//g')"
  [ -n "$_dd" ] && { case "$_dd" in /*) DATA_DIR="$_dd";; *) DATA_DIR="$PANEL_DIR/$_dd";; esac; }
fi
if [ ! -d "$DATA_DIR" ]; then
  for _c in "$PANEL_DIR/Dumb-Panel" "$PANEL_DIR/data" "$PANEL_DIR/Data" "$PANEL_DIR/panel-data"; do
    [ -d "$_c" ] && { DATA_DIR="$_c"; break; }
  done
fi
SCRIPT_DIR="$DATA_DIR/scripts"
mkdir -p "$SCRIPT_DIR"
ok "面板数据目录: $DATA_DIR（脚本目录: $SCRIPT_DIR）"

# 自选安装时可能跳过了步骤 3，这里自己登录一次，否则后面 TOKEN 未定义会直接崩
if [ -z "${TOKEN:-}" ] || [ ${#TOKEN} -lt 20 ]; then
  log "未持有登录令牌，先登录面板"
  TOKEN="$(python3 "$HELPER_DIR/panel_api.py" "$BASE_URL" login "$PANEL_USER" "$PANEL_PASS" 2>/dev/null | tail -n1)"
fi
[ "${#TOKEN}" -ge 20 ] || die "登录面板失败（$BASE_URL），无法写入脚本 / 创建计划任务"

cat > "$SCRIPT_DIR/$KEEPALIVE_SCRIPT" <<'DAIDAI_PYEOF'
#!/usr/bin/env python3
# 元宝机器人保活脚本 v3.4（通道推送 + 失败自动重试版）
#
# 保活策略：
#   · 元宝规则：15 天内没有任何消息，容器会被回收
#   · 每次运行真实发送一条消息；只要没真正发出去，就一直重试，直到成功
#   · 距上次成功越久，重试力度越大（紧急模式最多 12 次）
#   · 成功判据：gateway 日志出现 [C2C] send ok，不依赖 openclaw --json 的字段名
#   · 状态记录在 ~/.openclaw/yuanbao_keepalive_state.json

import subprocess, json, random, os, sys, time, re
from datetime import datetime

SESSIONS_FILE = os.path.expanduser("~/.openclaw/agents/main/sessions/sessions.json")
LOG_FILE      = os.path.expanduser("~/.openclaw/yuanbao_keepalive.log")
STATE_FILE    = os.path.expanduser("~/.openclaw/yuanbao_keepalive_state.json")
GATEWAY_DIR   = "/tmp/openclaw"      # gateway 日志目录，用于确认消息是否真的发出

# ---------------- 保活策略配置 ----------------
RECYCLE_DAYS   = 15     # 元宝回收阈值：15 天无消息
SAFE_DAYS      = 12     # 距上次成功超过这个天数 → 进入紧急模式，加大重试
URGENT_DAYS    = 14     # 距上次成功超过这个天数 → 日志重点告警（快被回收了）
MIN_INTERVAL_H = 12     # 距上次成功不足这么多小时则跳过，避免同一天重复打扰
                        # 每天跑一次（间隔 24h）不会被跳过；想完全不限制就改成 0
                        # （FORCE=1 环境变量可强制发送）
MAX_TRY        = 8      # 常规最大尝试次数
MAX_TRY_URGENT = 12     # 紧急模式最大尝试次数
RETRY_BASE     = 20     # 首次重试等待秒数
RETRY_MAX      = 120    # 单次重试最长等待秒数
RETRY_MAX_URGENT = 45   # 紧急模式下的单次最长等待（等太久会拖到总耗时上限）
DEADLINE_MIN   = 25     # 本轮总耗时上限（分钟），防止一直卡住
CMD_TIMEOUT    = 150    # 单次 openclaw 调用超时（秒）：gateway 启动 + 发送要几十秒
AGENT_TIMEOUT  = 90     # 传给 openclaw --timeout 的值（秒）
SESSION_TRY    = 3      # 读取会话配置的尝试次数

# 自然对话消息池：保证是真实对话，避免被判定为心跳空消息
MESSAGES = [
    "你觉得什么样的生活才算幸福？",
    "最近有什么好看的电影推荐吗？",
    "你会做梦吗？如果会，你会梦到什么？",
    "你最佩服什么样的人？",
    "如果你能拥有一种超能力，你想要什么？",
    "你觉得人工智能会改变世界吗？",
    "你平时喜欢听什么类型的音乐？",
    "你有没有特别想去旅行的地方？",
    "你觉得什么是真正的朋友？",
    "你对未来有什么期待？",
    "如果可以回到过去，你最想改变什么？",
    "你最近读到的一句话，有哪句让你印象很深？",
]


def log(msg):
    ts = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    line = "[%s] %s" % (ts, msg)
    print(line)
    try:
        os.makedirs(os.path.dirname(LOG_FILE), exist_ok=True)
        with open(LOG_FILE, "a") as f:
            f.write(line + "\n")
    except Exception:
        pass


def load_state():
    try:
        with open(STATE_FILE) as f:
            return json.load(f)
    except Exception:
        return {}


def save_state(patch):
    st = load_state()
    st.update(patch)
    try:
        os.makedirs(os.path.dirname(STATE_FILE), exist_ok=True)
        tmp = STATE_FILE + ".tmp"
        with open(tmp, "w") as f:
            json.dump(st, f, ensure_ascii=False, indent=2)
        os.replace(tmp, STATE_FILE)
    except Exception as e:
        log("   ! 状态文件写入失败: %s" % e)


def parse_ts(s):
    try:
        return datetime.fromisoformat(s)
    except Exception:
        return None


def get_session_config():
    """从 sessions.json 自动获取 SESSION_ID 和 USER_ID。

    按 updatedAt 取「最近活跃」的元宝会话，而不是遍历到的第一个：
    文件里往往同时躺着早已失效的群会话（...yuanbao:group:direct:xxx）和
    正常直连会话（...yuanbao:direct:xxx），两者都含 yuanbao 和 direct，
    按遍历顺序取第一个会选中死会话 —— 消息永远发不出去，且不报错。
    """
    with open(SESSIONS_FILE) as f:
        data = json.load(f)

    cands = []
    for key, session in data.items():
        if "yuanbao" not in key:
            continue
        session_id = session.get("sessionId")
        if not session_id:
            continue
        delivery_context = session.get("deliveryContext") or {}
        to_field = delivery_context.get("to", "") or ""
        if "direct:" in to_field:
            user_id = to_field.split("direct:")[1]
        else:
            user_id = key.split(":")[-1]
        if not user_id:
            continue
        cands.append((session.get("updatedAt") or 0, session_id, user_id, key))

    if not cands:
        return None, None

    cands.sort(reverse=True)          # updatedAt 最大的排最前
    updated, session_id, user_id, key = cands[0]
    try:
        ts = datetime.fromtimestamp(updated / 1000).strftime("%Y-%m-%d %H:%M")
    except Exception:
        ts = "未知"
    log("选中会话: %s（最后活跃 %s，候选 %d 个）" % (key, ts, len(cands)))
    return session_id, user_id


def _dig(d, *paths):
    """按多个候选路径依次取值，兼容不同版本 openclaw --json 的输出结构"""
    for p in paths:
        cur = d
        ok = True
        for k in p.split("."):
            if isinstance(cur, dict) and k in cur:
                cur = cur[k]
            else:
                ok = False
                break
        if ok and cur is not None and cur != "" and cur != {} and cur != []:
            return cur
    return None


def _gateway_log_path():
    return os.path.join(GATEWAY_DIR, "openclaw-%s.log" % datetime.now().strftime("%Y-%m-%d"))


def _gateway_sent_ok(start, user_id):
    """从 gateway 日志的 start 位置往后找 [C2C] send ok，确认消息真的发到了元宝。

    这是目前最可靠的送达判据：openclaw --json 的字段名各版本不一致，
    而 [transport] [C2C] send ok 是 yuanbao 插件打出的真实发送结果，带 msgId。
    """
    path = _gateway_log_path()
    try:
        with open(path, "r", errors="ignore") as f:
            f.seek(start)
            tail = f.read()
    except Exception as e:
        log("   ! 读取 gateway 日志失败: %s" % e)
        return False, ""

    key = user_id[:16].lower()      # 账号串很长，取前 16 位定位即可
    for line in tail.splitlines():
        if "send ok" not in line or key not in line.lower():
            continue
        m = re.search(r'"msgId"\s*:\s*"([^"]+)"', line)
        return True, (m.group(1) if m else "")
    return False, ""


def send_keepalive_message(session_id, user_id, message):
    """
    通过元宝通道发送一条消息。

    刻意不传 --session-id：gateway 里同一会话有队列/占用，显式指定会打架；
    改用 --channel yuanbao + --reply-to <账号> 让 gateway 自己路由。
    user_id 必须是带正确大小写的账号（取自 deliveryContext.to），
    用 key 里那段小写账号会一条都发不出去。
    """
    start = os.path.getsize(_gateway_log_path()) if os.path.exists(_gateway_log_path()) else 0

    cmd = ["openclaw", "agent",
           "--agent", "main",
           "--message", message,
           "--channel", "yuanbao",
           "--deliver",
           "--reply-to", user_id,
           "--json",
           "--timeout", str(AGENT_TIMEOUT)]

    result = subprocess.run(cmd, capture_output=True, text=True, timeout=CMD_TIMEOUT)

    sent, msg_id = _gateway_sent_ok(start, user_id)
    log("   gateway: send ok=%s, msgId=%s" % (sent, msg_id or "无"))

    if result.returncode != 0:
        err = (result.stderr or "").strip()[:300] or "返回码 %d" % result.returncode
        log("   openclaw 报错: %s" % err)
        # 命令虽然报错，日志里也可能已经发出去了，以日志为准
        return {"delivery": sent, "reply": "", "error": err, "msg_id": msg_id}

    raw = (result.stdout or "").strip()
    reply = error = ""
    try:
        out = json.loads(raw)
        reply = str(_dig(out, "result.meta.finalAssistantVisibleText",
                         "result.reply", "result.text", "reply", "text") or "")
        error = str(_dig(out, "result.deliveryError", "result.error", "error") or "")
    except Exception:
        error = "输出非 JSON，前 120 字: %s" % raw[:120]
    if error:
        log("   openclaw 报: %s" % error[:200])
    if reply:
        log("   Bot 回复 %d 字" % len(reply))

    return {"delivery": sent, "reply": reply, "error": error, "msg_id": msg_id}


def main():
    log("=" * 46)
    log("开始元宝保活 v3.4（失败自动重试，直到消息真正发出）")

    st = load_state()
    now = datetime.now()
    last = parse_ts(st.get("last_success", ""))

    if last:
        hours = (now - last).total_seconds() / 3600.0
        days = hours / 24.0
        log("上次成功: %s（%.1f 小时前 / %.1f 天）"
            % (last.strftime("%Y-%m-%d %H:%M"), hours, days))
        if hours < MIN_INTERVAL_H and os.environ.get("FORCE") != "1":
            log("距上次成功仅 %.1f 小时（< %dh），跳过本次，避免重复打扰" % (hours, MIN_INTERVAL_H))
            log("如需立即强制发送: FORCE=1 python3 %s" % os.path.basename(__file__))
            log("=" * 46)
            return
    else:
        days = None
        log("未找到上次成功记录，按最紧急处理")

    urgent = (days is None) or (days >= SAFE_DAYS)
    max_try = MAX_TRY_URGENT if urgent else MAX_TRY
    if urgent:
        log("! 紧急模式：距上次成功 %s，重试上限提高到 %d 次"
            % ("未知" if days is None else "%.1f 天" % days, max_try))

    # ---- 获取会话配置（文件可能正在被写入，多试几次）----
    session_id = user_id = None
    for i in range(SESSION_TRY):
        try:
            session_id, user_id = get_session_config()
        except Exception as e:
            log("读取会话配置失败: %s" % e)
        if session_id and user_id:
            break
        if i < SESSION_TRY - 1:
            time.sleep(10)
    if not session_id or not user_id:
        log("X 无法获取 SESSION_ID / USER_ID，检查 %s 里是否存在元宝会话" % SESSIONS_FILE)
        save_state({"last_attempt": now.isoformat(), "last_error": "no_session"})
        log("=" * 46)
        return
    log("SESSION_ID: %s...%s" % (session_id[:8], session_id[-8:]))
    log("USER_ID:    %s...%s" % (user_id[:8], user_id[-8:]))

    # ---- 发送，失败就重试，直到成功 ----
    deadline = time.time() + DEADLINE_MIN * 60
    attempt = 0
    last_err = ""
    used = []

    while attempt < max_try:
        if time.time() >= deadline:
            log("! 已达总耗时上限 %d 分钟，停止重试" % DEADLINE_MIN)
            break

        attempt += 1
        pool = [m for m in MESSAGES if m not in used] or MESSAGES
        msg = random.choice(pool)
        used.append(msg)
        log("第 %d/%d 次尝试: %s" % (attempt, max_try, msg))

        try:
            r = send_keepalive_message(session_id, user_id, msg)
        except subprocess.TimeoutExpired:
            last_err = "openclaw 调用超时（%ds）" % CMD_TIMEOUT
            log("  X " + last_err)
        except FileNotFoundError:
            log("X 找不到 openclaw 命令，重试无意义，直接退出")
            save_state({"last_attempt": now.isoformat(), "last_error": "openclaw_not_found"})
            log("=" * 46)
            return
        except Exception as e:
            last_err = "调用异常: %s" % e
            log("  X " + last_err)
        else:
            # 保活的目的是「15 天内有消息」，只要真的发出去就算成功，
            # 不要求 Bot 回复 —— 回复可能很慢，等它会白白拖长任务耗时
            if r["delivery"]:
                reply = (r["reply"] or "").strip().replace("\n", " ")
                if reply:
                    log("√ 保活成功：消息已发出，元宝端有回复（%d 字，第 %d 次尝试）" % (len(reply), attempt))
                    log("  回复: %s" % (reply[:50] + "..." if len(reply) > 50 else reply))
                else:
                    log("√ 保活成功：消息已发到元宝端（msgId=%s，第 %d 次尝试）"
                        % (r.get("msg_id") or "无", attempt))
                ts = datetime.now().isoformat()
                save_state({"last_success": ts, "last_attempt": ts, "last_error": "",
                            "fail_streak": 0,
                            "total_success": st.get("total_success", 0) + 1})
                log("=" * 46)
                return

            last_err = "消息未发出（gateway 日志里没有 send ok）"
            if r.get("error"):
                last_err += " / %s" % r["error"]
            log("  X " + last_err)

        retry_max = RETRY_MAX_URGENT if urgent else RETRY_MAX
        wait = min(RETRY_BASE * (2 ** (attempt - 1)), retry_max)
        if attempt < max_try and time.time() + wait < deadline:
            log("  %ds 后重试..." % wait)
            time.sleep(wait)

    log("X 已尝试 %d 次仍未收到成功回复，最后错误: %s" % (attempt, last_err))
    save_state({"last_attempt": datetime.now().isoformat(), "last_error": last_err,
                "fail_streak": st.get("fail_streak", 0) + 1})
    if days is None or days >= URGENT_DAYS:
        log("!! 距上次成功已 %s（回收阈值 %d 天），容器随时会被回收！"
            % ("未知" if days is None else "%.1f 天" % days, RECYCLE_DAYS))
        log("   请检查：openclaw 是否可用 / 元宝授权是否过期 / 网络是否通")
    log("=" * 46)


if __name__ == "__main__":
    main()
DAIDAI_PYEOF
# 去掉可能的 Windows 换行符：脚本若在 Windows 上编辑过，行尾 \r 会让 python 直接报错
sed -i 's/\r$//' "$SCRIPT_DIR/$KEEPALIVE_SCRIPT"
chmod +x "$SCRIPT_DIR/$KEEPALIVE_SCRIPT"
if command -v openclaw >/dev/null 2>&1; then
  ok "openclaw 已就绪: $(command -v openclaw)"
  SESS="$HOME/.openclaw/agents/main/sessions/sessions.json"
  [ -f "$SESS" ] || warn "未找到 $SESS —— 请先完成元宝会话授权，否则保活脚本读不到 session 会空跑"
else
  warn "未检测到 openclaw 命令，保活脚本执行会失败，请先安装 openclaw"
fi
ok "保活脚本已写入: $SCRIPT_DIR/$KEEPALIVE_SCRIPT"

# 通过 API 同步到面板（失败也不影响，文件已在脚本目录）
SAVE_OUT="$(python3 "$HELPER_DIR/panel_api.py" "$BASE_URL" savescript "$TOKEN" "$KEEPALIVE_SCRIPT" "$(cat "$SCRIPT_DIR/$KEEPALIVE_SCRIPT")" 2>/dev/null | tail -n1)"
[ "${SAVE_OUT:0:2}" = "OK" ] && ok "脚本已通过面板 API 写入 ($SAVE_OUT)" || warn "API 写入未命中，已直接落盘到脚本目录（面板刷新即可见）"

# 创建计划任务：每天执行一次
# 宽松匹配：只要已存在任何「保活 / keepalive」任务就不再新建，
# 否则任务名差一个字（元宝保活 vs 元宝自动保活）就会建出第二份，导致重复发送
EXIST_TASK="$(python3 "$HELPER_DIR/panel_api.py" "$BASE_URL" listtasks "$TOKEN" 2>/dev/null \
              | grep -icE '保活|keepalive')"
if [ "${EXIST_TASK:-0}" -gt 0 ]; then
  ok "计划任务已存在，跳过创建"
else
  # command 用脚本绝对路径（面板执行时工作目录不保证是脚本目录）
  PY_VER="$(python3 -c 'import sys;print("%d.%d" % sys.version_info[:2])' 2>/dev/null || echo 3.12)"
  TASK_OUT="$(python3 "$HELPER_DIR/panel_api.py" "$BASE_URL" createtask "$TOKEN" "$TASK_NAME" \
              "$SCRIPT_DIR/$KEEPALIVE_SCRIPT" "$TASK_CRON" "$PY_VER" 2>/dev/null | tail -n1)"
  if [ "${TASK_OUT:0:2}" = "OK" ]; then
    ok "计划任务已创建: $TASK_OUT"
  else
    warn "面板 API 创建任务未命中，追加系统 crontab 兜底（cron: $TASK_CRON）"
    ( crontab -l 2>/dev/null | grep -v "$KEEPALIVE_SCRIPT"; \
      echo "$TASK_CRON cd $SCRIPT_DIR && python3 $KEEPALIVE_SCRIPT >> /var/log/yuanbao_keepalive.log 2>&1" ) | crontab -
    ok "系统 crontab 已添加保活任务（面板内请手动新建任务: 脚本 $KEEPALIVE_SCRIPT, cron $TASK_CRON）"
  fi
fi
}

# ==============================================================================
run_step5() {
step 5 "安装 cloudflared"
# ==============================================================================
if command -v cloudflared >/dev/null 2>&1; then
  ok "cloudflared 已安装: $(cloudflared --version 2>/dev/null | head -n1)"
else
  dl "/tmp/dd-install/cloudflared.deb" "$CF_DEB_URL" || die "cloudflared 下载失败（所有镜像均不可用）"
  dpkg -i /tmp/dd-install/cloudflared.deb >/dev/null 2>&1 || apt-get -y -qq -f install >/dev/null 2>&1
  command -v cloudflared >/dev/null 2>&1 || die "cloudflared 安装失败"
  ok "cloudflared 安装完成: $(cloudflared --version 2>/dev/null | head -n1)"
fi
}

# ==============================================================================
run_step6() {
step 6 "创建 Cloudflare Tunnel 并映射域名"
# ==============================================================================
# 默认复用（CF_REUSE_EXISTING=1）：有可用隧道就原样保留，只按需追加域名。
# 只有 CF_TUNNEL_MODE=new，或 auto 且隧道是本脚本建的，才会重建并替换域名映射。
if systemctl is-active --quiet cloudflared 2>/dev/null; then
  if [ "${CF_REUSE_EXISTING:-0}" -eq 1 ]; then
    ok "检测到 cloudflared 已在运行 → 复用，不重建（要重建请用 CF_TUNNEL_MODE=new）"
  else
    warn "检测到 cloudflared 已在运行：本步骤会重建隧道，现有域名映射会被替换"
    warn "若只想装面板、保留现有隧道，请设置 CF_TUNNEL_MODE=reuse（默认）或不给 CF_API_TOKEN / CF_TUNNEL_TOKEN"
  fi
fi
mkdir -p /etc/cloudflared

# ---- Cloudflare API 助手（模式 C 用）----
cat > "$HELPER_DIR/cf_api.py" <<'CFEOF'
#!/usr/bin/env python3
"""Cloudflare API 助手：自动建隧道、拿 token、配 ingress、写 DNS"""
import json, os, sys, urllib.request, urllib.error
API = "https://api.cloudflare.com/client/v4"
TOKEN = sys.argv[1]
# 给了 CF_EMAIL 就走 Global API Key（X-Auth-Email + X-Auth-Key），否则走 Bearer API Token
EMAIL = os.environ.get('CF_EMAIL', '').strip()

def call(method, path, data=None):
    r = urllib.request.Request(API + path,
        data=json.dumps(data).encode() if data is not None else None, method=method)
    if EMAIL:
        r.add_header('X-Auth-Email', EMAIL)
        r.add_header('X-Auth-Key', TOKEN)
    else:
        r.add_header('Authorization', 'Bearer ' + TOKEN)
    r.add_header('Content-Type', 'application/json')
    try:
        with urllib.request.urlopen(r, timeout=40) as resp:
            return resp.getcode(), json.loads(resp.read().decode('utf-8', 'ignore'))
    except urllib.error.HTTPError as e:
        raw = e.read().decode('utf-8', 'ignore')
        try:    return e.code, json.loads(raw)
        except Exception: return e.code, {'raw': raw[:400]}
    except Exception as e:
        return 0, {'raw': str(e)}

def errs(j):
    e = j.get('errors') or []
    return '; '.join(str(x.get('message', x)) for x in e) or j.get('raw', '未知错误')

def _tunnel_names(acct):
    code, j = call('GET', '/accounts/%s/cfd_tunnel?is_deleted=false&per_page=100' % acct)
    if not j.get('success'): return None, errs(j)
    return set(t.get('name', '') for t in (j.get('result') or [])), None

def _dns_exists(zone, host):
    """该 CNAME 记录是否已被占用（无论指向谁）"""
    if not zone: return False
    import urllib.parse
    code, j = call('GET', '/zones/%s/dns_records?type=CNAME&name=%s' % (zone, urllib.parse.quote(host)))
    if not j.get('success'): return False
    return bool(j.get('result'))

def cmd_picknum(acct, prefix, skip_csv, zone, tpls):
    """挑一个真正空闲的编号：跳过手工指定已占用编号、已存在的同名隧道、DNS 已被占用的域名"""
    names, e = _tunnel_names(acct)
    if names is None: print("ERR %s" % e); return 1
    skip = set()
    for s in skip_csv.split(','):
        s = s.strip()
        if s.isdigit(): skip.add(int(s))
    for n in range(1, 201):
        if n in skip: continue
        if (prefix + str(n)) in names: continue
        busy = False
        for tpl in [x for x in tpls.split(',') if x.strip()]:
            if _dns_exists(zone, tpl.strip().replace('{N}', str(n))):
                busy = True; break
        if busy: continue
        print(n); return 0
    print("ERR 1~200 号全部被占用，请调整 CF_DOMAIN_TPL 或 CF_SKIP_NUMS"); return 1

def cmd_createtunnel(acct, name):
    code, j = call('POST', '/accounts/%s/cfd_tunnel' % acct,
                   {"name": name, "config_src": "cloudflare"})
    if not j.get('success'): print("ERR %s" % errs(j)); return 1
    r = j.get('result', {})
    if not r.get('token'): print("ERR 响应中没有 token 字段"); return 1
    print("%s %s" % (r.get('id'), r.get('token'))); return 0

def cmd_setconfig(acct, tid, hostnames, port):
    ingress = []
    for h in [x for x in hostnames.split(',') if x.strip()]:
        ingress.append({"hostname": h.strip(), "service": "http://127.0.0.1:%s" % port})
    ingress.append({"service": "http_status:404"})
    code, j = call('PUT', '/accounts/%s/cfd_tunnel/%s/configurations' % (acct, tid),
                   {"config": {"ingress": ingress, "originRequest": {"noTLSVerify": True}}})
    if not j.get('success'): print("ERR %s" % errs(j)); return 1
    print("OK"); return 0

def cmd_appendconfig(acct, tid, hostnames, port):
    """给已存在的隧道追加 ingress 规则，保留原有规则。

    复用别人/已有的隧道时用这个，绝不能像 cmd_setconfig 那样整体覆盖，
    否则会把隧道上已有的其它域名映射（别人的服务）一起冲掉。
    """
    code, j = call('GET', '/accounts/%s/cfd_tunnel/%s/configurations' % (acct, tid))
    if not j.get('success'):
        print("ERR 读取现有隧道配置失败: %s" % errs(j)); return 1
    cfg = (j.get('result') or {}).get('config') or {}
    ingress = [r for r in (cfg.get('ingress') or [])
               if r.get('service') != 'http_status:404' and not (r.get('service') or '').startswith('http_status:')]
    have = set(r.get('hostname') for r in ingress if r.get('hostname'))
    added, changed = [], []
    for h in [x for x in hostnames.split(',') if x.strip()]:
        h = h.strip()
        if h in have:
            for r in ingress:
                if r.get('hostname') == h:
                    r['service'] = 'http://127.0.0.1:%s' % port
                    changed.append(h)
            continue
        ingress.append({"hostname": h, "service": "http://127.0.0.1:%s" % port})
        added.append(h)
    ingress.append({"service": "http_status:404"})
    code, j = call('PUT', '/accounts/%s/cfd_tunnel/%s/configurations' % (acct, tid),
                   {"config": {"ingress": ingress,
                               "originRequest": cfg.get('originRequest') or {"noTLSVerify": True}}})
    if not j.get('success'):
        print("ERR %s" % errs(j)); return 1
    print("OK added=%s changed=%s total=%d" % (','.join(added) or '-', ','.join(changed) or '-', len(ingress)))
    return 0

def cmd_accounts():
    """列出该凭据能访问的所有账号，供脚本自动挑 Account ID。

    用 Global API Key 时用户几乎不可能记得 Account ID，必须自动探测。
    """
    code, j = call('GET', '/accounts')
    if not j.get('success'):
        print("ERR %s" % errs(j)); return 1
    for a in (j.get('result') or []):
        print("%s %s" % (a.get('id', ''), a.get('name', '')))
    return 0

def cmd_tunnelowned(acct, tid):
    """判断某个隧道 UUID 是否在本账号下 —— 决定「能不能覆盖它」。

    这是隧道归属的唯一可靠判据：域名后缀会变、机器上的记录可能丢，
    但账号下的隧道列表不会说谎。不属于本账号的隧道绝不能覆盖（那是别人的服务）。
    """
    code, j = call('GET', '/accounts/%s/cfd_tunnel?per_page=100' % acct)
    if not j.get('success'):
        print("ERR %s" % errs(j)); return 1
    for t in (j.get('result') or []):
        if (t.get('id') or '').lower() == (tid or '').lower():
            print("YES %s" % (t.get('name') or '')); return 0
    print("NO"); return 0

def cmd_deltunnel(acct, tid):
    """删除云端隧道（不可恢复）。只应在确认隧道属于本账号之后调用。

    有机器还连着时直接删会被拒绝，所以先踢掉活跃连接。
    """
    call('DELETE', '/accounts/%s/cfd_tunnel/%s/connections' % (acct, tid))
    code, j = call('DELETE', '/accounts/%s/cfd_tunnel/%s' % (acct, tid))
    if not j.get('success'):
        print("ERR %s" % errs(j)); return 1
    print("OK deleted %s" % tid); return 0

def cmd_deldns(zone, host):
    """按域名删除 CNAME 记录（可能多条），隧道删了之后的残留就是靠这个清。"""
    code, j = call('GET', '/zones/%s/dns_records?type=CNAME&name=%s' % (zone, host))
    if not j.get('success'):
        print("ERR %s" % errs(j)); return 1
    recs = j.get('result') or []
    if not recs:
        print("NONE"); return 0
    out = []
    for r in recs:
        c2, j2 = call('DELETE', '/zones/%s/dns_records/%s' % (zone, r.get('id')))
        out.append('%s:%s' % ((r.get('id') or '')[:8], 'OK' if j2.get('success') else 'ERR'))
    print("OK %s" % ','.join(out)); return 0

def cmd_findzone(host):
    parts = host.split('.')
    for i in range(len(parts) - 1):
        name = '.'.join(parts[i:])
        code, j = call('GET', '/zones?name=%s&status=active' % name)
        res = (j.get('result') or []) if j.get('success') else []
        if res: print("%s %s" % (res[0]['id'], name)); return 0
    print("ERR 找不到域名 %s 对应的 Zone（确认该域名托管在当前 CF 账号且 API Token 有 Zone:Read）" % host); return 1

def cmd_dns(zone, host, target):
    code, j = call('POST', '/zones/%s/dns_records' % zone,
        {"type": "CNAME", "name": host, "content": target, "proxied": True, "ttl": 1})
    if code == 409 or (not j.get('success') and 'already exists' in errs(j).lower()):
        print("EXIST"); return 0
    if not j.get('success'): print("ERR %s" % errs(j)); return 1
    print("OK"); return 0

if __name__ == '__main__':
    globals()['cmd_' + sys.argv[2]](*sys.argv[3:])
CFEOF
chmod +x "$HELPER_DIR/cf_api.py"
sed -i 's/\r$//' "$HELPER_DIR/cf_api.py"

# 把域名模板里的 {N} 换成实际机器编号
render_domains() {
  local n="$1" tpl out=() t
  if [ -z "$CF_DOMAIN_TPL" ]; then return 0; fi
  IFS=',' read -r -a _tpls <<< "$CF_DOMAIN_TPL"
  for tpl in "${_tpls[@]}"; do
    tpl="$(echo "$tpl" | tr -d ' ')"
    [ -n "$tpl" ] && out+=("${tpl//\{N\}/$n}")
  done
  DOMAIN_LIST=("${out[@]}")
}

# ---------- 读取本机上次安装记录 ----------
# 重跑脚本时复用已领到的编号/隧道/token，否则每跑一次就会新建一条垃圾隧道
SAVED_NUM=""; SAVED_TID=""; SAVED_TOKEN=""; SAVED_ZONE=""
if [ -f "$STATE_FILE" ]; then . "$STATE_FILE" 2>/dev/null || true; fi

# ---------- 需要 CF API 但没给凭据时，交互询问 ----------
# 复用模式：只有确实要追加域名才问；否则没给 TUNNEL_TOKEN 也没 cert.pem 才问
_NEED_API=0
if [ "${CF_REUSE_EXISTING:-0}" -eq 1 ]; then
  [ ${#DOMAIN_LIST[@]} -gt 0 ] && _NEED_API=1
elif [ -z "$CF_TUNNEL_TOKEN" ] && [ ! -f "${HOME:-/root}/.cloudflared/cert.pem" ]; then
  _NEED_API=1
fi
_CREDS_ASKED=0
if [ "$_NEED_API" -eq 1 ] && [ -z "$CF_API_TOKEN" ]; then
  ask_cf_creds && _CREDS_ASKED=1
fi

# Account ID 没给就自动探测：用 Global API Key 的人基本不可能记得住这串 ID
if [ -n "$CF_API_TOKEN" ] && [ -z "$CF_ACCOUNT_ID" ]; then
  _ACC="$(python3 "$HELPER_DIR/cf_api.py" "$CF_API_TOKEN" accounts 2>&1)"
  case "$_ACC" in
    ERR*) warn "自动获取账号列表失败: ${_ACC#ERR }";;
    *)
      _N_ACC="$(printf '%s\n' "$_ACC" | grep -c . || true)"
      if [ "${_N_ACC:-0}" -eq 1 ]; then
        CF_ACCOUNT_ID="$(printf '%s\n' "$_ACC" | awk '{print $1}')"
        ok "自动选定 Account: $_ACC"
      elif [ "${_N_ACC:-0}" -gt 1 ]; then
        echo "      该凭据下有 $_N_ACC 个账号："
        printf '%s\n' "$_ACC" | sed 's/^/        /'
        [ -t 0 ] && read -r -p "      请输入要用的 Account ID: " CF_ACCOUNT_ID
      fi;;
  esac
fi

# 本次交互输进来的凭据，问一句要不要落盘（下次同机重跑不用再输）
if [ "$_CREDS_ASKED" -eq 1 ]; then
  read -r -p "保存到 $CF_CONF 供下次使用？[y/N] " _save
  case "$_save" in
    y|Y)
      mkdir -p "$(dirname "$CF_CONF")"
      { echo "export CF_EMAIL='${CF_EMAIL}'"
        echo "export CF_API_TOKEN='${CF_API_TOKEN}'"
        echo "export CF_ACCOUNT_ID='${CF_ACCOUNT_ID}'"; } > "$CF_CONF"
      chmod 600 "$CF_CONF"
      ok "已保存到 $CF_CONF（权限 600，含账号凭据，别外传）";;
  esac
fi

# ---------- auto 模式：用 API 精确判断现有隧道的归属 ----------
# 比「域名后缀」「本机 SAVED_TID」都可靠：账号下的隧道列表是唯一不会说谎的判据。
# 不在本账号的隧道 = 别人的服务，绝不能覆盖（本账号的凭据也管不到它的域名映射）。
if [ "$CF_TUNNEL_MODE" = "auto" ] && [ -n "$CF_API_TOKEN" ] && [ -n "$CF_ACCOUNT_ID" ] && [ -n "$EXIST_TUNNEL_ID" ]; then
  _OWN="$(python3 "$HELPER_DIR/cf_api.py" "$CF_API_TOKEN" tunnelowned "$CF_ACCOUNT_ID" "$EXIST_TUNNEL_ID" 2>&1)"
  case "$_OWN" in
    YES*)
      TUNNEL_IS_OURS=1; CF_REUSE_EXISTING=0
      warn "auto：隧道 $EXIST_TUNNEL_ID（${_OWN#YES }）在本账号下 → 允许重建";;
    NO)
      TUNNEL_IS_OURS=0; CF_REUSE_EXISTING=1
      warn "auto：隧道 $EXIST_TUNNEL_ID 不在本账号（属于别的 CF 账号）→ 强制复用，绝不覆盖"
      warn "     本账号的凭据管不到它，要改域名映射只能去对应账号的后台手动操作";;
    *)
      warn "auto：归属查询失败，保守起见按复用处理：$_OWN"
      CF_REUSE_EXISTING=1;;
  esac
fi

# ---------- 模式 C：Cloudflare API 全自动（多台机器批量上线）----------
# 复用模式下绝不能走模式 C：它会新建隧道并整体覆盖 ingress，把现有服务直接打断
if [ -n "$CF_API_TOKEN" ] && [ -n "$CF_ACCOUNT_ID" ] && [ "${CF_REUSE_EXISTING:-0}" -ne 1 ]; then
  log "模式 C：调用 Cloudflare API 自动创建隧道"

  if [ -n "$SAVED_NUM" ] && [ -n "$SAVED_TID" ] && [ -n "$SAVED_TOKEN" ]; then
    # --- 复用上次已创建的隧道 ---
    MACHINE_NUM="$SAVED_NUM"; CF_TUNNEL_ID="$SAVED_TID"; CF_TUNNEL_TOKEN="$SAVED_TOKEN"
    TUNNEL_NAME="${TUNNEL_PREFIX}${MACHINE_NUM}"
    [ -n "$SAVED_ZONE" ] && CF_ZONE_ID="$SAVED_ZONE"
    [ -z "$CF_DOMAIN_TPL" ] || render_domains "$MACHINE_NUM"
    # zone 缺失时仍需探测，否则 DNS 接口会调空 zone
    if [ -z "$CF_ZONE_ID" ]; then
      ZONE_HOST="${CF_DOMAIN_TPL%%,*}"; [ -n "$ZONE_HOST" ] || ZONE_HOST="${DOMAIN_LIST[0]:-}"
      ZONE_RES="$(python3 "$HELPER_DIR/cf_api.py" "$CF_API_TOKEN" findzone "$ZONE_HOST")"
      case "$ZONE_RES" in ERR*) die "$ZONE_RES";; esac
      CF_ZONE_ID="${ZONE_RES%% *}"
    fi
    ok "复用本机上次安装记录：编号 $MACHINE_NUM / 隧道 $TUNNEL_NAME，不重复创建隧道"
  else
  # 先探测 Zone（用模板/域名本身，逐级向上找，含 {N} 也能定位到主域名）
  if [ -z "$CF_ZONE_ID" ]; then
    ZONE_HOST="${CF_DOMAIN_TPL%%,*}"
    [ -n "$ZONE_HOST" ] || ZONE_HOST="${DOMAIN_LIST[0]:-}"
    ZONE_RES="$(python3 "$HELPER_DIR/cf_api.py" "$CF_API_TOKEN" findzone "$ZONE_HOST")"
    case "$ZONE_RES" in ERR*) die "$ZONE_RES";; esac
    CF_ZONE_ID="${ZONE_RES%% *}"
    ok "Zone: ${ZONE_RES#* } (${CF_ZONE_ID})"
  fi

  # 挑一个真正空闲的编号：跳过 CF_SKIP_NUMS、已存在的同名隧道、DNS 已被占用的域名
  if [ -n "$CF_MACHINE_NUM" ]; then
    MACHINE_NUM="$CF_MACHINE_NUM"
    ok "本机编号（手动指定）: $MACHINE_NUM"
  else
    MACHINE_NUM="$(python3 "$HELPER_DIR/cf_api.py" "$CF_API_TOKEN" picknum \
                    "$CF_ACCOUNT_ID" "$TUNNEL_PREFIX" "$CF_SKIP_NUMS" "$CF_ZONE_ID" "$CF_DOMAIN_TPL")"
    case "$MACHINE_NUM" in
      ERR*) die "分配机器编号失败: $MACHINE_NUM";;
      '')   die "分配机器编号失败: 返回为空";;
    esac
    ok "本机编号（自动分配）: $MACHINE_NUM（已跳过占用编号: ${CF_SKIP_NUMS:-无}）"
  fi

  CREATED=""
  for attempt in $(seq 1 5); do
    TUNNEL_NAME="${TUNNEL_PREFIX}${MACHINE_NUM}"
    RES="$(python3 "$HELPER_DIR/cf_api.py" "$CF_API_TOKEN" createtunnel "$CF_ACCOUNT_ID" "$TUNNEL_NAME")"
    case "$RES" in
      ERR*) warn "创建 $TUNNEL_NAME 失败: $RES （尝试下一个编号）"; MACHINE_NUM=$((MACHINE_NUM+1));;
      *)    CREATED="$RES"; break;;
    esac
  done
  [ -n "$CREATED" ] || die "创建隧道失败（已重试 5 次），检查 CF_API_TOKEN 权限是否含 Account:Cloudflare Tunnel:Edit"
  CF_TUNNEL_ID="${CREATED%% *}"
  CF_TUNNEL_TOKEN="${CREATED#* }"
  ok "隧道已创建: $TUNNEL_NAME (ID: $CF_TUNNEL_ID)"

  # 记录本次分配结果，供重跑时复用（含 token，权限 600）
  umask 077
  cat > "$STATE_FILE" <<STEOF
SAVED_NUM=$MACHINE_NUM
SAVED_TID=$CF_TUNNEL_ID
SAVED_TOKEN=$CF_TUNNEL_TOKEN
SAVED_ZONE=$CF_ZONE_ID
STEOF
  chmod 600 "$STATE_FILE"
  umask 022
  fi

  # 编号最终确定后再渲染域名（重试换号时域名要跟着变）
  [ -z "$CF_DOMAIN_TPL" ] || render_domains "$MACHINE_NUM"
  [ "${#DOMAIN_LIST[@]}" -gt 0 ] || die "没有可用域名，请设置 CF_DOMAIN_TPL 或 CF_DOMAINS"
  ok "本机域名: ${DOMAIN_LIST[*]}"

  CFG_OUT="$(python3 "$HELPER_DIR/cf_api.py" "$CF_API_TOKEN" setconfig "$CF_ACCOUNT_ID" "$CF_TUNNEL_ID" "$(IFS=','; echo "${DOMAIN_LIST[*]}")" "$PANEL_PORT")"
  case "$CFG_OUT" in OK*) ok "隧道 ingress 已配置（${#DOMAIN_LIST[@]} 个域名 → :$PANEL_PORT）";; *) die "配置 ingress 失败: $CFG_OUT";; esac

  for d in "${DOMAIN_LIST[@]}"; do
    DNS_OUT="$(python3 "$HELPER_DIR/cf_api.py" "$CF_API_TOKEN" dns "$CF_ZONE_ID" "$d" "${CF_TUNNEL_ID}.cfargotunnel.com")"
    case "$DNS_OUT" in
      OK)    ok "DNS CNAME 已添加: $d";;
      EXIST) warn "DNS 记录已存在: $d（沿用）";;
      *)     die "添加 DNS 记录 $d 失败: $DNS_OUT";;
    esac
  done
  log "隧道 $TUNNEL_NAME 配置完成，下面将 token 装成本机服务"
fi

# 安装隧道服务前先卸载旧服务：否则重复安装会 unit 冲突，或残留旧配置覆盖新隧道
# 但复用模式下绝不能停止/卸载——那会把机器上正在提供服务的隧道直接打断
if [ "${CF_REUSE_EXISTING:-0}" -eq 1 ]; then
  log "复用模式：跳过 stop / pkill / service uninstall，现有隧道保持运行"
else
  systemctl stop cloudflared 2>/dev/null || true
  pkill -f 'cloudflared.*tunnel run' 2>/dev/null || true
  cloudflared service uninstall >/dev/null 2>&1 || true
fi

# 无 systemd 环境（容器）用的隧道守护脚本
cat > /usr/local/bin/cloudflared-run.sh <<CFRUN
#!/usr/bin/env bash
# cloudflared 存活守护（由一键安装脚本生成，无 systemd 环境使用）
LOG=/var/log/cloudflared.log
if [ -d /run/systemd/system ] && command -v systemctl >/dev/null 2>&1 \\
   && systemctl list-unit-files 2>/dev/null | grep -q '^cloudflared'; then
  systemctl is-active --quiet cloudflared || systemctl restart cloudflared
  exit 0
fi
pgrep -f 'cloudflared.*tunnel run' >/dev/null 2>&1 && exit 0
CFRUN
if [ -n "$CF_TUNNEL_TOKEN" ]; then
  echo "nohup cloudflared tunnel run --token '$CF_TUNNEL_TOKEN' >> \"\$LOG\" 2>&1 &" \
    >> /usr/local/bin/cloudflared-run.sh
else
  echo "nohup cloudflared --config /etc/cloudflared/config.yml tunnel run >> \"\$LOG\" 2>&1 &" \
    >> /usr/local/bin/cloudflared-run.sh
fi
chmod 700 /usr/local/bin/cloudflared-run.sh

if [ "${CF_REUSE_EXISTING:-0}" -eq 1 ]; then
  # ---------- 复用现有隧道：不做任何破坏性动作 ----------
  log "复用现有隧道：跳过 tunnel create / ingress 重写 / route dns / service install"
  MISSING_DOMAINS=()
  for _d in "${DOMAIN_LIST[@]}"; do
    _found=0
    for _e in "${EXIST_DOMAINS[@]}"; do [ "$_d" = "$_e" ] && _found=1; done
    [ "$_found" -eq 0 ] && MISSING_DOMAINS+=("$_d")
  done
  if [ ${#MISSING_DOMAINS[@]} -eq 0 ]; then
    ok "本次域名已全部在现有隧道映射中: ${DOMAIN_LIST[*]}"
  elif [ -n "$CF_API_TOKEN" ] && [ -n "$CF_ACCOUNT_ID" ] && [ -n "$EXIST_TUNNEL_ID" ]; then
    # 有 API 凭据 → 调 CF API 给现有隧道追加规则，原有映射一条不动
    log "检测到 CF API 凭据 → 自动给现有隧道 $EXIST_TUNNEL_ID 追加域名（保留原有规则）"
    _AP_OUT="$(python3 "$HELPER_DIR/cf_api.py" "$CF_API_TOKEN" appendconfig \
                "$CF_ACCOUNT_ID" "$EXIST_TUNNEL_ID" \
                "$(IFS=','; echo "${MISSING_DOMAINS[*]}")" "$PANEL_PORT" 2>&1)"
    case "$_AP_OUT" in
      OK*)
        ok "ingress 已追加: ${_AP_OUT#OK }"
        for _d in "${MISSING_DOMAINS[@]}"; do
          _Z_OUT="$(python3 "$HELPER_DIR/cf_api.py" "$CF_API_TOKEN" findzone "$_d" 2>&1 | head -n1)"
          case "$_Z_OUT" in
            ERR*) warn "跳过 $_d 的 DNS 记录：${_Z_OUT#ERR }"; continue;;
          esac
          _ZONE="${_Z_OUT%% *}"
          _D_OUT="$(python3 "$HELPER_DIR/cf_api.py" "$CF_API_TOKEN" dns \
                     "$_ZONE" "$_d" "${EXIST_TUNNEL_ID}.cfargotunnel.com" 2>&1)"
          case "$_D_OUT" in
            OK|EXIST) ok "DNS 就绪: $_d → ${EXIST_TUNNEL_ID}.cfargotunnel.com";;
            *) warn "DNS 写入失败 $_d: $_D_OUT";;
          esac
        done
        log "云端配置已更新，等待隧道拉取新配置（约 10~30 秒）"; sleep 15
        ;;
      *) warn "自动追加 ingress 失败: $_AP_OUT";;
    esac
  else
    warn "本次指定的域名不在现有隧道映射中，需到 Cloudflare 后台补 Public Hostname："
    echo "      Zero Trust → Networks → Tunnels → 现有隧道 → Published application routes"
    for _d in "${MISSING_DOMAINS[@]}"; do
      echo "      · $_d  →  http://127.0.0.1:$PANEL_PORT"
    done
    echo "      （补上 CF_API_TOKEN + CF_ACCOUNT_ID 后本脚本可自动追加，且不动已有映射）"
  fi
elif [ -n "$CF_TUNNEL_TOKEN" ]; then
  # ---------- 模式 A：Zero Trust 远程托管隧道（token / JWT）----------
  log "使用远程托管模式安装隧道服务"
  if [ "$HAS_SYSTEMD" -eq 1 ]; then
    cloudflared service install "$CF_TUNNEL_TOKEN" >/dev/null 2>&1 \
      || die "cloudflared service install <token> 失败，请检查 token 是否正确"
    systemctl enable --now cloudflared >/dev/null 2>&1 \
      || die "cloudflared 服务启动失败，查看 journalctl -u cloudflared"
    ok "隧道服务已安装并开机自启（远程托管模式）"
  else
    warn "无 systemd，隧道改用后台进程 + cron 守护"
    /usr/local/bin/cloudflared-run.sh
    ( crontab -l 2>/dev/null | grep -v 'cloudflared-run.sh'; \
      echo "@reboot sleep 15 && /usr/local/bin/cloudflared-run.sh"; \
      echo "*/5 * * * * /usr/local/bin/cloudflared-run.sh" ) | crontab -
    ok "隧道已后台启动并加入 cron 守护"
  fi
  if [ "${#DOMAIN_LIST[@]}" -gt 1 ] && [ -z "$CF_API_TOKEN" ]; then
    echo
    warn "远程托管模式下，多域名需要在 Cloudflare 后台手动补 Public Hostname："
    echo "      Zero Trust → Networks → Tunnels → 你的隧道 → Published application routes"
    for d in "${DOMAIN_LIST[@]}"; do
      echo "      · $d  →  http://127.0.0.1:$PANEL_PORT"
    done
    echo "      (第一个域名通常建隧道时已自动添加，其余按上面格式逐条加)"
  fi
else
  # ---------- 模式 B：本地命名隧道 yb-dd1 / yb-dd2 ... ----------
  if [ ! -f /root/.cloudflared/cert.pem ] && [ ! -f "$HOME/.cloudflared/cert.pem" ]; then
    echo
    err "未找到 Cloudflare 授权证书 (cert.pem)，无法自动创建命名隧道"
    cat <<TIP
  请在本地浏览器完成一次授权（任选其一）：
    方式1（推荐，全自动）：在 Cloudflare Zero Trust 后台创建隧道，复制它的 token，然后：
        CF_TUNNEL_TOKEN=<token> CF_DOMAINS=$CF_DOMAINS sudo -E bash $0
    方式2（命名隧道）：在本机执行
        cloudflared tunnel login
    会输出一个 URL，用浏览器打开并选择域名授权，证书会落到 ~/.cloudflared/cert.pem
    授权完成后重新执行：
        CF_DOMAINS=$CF_DOMAINS bash $0
TIP
    die "缺少 Cloudflare 授权证书 (cert.pem)"
  fi

  # 编号：手动指定 > 复用上次 > 现有隧道最大编号 +1
  if [ -n "$CF_MACHINE_NUM" ]; then
    NEXT="$CF_MACHINE_NUM"
    ok "本机编号（手动指定）: $NEXT"
  elif [ -n "$SAVED_NUM" ] \
     && cloudflared tunnel list 2>/dev/null | grep -qE "[[:space:]]${TUNNEL_PREFIX}${SAVED_NUM}([[:space:]]|$)"; then
    NEXT="$SAVED_NUM"
    ok "复用本机上次编号: $NEXT"
  else
    LAST="$(cloudflared tunnel list 2>/dev/null | grep -oE "${TUNNEL_PREFIX}[0-9]+" | grep -oE '[0-9]+' | sort -n | tail -n1)"
    NEXT=$(( ${LAST:-0} + 1 ))
    # 跳过已占用的编号（如 dd3 已在使用）
    _skip=",${CF_SKIP_NUMS},"
    while [[ "$_skip" == *",$NEXT,"* ]]; do NEXT=$((NEXT+1)); done
  fi
  TUNNEL_NAME="${TUNNEL_PREFIX}${NEXT}"
  render_domains "$NEXT"
  log "即将创建隧道: $TUNNEL_NAME  域名: ${DOMAIN_LIST[*]:-未指定}"

  if cloudflared tunnel list 2>/dev/null | grep -q "[[:space:]]$TUNNEL_NAME[[:space:]]*$"; then
    warn "隧道 $TUNNEL_NAME 已存在，复用"
  else
    cloudflared tunnel create "$TUNNEL_NAME" >/tmp/dd-install/tunnel_create.log 2>&1 \
      || { cat /tmp/dd-install/tunnel_create.log; die "cloudflared tunnel create $TUNNEL_NAME 失败"; }
  fi
  ok "隧道 $TUNNEL_NAME 已就绪"

  TUNNEL_ID="$(cloudflared tunnel list 2>/dev/null | awk -v n="$TUNNEL_NAME" '$2==n || $NF==n {print $1; exit}')"
  [ -n "$TUNNEL_ID" ] || TUNNEL_ID="$(cloudflared tunnel info "$TUNNEL_NAME" 2>/dev/null | grep -oE '[0-9a-f-]{36}' | head -n1)"
  [ -n "$TUNNEL_ID" ] || die "无法解析隧道 $TUNNEL_NAME 的 UUID"
  ok "隧道 UUID: $TUNNEL_ID"

  CRED_FILE="$(find /root/.cloudflared "$HOME/.cloudflared" /etc/cloudflared -name "${TUNNEL_ID}.json" 2>/dev/null | head -n1)"
  [ -n "$CRED_FILE" ] || die "找不到隧道凭据文件 ${TUNNEL_ID}.json"
  cp -f "$CRED_FILE" "/etc/cloudflared/${TUNNEL_ID}.json"

  # 多域名用 ingress 规则，每个 hostname 都指向本机面板
  {
    echo "tunnel: $TUNNEL_ID"
    echo "credentials-file: /etc/cloudflared/${TUNNEL_ID}.json"
    echo "originRequest:"
    echo "  noTLSVerify: true"
    echo "ingress:"
    for d in "${DOMAIN_LIST[@]}"; do
      echo "  - hostname: $d"
      echo "    service: http://127.0.0.1:$PANEL_PORT"
    done
    echo "  - service: http_status:404"
  } > /etc/cloudflared/config.yml
  ok "已写入 /etc/cloudflared/config.yml（${#DOMAIN_LIST[@]} 条 ingress 规则 → 本机 :$PANEL_PORT）"

  for d in "${DOMAIN_LIST[@]}"; do
    cloudflared tunnel route dns "$TUNNEL_NAME" "$d" >/tmp/dd-install/route_dns.log 2>&1 \
      || { cat /tmp/dd-install/route_dns.log; die "DNS 路由 $d 创建失败（域名可能不在当前 Cloudflare 账号下）"; }
    ok "已自动添加 CNAME: $d -> ${TUNNEL_ID}.cfargotunnel.com"
  done

  # 凭据与配置含密钥，收紧权限
  chmod 600 /etc/cloudflared/config.yml /etc/cloudflared/*.json 2>/dev/null || true

  # 记录本机编号，重跑时复用（模式 B 不保存 token）
  umask 077
  cat > "$STATE_FILE" <<STEOF
SAVED_NUM=$NEXT
SAVED_TID=$TUNNEL_ID
SAVED_TOKEN=
SAVED_ZONE=
STEOF
  chmod 600 "$STATE_FILE"
  umask 022

  if [ "$HAS_SYSTEMD" -eq 1 ]; then
    cloudflared service install >/dev/null 2>&1 || warn "service install 返回非 0，尝试手动启用服务"
    systemctl enable --now cloudflared >/dev/null 2>&1 \
      || die "cloudflared 服务启动失败，查看 journalctl -u cloudflared"
    ok "隧道服务已安装并开机自启（systemd: cloudflared）"
  else
    warn "无 systemd，隧道改用后台进程 + cron 守护"
    /usr/local/bin/cloudflared-run.sh
    ( crontab -l 2>/dev/null | grep -v 'cloudflared-run.sh'; \
      echo "@reboot sleep 15 && /usr/local/bin/cloudflared-run.sh"; \
      echo "*/5 * * * * /usr/local/bin/cloudflared-run.sh" ) | crontab -
    ok "隧道已后台启动并加入 cron 守护"
  fi
fi
}

# ==============================================================================
run_step7() {
step 7 "检测 ${#DOMAIN_LIST[@]} 个公网域名是否可访问"
# ==============================================================================
if [ ${#DOMAIN_LIST[@]} -eq 0 ]; then
  # 面板和保活都装完了，不该因为没有公网域名就把整脚本判成失败退出
  warn "没有可映射的域名（未指定 CF_DOMAINS，且现有隧道上没有指向面板端口的域名）"
  warn "面板本机可用: http://127.0.0.1:$PANEL_PORT"
  warn "需要公网访问时：给隧道加 Public Hostname → http://127.0.0.1:$PANEL_PORT，或用 CF_DOMAINS=xxx 重跑"
  return 0
fi

# 先确认隧道本身是否已连上 Cloudflare 边缘（隧道没连上时域名必然打不开）
log "等待隧道与 Cloudflare 边缘建立连接（最长 90 秒）..."
TUNNEL_UP=0
for i in $(seq 1 18); do
  if [ "$HAS_SYSTEMD" -eq 1 ] \
     && journalctl -u cloudflared --since '3 min ago' --no-pager 2>/dev/null \
        | grep -qi 'Registered tunnel connection'; then
    TUNNEL_UP=1; break
  fi
  grep -qi 'Registered tunnel connection' /var/log/cloudflared.log 2>/dev/null && { TUNNEL_UP=1; break; }
  sleep 5
done
[ "$TUNNEL_UP" -eq 1 ] && ok "隧道已连上 Cloudflare 边缘" \
  || warn "未能从日志确认隧道已连上边缘，继续检测域名；若打不开请看 journalctl -u cloudflared"

log "等待 DNS 与隧道生效（每个域名最长 180 秒）..."
DOMAIN_OK=1
FAILED_DOMAINS=()
for d in "${DOMAIN_LIST[@]}"; do
  THIS_OK=0; HTTP_CODE="无响应"
  for i in $(seq 1 60); do
    HTTP_CODE="$(curl -s -o /dev/null -w '%{http_code}' --max-time 10 --noproxy '*' -L "https://$d/" 2>/dev/null)"
    case "$HTTP_CODE" in
      200|201|204|301|302|303|307|308|403|405) THIS_OK=1; break;;
    esac
    sleep 3
  done
  if [ "$THIS_OK" -eq 1 ]; then
    ok "https://$d 可访问 (HTTP $HTTP_CODE)"
  else
    # 解析 Cloudflare 错误页，给出准确原因而不是干巴巴一个状态码
    CF_ERR="$(curl -s --max-time 10 --noproxy '*' -L "https://$d/" 2>/dev/null \
              | grep -oiE 'error code[^0-9]*[0-9]+' | grep -oE '[0-9]+' | head -n1)"
    case "$CF_ERR" in
      1033) WHY="隧道未连通(1033)：cloudflared 没跑起来或 token/配置不对";;
      1034) WHY="隧道找不到该域名(1034)：域名未加入隧道的 Public Hostname";;
      522)  WHY="源站无响应(522)：本机面板未监听 $PANEL_PORT";;
      1020) WHY="被 Cloudflare 防火墙规则拒绝(1020)";;
      525|526) WHY="SSL 握手失败($CF_ERR)";;
      *)    WHY="HTTP ${HTTP_CODE:-无响应}${CF_ERR:+ / CF错误码 $CF_ERR}";;
    esac
    err "https://$d 无法访问 —— $WHY"
    FAILED_DOMAINS+=("$d($WHY)")
    DOMAIN_OK=0
  fi
done

if [ "$DOMAIN_OK" -eq 1 ]; then
  # 把结果落到文件，避免刷屏后找不到账号密码
  {
    echo "元宝派一键安装结果  $(date '+%F %T')"
    for d in "${DOMAIN_LIST[@]}"; do echo "面板公网地址 : https://$d"; done
    echo "面板账号密码 : $PANEL_USER / $PANEL_PASS"
    echo "面板内网地址 : http://127.0.0.1:$PANEL_PORT"
    echo "保活脚本     : $SCRIPT_DIR/$KEEPALIVE_SCRIPT"
    echo "计划任务     : $TASK_NAME (cron: $TASK_CRON，每天 1 次)"
    [ -n "${TUNNEL_NAME:-}" ] && echo "隧道名称     : $TUNNEL_NAME（本机编号 ${MACHINE_NUM:-${NEXT:-}}）"
    echo "安装日志     : $LOG_FILE"
  } > "$SUMMARY_FILE"
  chmod 600 "$SUMMARY_FILE"
  echo
  echo -e "${C_G}=========================================================${C_0}"
  echo -e "${C_G}  全部完成！${C_0}"
  for d in "${DOMAIN_LIST[@]}"; do
    echo -e "  面板公网地址 : ${C_Y}https://$d${C_0}"
  done
  echo -e "  面板账号密码 : ${C_Y}$PANEL_USER / $PANEL_PASS${C_0}"
  echo -e "  面板内网地址 : http://127.0.0.1:$PANEL_PORT"
  echo -e "  保活脚本     : $SCRIPT_DIR/$KEEPALIVE_SCRIPT"
  echo -e "  计划任务     : $TASK_NAME (cron: $TASK_CRON，每天 1 次)"
  [ -n "$CF_API_TOKEN" ] && echo -e "  隧道名称     : ${TUNNEL_NAME:-}（本机编号 ${MACHINE_NUM:-}）"
  echo -e "${C_G}=========================================================${C_0}"
  exit 0
else
  BAD="$(printf '%s, ' "${FAILED_DOMAINS[@]}")"
  echo
  err "现场诊断："
  for f in "${FAILED_DOMAINS[@]}"; do
    _h="${f%%(*}"
    _ip="$(getent hosts "$_h" 2>/dev/null | head -n1 | awk '{print $1}')"
    echo "    · $_h  →  ${_ip:-未解析}  （期望是 Cloudflare 边缘 IP，且 CNAME 指向 *.cfargotunnel.com）"
  done
  echo "    · cloudflared 状态: $(systemctl is-active cloudflared 2>/dev/null || echo '未知/非 systemd 环境')"
  echo "    · 本机面板 HTTP 码: $(curl -s -o /dev/null -w '%{http_code}' --max-time 3 --noproxy '*' "http://127.0.0.1:$PANEL_PORT/api/auth/check-init" 2>/dev/null || echo 无响应)"
  echo "    · 隧道日志: journalctl -u cloudflared -n 50 --no-pager"
  die "以下域名无法访问: ${BAD%, }
   可能原因：
     1) DNS 未生效 -> dig +short <域名>  应返回 *.cfargotunnel.com
     2) cloudflared 服务未正常运行 -> systemctl status cloudflared
     3) 域名未托管在当前 Cloudflare 账号 -> 到 CF 后台确认
     4) 多域名时，远程托管模式需到后台 Public Hostname 补齐条目
     5) 面板未监听 -> curl http://127.0.0.1:$PANEL_PORT/api/auth/check-init"
fi

}

# ==============================================================================
# 主流程：按选择的步骤依次执行
# ==============================================================================
for _s in 1 2 3 4 5 6 7; do
  if should_run "$_s"; then
    "run_step$_s"
  else
    log "跳过步骤 $_s（本次未选择）"
  fi
done
```

---

## 附录 B：参数表

# 参数速查（scripts/yuanbao_install.sh）

所有变量都可以用环境变量覆盖，不传就用默认值。非 root 执行时记得 `sudo -E` 保留环境。

## 运行模式

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `CF_SURVEY_ONLY=1` | 0 | 只做隧道/域名体检就退出，不装任何东西（只读） |
| `UNINSTALL=1` | 0 | 走卸载分支 |
| `STEPS=1,3,4` | all | 只执行指定步骤（自选安装，1–7） |
| `PANEL_PORT=5701` | 5700 | 面板端口，隧道 ingress 会指向它 |

## Cloudflare 凭据与隧道

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `CF_API_TOKEN` | 空 | API Token（Bearer）**或** Global API Key（配 `CF_EMAIL` 时） |
| `CF_EMAIL` | 空 | 仅 Global API Key 需要；用 API Token 时必须留空 |
| `CF_ACCOUNT_ID` | 自动探测 | 32 位 hex；不填会用 `/accounts` 探测，单账号直接选定 |
| `CF_ZONE_ID` | 自动探测 | 不填时按域名找 zone |
| `CF_TUNNEL_TOKEN` | 空 | 直接用现成的远程托管 token（跳过建隧道） |
| `CF_TUNNEL_MODE` | `reuse` | `reuse` 有可用隧道就复用（默认，最安全）/ `auto` 按 UUID 归属判断能否重建 / `new` 无条件重建 |
| `CF_DOMAINS=yb1.a.com,yb2.a.com` | 空 | 面板域名列表，逗号分隔 |
| `CF_DOMAIN_TPL='yb{N}.a.com'` | 空 | 域名模板，`{N}` 换成机器编号 |
| `CF_MACHINE_NUM=7` | 自动 | 手动指定机器编号（模板渲染用） |
| `CF_SKIP_NUMS=3,7` | 3 | 已占用的编号，自动跳过 |
| `TUNNEL_PREFIX=yb-dd` | `yb-dd` | 隧道名前缀（模式 B/C 新建时用） |

## 卸载

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `UNINSTALL_SCOPE` | `cf` | `cf` 只停本机 cloudflared / `panel` 停面板 / `keepalive` 删保活 / `all` |
| `UNINSTALL_PURGE=1` | 0 | 连云端隧道 + DNS 一起删（不可恢复，非本账号隧道会跳过） |
| `UNINSTALL_YES=1` | 0 | 跳过交互确认；非交互环境必填，否则直接中止 |
| `PANEL_TOKEN` | 空 | scope 含 `keepalive` 时用它调面板 API 删定时任务 |

## 其它

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `LOCAL_TARBALL=/path/x.tar.gz` | 空 | 用本地离线包，跳过下载 |
| `CF_CONF` | `/etc/yuanbao/cf.conf` | 凭据落盘位置（权限 600） |

## 脚本自带的 CF API 子命令

调试时可单独调用（token 走第一个参数，邮箱走 `CF_EMAIL` 环境变量）：

```
python3 /opt/yuanbao-install/cf_api.py "$CF_API_TOKEN" <子命令> [参数]

accounts                          列出凭据可见的账号（id + 名字）
tunnelowned <acct> <uuid>         判断隧道是否在本账号 → YES <名字> / NO
findzone <host>                   按域名找 zone（id + 名字）
dns <zone> <host> <target>        写/更新 CNAME
deltunnel <acct> <uuid>           删云端隧道（先踢连接再删，不可恢复）
deldns <zone> <host>              删该域名的 CNAME 记录
createtunnel <acct> <name>        建隧道
setconfig <acct> <uuid> <hosts> <port>    整体覆盖 ingress（会冲掉已有规则！）
appendconfig <acct> <uuid> <hosts> <port> 追加 ingress（保留原有规则，复用时用它）
picknum <acct> <prefix> <skip> <zone> <tpls>   挑一个没占用的机器编号
```

`appendconfig` 与 `setconfig` 的区别是复用场景的关键：**前者保留原规则，后者整表覆盖**。改现有隧道一律用前者。

## 落盘路径

| 路径 | 内容 |
| --- | --- |
| `/opt/yuanbao-install/` | 辅助脚本 `cf_api.py` / `panel_api.py`、安装记录 `.install_state`、卸载备份目录 |
| `/var/log/yuanbao_install.log` | 完整安装日志（排障第一现场） |
| `/etc/yuanbao/cf.conf` | 可选凭据文件（600） |
| `/opt/daidai-panel/` | 面板安装目录，`data/scripts/` 下放保活脚本 |
| `/etc/cloudflared/` | cloudflared 配置与凭据（卸载时会备份） |

---

## 附录 C：排错手册

# 隧道 / 域名 排错手册

排错顺序：**先拿现场**（日志 / 进程 / 配置）→ 再定位 → 最后动手。别一上来就重启，那会把现场清掉。

## 0. 一次性拿全现场

```bash
echo "=== cloudflared ==="
command -v cloudflared >/dev/null && cloudflared --version 2>/dev/null | head -1 || echo "未安装"
echo "systemd: $(systemctl is-active cloudflared 2>/dev/null || echo '无/未激活')"
pgrep -af cloudflared 2>/dev/null | head -3 || echo "(无进程)"

echo "=== 运行模式（token 已隐藏）==="
pid=$(pgrep -f cloudflared | head -1)
[ -n "$pid" ] && tr '\0' ' ' < /proc/$pid/cmdline | sed 's/--token [^ ]*/--token <隐藏>/' | head -c 300; echo
systemctl cat cloudflared 2>/dev/null | grep -m1 ExecStart | sed 's/--token [^ ]*/--token <隐藏>/'

echo "=== 边缘连接 ==="
journalctl -u cloudflared --since '10 min ago' --no-pager 2>/dev/null \
  | grep -iE 'Registered tunnel connection|error|ERR' | tail -5 || echo "(无 systemd 日志)"

echo "=== ingress（本地配置，token 模式下可能不生效）==="
CFG=""; for f in /etc/cloudflared/config.yml /etc/cloudflared/config.yaml; do [ -f "$f" ] && CFG="$f" && break; done
if [ -n "$CFG" ]; then
  echo "配置: $CFG"
  grep -E '^[[:space:]]*(tunnel|credentials-file):' "$CFG"
  awk '/hostname:[[:space:]]*[^[:space:]]+/{h=$NF} /service:[[:space:]]*[^[:space:]]+/{if(h!=""){print "  "h" -> "$NF; h=""}}' "$CFG"
else
  echo "(无本地 config.yml → 应为远程托管 token 模式)"
fi

echo "=== 域名解析与访问 ==="
for d in $(grep -hoE 'hostname:[[:space:]]*[^[:space:]]+' /etc/cloudflared/config.y*ml 2>/dev/null \
           | sed -E 's/.*:[[:space:]]*//' | sort -u); do
  ip=$(python3 -c 'import socket,sys
try: print(socket.gethostbyname(sys.argv[1]))
except Exception: print("未解析")' "$d" 2>/dev/null)
  code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 10 --noproxy '*' -L "https://$d/")
  echo "  $d -> $ip | HTTPS $code"
done
```

## 1. 隧道归属：这台机器上的隧道到底是谁的

**这是决定「能不能动它」的前提，必须先查清。**

```bash
# 从运行中的 token 解出隧道 UUID（token 模式最可靠，本地 config.yml 可能压根没被加载）
pid=$(pgrep -f cloudflared | head -1)
tk=$(tr '\0' ' ' < /proc/$pid/cmdline 2>/dev/null | grep -oE -- '--token [^ ]+' | awk '{print $NF}')
[ -n "$tk" ] && printf '%s' "$tk" | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null \
  | grep -oE '[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}'

# 用 API 判断它是否在本账号
python3 /opt/yuanbao-install/cf_api.py "$CF_API_TOKEN" tunnelowned "$CF_ACCOUNT_ID" "<上面的UUID>"
# YES <隧道名> = 本账号，可覆盖；NO = 别人的，只能复用/追加
```

判断能不能覆盖的**三条可靠判据**（按优先级）：

1. API 查 UUID 是否在本账号隧道列表 —— 最可靠
2. 本机安装记录 `/opt/yuanbao-install/.install_state` 里的 `SAVED_TID` 与当前 UUID 相同 —— 说明是本脚本建的
3. 有本机 `cert.pem`（本地命名隧道）且能 `cloudflared tunnel list` 查到同名隧道

**不可靠判据（不要用）**：域名后缀、机器上有没有配置文件、端口占用。

## 2. 常见故障表

| 症状 | 原因 | 处理 |
| --- | --- | --- |
| `HTTPS 000` | DNS 没解析 / 隧道没连上边缘 / 出网被拦 | 先看 DNS 是否解析到 CF IP；再看 journalctl 有没有 `Registered tunnel connection` |
| `HTTPS 200` 但打开不是面板 | 域名指向了别的本机端口（如 20447） | 改 ingress：`appendconfig` 把该域名指到面板端口；别拿这个域名当验收依据 |
| `502 Bad gateway` | 隧道通了但源站没响应 | 面板没起 / 端口不对：`systemctl is-active daidai-panel`、`PANEL_PORT` 核对 |
| 隧道 active 但域名不通 | DNS CNAME 缺失或指向旧隧道 | 查 CNAME 是否指向 `<tunnel-id>.cfargotunnel.com` |
| DNS 指向的隧道 UUID 在账号里查不到 | 隧道已删，DNS 是残留记录（僵尸记录） | 确认不可达后可删：`deldns` 或后台删 |
| 隧道反复重连失败 | token 被吊销 / 云端隧道被删 | 去后台确认隧道还在；不在就重新建或换 token |
| 本地改了 config.yml 不生效 | 远程托管（token）模式，云端配置为准 | 改 Cloudflare 后台的 Public Hostname，或用 `appendconfig` 调 API 改 |
| 重装后多出垃圾隧道 | 每次跑都新建 | 用 `reuse`/`auto`；安装记录在 `/opt/yuanbao-install/.install_state` |
| `service install` 报 unit 冲突 | 旧服务没卸干净 | 复用模式不会卸；要重建就先确认归属，再 `cloudflared service uninstall` |

## 3. 一台机器跑多个隧道（不想打断现有服务时）

systemd 一个 unit 名只能装一个 cloudflared 服务。要在不打断现有隧道的前提下再挂一个，需要给新隧道用**不同的 unit 名**（如 `cloudflared-yb`），两个实例共存。

当前脚本默认把服务装成 `cloudflared`（会覆盖同名 unit），所以这种场景需要改脚本再装 —— 遇到就先跟用户确认要不要这么做，不要直接改。

## 4. 卸载后收尾

```bash
# 机器上确实没 cloudflared 了
pgrep -f cloudflared >/dev/null && echo "还在跑" || echo "已停"
systemctl is-active cloudflared 2>/dev/null || echo "服务已停"
ls /etc/cloudflared 2>/dev/null || echo "配置已清空"

# 云端（本账号的才删得掉）
python3 /opt/yuanbao-install/cf_api.py "$CF_API_TOKEN" accounts
python3 /opt/yuanbao-install/cf_api.py "$CF_API_TOKEN" tunnelowned "$CF_ACCOUNT_ID" "<UUID>"   # YES 才能删
python3 /opt/yuanbao-install/cf_api.py "$CF_API_TOKEN" deldns "$CF_ZONE_ID" "<域名>"
python3 /opt/yuanbao-install/cf_api.py "$CF_API_TOKEN" deltunnel "$CF_ACCOUNT_ID" "<UUID>"
```

注意 Cloudflare 后台删除隧道时提示的「在每台计算机上跑 `sudo cloudflared service uninstall`」：**云端删除不会停掉机器上的进程**，必须每台机器都跑一次本机卸载，否则进程会一直重连失败。

## 5. 凭据相关的坑

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `9109 Invalid access token` | 认证方式不匹配（Key 配了 Bearer，或 Token 配了邮箱） | Global API Key（`cfk_` 前缀 / 37 位 hex）→ 必须带 `CF_EMAIL`；API Token → 邮箱留空 |
| 给了 Token 却填了邮箱 | 走了 X-Auth-Key 分支 | `unset CF_EMAIL` |
| 有 `cert.pem` 却查不到隧道列表 | cert 属于别的账号 | 用 API Token 走 API，或用对应账号的 cert |
| 凭据曾明文出现在聊天/日志里 | 泄露 | 去后台**轮换**；Global API Key 尤其要换（等同账号全权） |
