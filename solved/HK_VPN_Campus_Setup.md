收到。你的观察非常敏锐，直接抓住了网络底层的核心反馈——**日志的出现意味着流量真正命中了 Socket 端口**。昨天在 `Inclusive`（仅包含）模式下，`proxy.py` 没有任何输出，是因为系统的路由引渡机制存在盲区，流量根本就没有被送进那个 Python 进程，而是直接撞死了。

我们现在彻底抛弃“进程级接管（Inclusive）”，改用极其暴力且彻底的“全局路由接管 + CIDR 子网豁免（Exclusive）”。

按照你的要求，我已将 Markdown 更新。本次输出仅包含**第一阶段（正确操作 SOP）**，分两块内容呈现。准备好后，随时回复“生成下一部分”，我再为你深度剖析之前的陷阱以及背后隐藏的网络原理。

---

# [Skill SOP] 香港高校科研工作站：全局代理豁免 + SSH 内网穿透出海终极指南

> **文档元数据**
> * **适用生态**：香港高校内网（典型如 PolyU 理大 `158.132.x.x` 公网级局域网）
> * **硬件拓扑**：本地 Windows PC (直连校园网) $\leftrightarrow$ 校内 Ubuntu Server
> * **核心诉求**：本地维持 0ms 裸连内网服务器，同时让服务器内的 AI Agent 穿透 OpenAI 地理封锁。
> * **架构定性**：基于“全局 VPN 路由接管（Exclusive）” + “CIDR 子网物理豁免” + “SSH 反向隧道（RemoteForward）”。

---

## 第一部分：确保开启 VPN 不阻断校园内网（全局接管与子网豁免）

此部分的核心在于解决传统 VPN 开启后导致内网 SSH 断联的死局。我们使用 Default-Deny（排除流）策略，在系统底层强制划分网络租界。

### 1. 核心基建与清理
1. **卸载/停用一切强制全局型 VPN**（如 ProtonVPN 等，这类软件会强行篡改底层默认网关 `0.0.0.0/0`，物理掐死非网关层面的 Layer 3 握手，导致内网直接断线）。
2. 在 Windows 上安装支持高级拆分隧道（Split Tunneling）的 **Windscribe VPN**。

### 2. Windscribe “掀桌子”级 CIDR 豁免配置
1. 打开 Windscribe $\rightarrow$ Preferences (设置) $\rightarrow$ **Connection (连接)**。
2. 进入 **Split Tunneling (拆分隧道)**，将顶部大开关拨到 **`ON`**。
3. **【生死决断点】Mode（模式）**：强行切换为 **`Exclusive`（排除）**。
   *(逻辑：全电脑的所有应用、所有底层组件强制走美国 VPN，绝不漏网，从而彻底解决 WebRTC、DNS 等底层信息的环境泄露问题。)*
4. **清空 Applications 列表**：确保下方的应用列表里没有任何 `.exe` 文件。
5. 切换到右侧的 **`Hostnames / IPs`** 选项卡。
6. 点击 **`+`**，填入你的大学的完整 CIDR 网段：
   ```text
   xxx.xxx.0.0/16 (推荐将后两个地址选为掩码，免得校园分发的地址一直在变)
   ```

*(原理：在 Windows 路由表中开辟物理免死金牌。只要系统检测到去往 `xxx.xxx.x.x` 的握手包，一律跳过 VPN 隧道，直接走物理以太网卡。)*

---

## 第二部分：建立 Local PC 与 Remote Server 的隧道引渡出海

此部分旨在将远程服务器（Remote Server）的 AI 请求，安全地顺着 SSH 隧道回传到本机，再借由本机的 VPN 通道发往全球。

### 1. 准备本地中转站与 SSH 隧道

1. 在 Windows 本机安装轻量级 HTTP 代理中枢：`pip install proxy.py`。
2. 打开 Windows 的 `~/.ssh/config`，配置反向端口映射（RemoteForward）：
   ```text
   Host univ-server
       HostName 158.132.74.xx  # 必须是服务器当前真实的局域网分配 IP
       User shihao

       # 核心映射：将服务器的 7890 端口，反向绑死在 Windows 本机的 7890 代理监听口上
       RemoteForward 7890 127.0.0.1:7890
   ```



### 2. 标准开工点火军规（Launch Order）

网络链路存在严格的依赖次序，每次开工请**严格按以下 1-2-3-4 顺序执行**：

* **第 1 步（底层网络接管）**：直接在 Windscribe 客户端点击大开关，连上美国节点。此时你的 Windows PC 已具备完整的出海能力，且理大内网已被安全豁免。
* **第 2 步（升起代理中枢）**：在 Windows 桌面开一个终端（建议使用 Anaconda Prompt 以避开 WindowsApp 目录下的 0 字节幽灵软链接，防止触发 `WinError 5`），敲入：
  ```cmd
  python -m proxy --hostname 127.0.0.1 --port 7890
  ```


*(注：将其挂在屏幕一侧。在 Exclusive 模式下，当有请求命中该端口时，此终端会实时刷出连接日志。)*
* **第 3 步（凿开跨海隧道）**：打开 VSCode 或新的 cmd，执行 `ssh shihao@158.132.74.xx` 连入 Ubuntu 服务器。
* **第 4 步（链路查验）**：在 Ubuntu 服务器终端内输入：
  ```bash
  export https_proxy=http://127.0.0.1:7890
  curl -I https://api.openai.com
  ```


此时转头看 Windows 上的 Python 黑窗，若刷出绿字 **`CONNECT api.openai.com:443  200 OK`**，说明全链路合体完毕。

### 3. AI Agent / 纯 Python 进程出海写法

在远程服务器运行的任何 Python 脚本（如 `PaperForge`）最顶部，强行注入环境变量：

```python
import os

# 必须写在所有第三方网络请求库（如 openai, requests）被 import 之前！
os.environ["http_proxy"] = "http://127.0.0.1:7890"
os.environ["https_proxy"] = "http://127.0.0.1:7890"

from openai import OpenAI
client = OpenAI() # 发出的 API 请求将顺着隧道直达美国

```

***

以上是确保你能够无痛开工的“干净版本”。你随时可以回复 **“生成下一部分”**，我将为你深度复盘我们踩过的坑，以及那些“让你先用着，现在来给你解释”的底层网络原理。

---

## 第三部分：避坑血泪史与底层网络原理解剖（Traps & Principles）

本部分是对我们在搭建过程中经历的数十轮“试错与重构”的法医级复盘。网络工程往往是“知其然，更要知其所以然”。以下总结了我们在复杂内网环境下，顺着直觉走却踩中的 8 大逻辑陷阱，以及背后的核心网络原理。

> **隐私脱敏说明**：本章节已对特定的高校名称、公网 IP 段（如 `111.222.x.x`）及用户 ID 进行匿名化处理，统称为“校园网”和“内网公网化 IP”。

---

### 陷阱一：免费全局 VPN 的“底层霸权”死锁

* **【起因逻辑链】**：我们需要让服务器绕过 OpenAI 的地理封锁。最符合常理的第一直觉是：直接在本地电脑下个免费的 VPN（比如 ProtonVPN），打开它，让全电脑流量翻墙，服务器连着电脑，自然也就翻墙了。
* **【案发现场】**：开启 ProtonVPN 的瞬间，校园网立刻断线。
* **【原理解剖】**：
  免费版全局 VPN 启动时，会在操作系统底层强行篡改**默认网关（Default Gateway `0.0.0.0/0`）**。校园网用来维持你设备在线状态的“心跳保活包”被强行塞进了发往海外的加密隧道，导致校园网关判定该设备离线，直接将你踢出。
  **幽灵效应**：哪怕你点了 Disconnected，后台服务依然会挂载 WFP 驱动锁，**物理掐死**本机向非网关发起的 Layer 3 握手，导致你一直遇到 `Connection timed out`。

### 陷阱二：“内网公网化”的名校富豪病

* **【起因逻辑链】**：既然全局 VPN 会把校园网掐断，我们想到了 Windows 的静态路由功能。思路是：给电脑注入一条规则（`route add 111.222...`），告诉电脑“只要是去校园网机房的流量，就走物理网卡，其余走 VPN”。
* **【案发现场】**：添加完路由表，开启 VPN 后，SSH 依然断联。
* **【原理解剖】**：
  寻常大学内网全是 `192.168.x.x` 或 `10.x.x.x`（私有保留地址）。但部分财大气粗的高校，**内网分配的直接是全球互联网挂号的正规公网 IP**（如 `111.222.x.x`）。
  绝大多数 VPN 的分流驱动极其死板——**它只把私有地址判定为“本地局域网”并放行**；一旦看到公网地址，管你路由表怎么写，一律按“公网流量”强行吸进海外隧道，再从海外绕回学校，被学校防火墙按“非法外部入侵”静默丢弃。

### 陷阱三：UDP 阻断与 MASQUE 协议撞墙

* **【起因逻辑链】**：既然 Windows 端的 VPN 路由太难调教，我们改变策略，打算直接在 Ubuntu 服务器内部安装 Cloudflare WARP（官方 CLI），让服务器自己解决出海问题，不连累本地电脑。
* **【案发现场】**：在 Ubuntu 终端敲下 `warp-cli connect`，屏幕提示 Success，但实际毫无代理效果。
* **【原理解剖】**：
  官方提示的 `Success` 只是**异步指令欺诈**，仅代表后台收到了命令。实际上它永远卡在 `Connecting` 状态。
  因为 WARP 新版底层采用基于 HTTP/3 的 **MASQUE（纯 UDP 持续流）协议**。高校校园网为了防范内网主机沦为 DDoS 肉鸡，**对 Layer 4 的非标准 UDP 大流量握手包实施高强度无差别物理封杀**。

### 陷阱四：直觉自雷 —— 试图把 VSCode 放入 VPN 白名单

* **【起因逻辑链】**：我们放弃了服务器端出海，换用了支持“按应用拆分隧道（Inclusive模式）”的 Windscribe VPN。思路很顺：我平时主要在 VSCode 里写代码、连服务器、跑 AI 插件，那我直接把 `vscode.exe` 勾选进 VPN 白名单，让 VSCode 翻墙不就行了？
* **【案发现场】**：一旦勾选，VSCode 的 SSH 终端当场暴毙。
* **【原理解剖】**：
  这是一个逻辑自雷。VSCode 建立远程连接的母体进程是 `ssh.exe`。把它放进白名单，意味着 **[你在桌面的敲击] $\rightarrow$ [数据包飞去美国达拉斯] $\rightarrow$ [达拉斯服务器尝试从外网访问校园内网机柜] $\rightarrow$ [校园防火墙物理超度]**。
  *永远记住：`ssh.exe` 必须留在本地港湾走物理网线，只有负责转发 API 请求的代理进程（潜望镜）才配进隧道。*

### 陷阱五：微软商店的“幽灵软链接 (Ghost Alias)”

* **【起因逻辑链】**：既然 VSCode 不能进白名单，我们明白真正需要翻墙的是负责中转的本地 Python 代理（`proxy.py`）。为了把 Python 添加进 Windscribe 的白名单，我们在 cmd 里敲了 `where python`，系统返回了 `AppData\Local\Microsoft\WindowsApps\python.exe`。我们直接把它塞进了 VPN。
* **【案发现场】**：开启 VPN 后，代理依然不通，Socket 甚至报错 `WinError 5 拒绝访问`。
* **【原理解剖】**：
  这是 Windows Store 的霸王条款陷阱。`WindowsApps` 目录下的 `.exe` 根本不是真程序，而是 0 字节的**假体（重解析点）**，用于自动弹窗引导你去商店下载。让 VPN 底层驱动去 Hook 一个假体的 Socket，不仅抓不到流量，还会触发越权崩溃。必须找准 Anaconda 等环境的“原生真身”路径。

---

### 🌟 陷阱六：Inclusive（仅包含模式）的底层大漏勺与 Web 风控

* **【起因逻辑链】**：历经千辛万苦，我们成功把真实的 `python.exe` 和浏览器放进了 VPN 的 Inclusive（仅包含）名单。测了 IP 确实显示美国，本以为万事大吉，兴冲冲去登录 OpenAI 的网页端签 Consent 知情书。
* **【案发现场】**：依然被 Cloudflare 终极人机验证精准拦截，提示不支持的地区。
* **【原理解剖】**：
  按进程拆分的 Inclusive 模式只能骗过普通的 IP 校验，但根本防不住 Web 风控的底牌读取。
  1. **DNS 借刀杀人**：浏览器查域名时，是委托系统底层的 `svchost.exe` 去查的。因为 `svchost` 不在你的 VPN 白名单里，导致你的 DNS 请求依然顺着校园网发到了香港本地。
  2. **WebRTC 真身穿透**：浏览器底层直接询问操作系统内核读取网卡硬件，获取了真实的校园网 IP，打包发给了 OpenAI。
  *这就是为什么我们最终要掀桌子，采用 **Exclusive（全局接管 + CIDR 豁免）**模式的原因。因为只有 Exclusive 能把整台电脑的灵魂（包括底层服务、时区、DNS）全部塞进隧道，只留一条极其狭窄的物理豁免通道给机房。*

### 陷阱七：VSCode Remote 的“环境变量黑洞”

* **【起因逻辑链】**：隧道彻底通了后，我们需要让跑在 Ubuntu 上的 Codex 插件知道本地代理的存在。按照 Linux 常识，我们在 Ubuntu 的 `~/.bashrc` 里写入了 `export https_proxy=...`。
* **【案发现场】**：在服务器终端里手动 `curl` 能通，但 VSCode 里的 Codex 插件依然报错 `error sending request for url`，死活连不上。
* **【原理解剖】**：
  `~/.bashrc` 只对你在终端里敲字的 **Interactive Shell（交互式终端）** 生效。而 VSCode 连上服务器后，在后台静默启动 `Extension Host`（插件宿主进程）时，使用的是 **Non-interactive Shell（非交互式）**，根本不读 `.bashrc`！
  导致底层插件引擎完全不知道代理的存在，依然裸奔冲向校园网关被墙死。必须在 VSCode 的 Remote Settings 里使用 `remote.extensionEnvironment` 进行硬注入。

### 陷阱八：SQLite 共享锁死（无限 Loading 转圈）

* **【起因逻辑链】**：解决环境变量后，网络彻底通畅，我们期待插件立刻亮起绿灯。
* **【案发现场】**：重新加载 VSCode，插件前端 UI 永远卡在 Loading 转圈状态。
* **【原理解剖】**：
  前几次的断网导致了插件底层（Rust编写）的恶性 I/O Bug。它疯狂往本地写入错误日志，把 SQLite 数据库撑爆。当内核尝试执行日志合并（WAL Checkpoint）时，直接把异步线程池**阻塞挂起**了。
  此时后台僵尸进程死死咬住共享内存写锁（`.sqlite-shm`），导致新进程永远拿不到数据库句柄，只能让前端无限期等待回调。必须在服务器端暴力 `killall -9` 并手动切除坏死的缓存文件。

---

### 🌟 核心剖析课：为什么昨天 Inclusive（仅包含）模式失败了，而今天 Exclusive（全局豁免）模式却神兵天降？

**案情回顾**：
昨天，我们把 `python.exe` 和 `chrome.exe` 放进了 VPN 的 Inclusive（仅包含）白名单。表面上测试 `ipleak.net` 发现 IP 确实变成了美国，但在登录 OpenAI 时，依然被 Cloudflare 终极人机验证（Turnstile）无情狙杀。
今天，我们把模式改成了 Exclusive（全局全走 VPN，只豁免校园网 IP 段），突然一切都通了，OpenAI 秒过！



**底层原理解剖（Web 风控的降维打击）**：
当你使用 Inclusive（仅包含应用）模式时，你只是派了一个保镖守在了 Chrome 软件的门口。流量确实去了美国，但浏览器在房间里依然向 OpenAI 泄露了 **3 个致命的“操作系统级底牌”**：

1. **DNS 借刀杀人（最隐蔽的死因）**：
   Chrome 遇到 `openai.com` 时，不会自己去查 IP，而是委托给 Windows 系统底层的 `svchost.exe` (DNS Client 服务) 去查。**但 `svchost.exe` 不在你的 VPN 白名单里！** 于是，Windows 顺着校园物理网线发出了 DNS 请求。OpenAI 一看：*“一个自称在美国的人，为什么他的 DNS 解析请求来自某亚洲高校？”* 瞬间判死刑。
2. **WebRTC 真身穿透**：
   现代浏览器为了支持视频通话（WebRTC），拥有直接向操作系统内核索要物理网卡硬件信息的特权。VPN 的应用白名单管不住内核问答。于是浏览器诚实地把你的真实校园网 IP 读了出来，打包发给了 OpenAI。
3. **系统时区背叛**：
   你的 IP 是美国达拉斯，但浏览器 JS 引擎读取系统时钟，发现是 UTC+8（亚洲时间）。时间差悖论直接触发风控。

**为什么 Exclusive（全局豁免）是终极解法？**
在 Exclusive 模式下，整台电脑（包括 DNS 服务、内核状态、时区探测）**全部被物理塞进了美国的加密隧道里**。我们只在网卡路由表的最顶端，开了一条极细的缝隙（CIDR `111.222.0.0/16`）：*“只有看到去往学校机房的数据包，才准放行！”*
这种掀桌子的玩法，不仅骗过了 OpenAI 史诗级的风控，还完美保全了内网的万兆互联。

---

### 陷阱六：VSCode Remote-SSH 的“环境变量黑洞”与 OAuth 断链

* **我们做了什么**：在 Ubuntu 服务器的 `~/.bashrc` 中写入了 `export https_proxy`。然后在 VSCode 界面点击 Codex 插件的登录，浏览器同意后，插件报错 `Token exchange failed: error sending request for url`。
* **原理解剖**：
  这是一个极其经典的 Linux 环境变量作用域盲区。
  * `~/.bashrc` 仅对 **Interactive Shell（交互式终端，也就是你能在里面敲字的黑框）** 生效。
  * 当你用 VSCode 连上远程服务器时，它在后台静默启动的 `Extension Host`（插件宿主进程）使用的是 **Non-interactive Shell（非交互式）**，根本不会去加载你的 `.bashrc`！
  结果就是，跑在后台的 Rust 插件核心拿着授权码，试图**不挂代理直接从校园网物理口裸奔访问 OpenAI 换取 Token**，当场被墙撞死。
  *解法：必须在 VSCode 的 `settings.json` 中使用 `remote.extensionEnvironment`，将代理强行注入到底层操作系统的环境变量树中。*

### 陷阱七：日志狂躁症与 SQLite 共享锁死（无限 Loading 转圈）

* **我们做了什么**：配置好代理后，Codex 插件不报错了，但是永远卡在 Loading 转圈状态。
* **案发现场**：服务器 `.codex/` 目录下出现了巨大的 `logs_2.sqlite` 和带着 `-wal`（预写日志）、`-shm`（共享内存）尾缀的文件。
* **原理解剖**：
  这是插件本身的底层 I/O 恶性 Bug。Rust 后端进程开启了 `TRACE` 级别的狂躁日志模式，疯狂往 SQLite 里写底层的网络帧，导致数据库迅速膨胀。
  当 SQLite 尝试进行 WAL Checkpoint（将日志合并回主库）时，耗时过长，**直接把 Rust 的异步线程池在底层给阻塞挂起了**！后台进程变成僵尸，死死咬住数据库写锁（`-shm`），导致前端 UI 永远等不到回调，只能无限转圈。
  *解法：必须暴力 `killall -9 codex` 斩首僵尸进程，物理删除坏死的 `.sqlite` 缓存，并在环境变量中强行关闭其低价值同步功能（`CODEX_DISABLE_PLUGINS=1`）。*