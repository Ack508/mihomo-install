# Mihomo 通用安装、配置与使用指南（供人和 Agent 使用）

本文是一份独立操作手册，不依赖其他安装记录或现成脚本。覆盖常见 Linux、无 systemd 的容器/服务器、macOS 和 Windows；其他系统沿用“检测环境 → 选择发行资产 → 配置 → 前台验证 → 适配进程管理”的流程。

默认方案：仅本机 HTTP/SOCKS5 显式代理，手动选择节点。不启用 TUN，不自动修改系统代理，不自动把所有终端切到代理。需要这些功能时应另行明确配置范围。

## 1. 怎么把这份文档交给另一个 Agent

把本文件上传到目标机器，或作为附件交给 Agent，再复制以下指令并填写参数：

```text
请完整阅读 mihomo.md，按文档完成安装与配置。

目标机器：<SSH 别名或当前机器>
订阅链接：<订阅链接>
代理启用方式：<所有终端启用 / 终端中手动启用>
节点要求：<地区或具体节点>
安装位置：<指定绝对目录>
```

Agent 执行时遵循以下要求，无需将这些要求重复写进指令：

先检查系统、CPU 架构、权限、现有安装、进程管理器、端口和代理设置。
若已有可用安装，优先复用；修改前备份，不重复启动，不覆盖无关配置。
缺省采用本文的本机显式代理方案，按实际系统选择安装包和启动方式。
不要在执行输出或交付报告中回显订阅链接、管理密钥和节点凭据。
根据目标机器实际情况选择架构、版本、Python 路径、节点名和终端加载方式。
完成配置校验、HTTP/SOCKS5 测试、管理接口检查和节点选择记忆验证。
最后报告版本、文件路径、启动/停止/启用代理命令、测试结果及未完成项。
若缺少订阅或目标机器访问条件，先完成其余可做步骤，再明确说明缺少什么。

只维护已有机器时，可使用：

```text
阅读 mihomo.md，在 <目标机器> 检查现有 Mihomo 并修复配置。
沿用现有订阅和代理作用范围，先备份再修改，最后按第 10 节验收。
```

## 2. 检测环境，确定安装分支

通过 SSH 操作时，以下命令在目标机器运行；`127.0.0.1` 指执行命令的机器。

Linux / macOS（Bash 或 zsh）：

```bash
uname -s
uname -m
id
command -v mihomo || true
command -v python3 || true
command -v curl || true
```

Linux 另查 `cat /etc/os-release`、`ps -p 1 -o comm=`，以及 `ss -lntp`。macOS 用 `lsof -nP -iTCP:7890 -sTCP:LISTEN` 和相同方式检查 9090。

Windows PowerShell：

```powershell
[System.Runtime.InteropServices.RuntimeInformation]::OSArchitecture
Get-Command mihomo.exe, python, py, curl.exe -ErrorAction SilentlyContinue
Get-Process *mihomo* -ErrorAction SilentlyContinue
Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue |
    Where-Object { $_.LocalPort -in 7890,9090 }
```

如已有 Mihomo，先执行其绝对路径加 `-v`，检查实际启动参数及配置目录。GUI 客户端可能也在管理 Mihomo，不要再启动第二个实例争抢端口。

| 检测结果 | 发行资产架构选择 |
|---|---|
| `x86_64` / `AMD64` / `X64` | amd64；按 CPU 指令集选择兼容变体 |
| `aarch64` / `arm64` / `Arm64` | arm64 |
| `armv7l` | 根据发行资产说明选择 armv7，核对 ABI |
| 32 位 x86 | 核对是否提供 386 资产 |
| MIPS、RISC-V、其他架构 | 核对官方资产及 ABI，不猜包名 |

amd64 的指令集变体不能只看“64 位”决定；发生 `Illegal instruction` 应改用兼容构建。WSL 与 Windows 是不同运行环境，安装包按实际运行核心的环境选择。

## 3. 安装核心，选择目录

从 [官方 Releases](https://github.com/MetaCubeX/mihomo/releases) 获取与 OS、CPU 和 ABI 匹配的稳定版。用户指定版本时遵从指定，不把历史版本当作最新版本。官方另有 [安装选择器](https://wiki.metacubex.one/startup/)。

下载前读取所选 release 的资产列表，不凭空拼接文件名；可用 GitHub API `https://api.github.com/repos/MetaCubeX/mihomo/releases/tags/<版本标签>` 查询。下载后计算 SHA-256，并与官方资产 digest 或校验清单比较；没有官方摘要时如实记录校验限制。

| 系统/权限 | 安装方式 |
|---|---|
| Debian / Ubuntu，有管理员权限 | 检查 `.deb` 元数据架构后 `sudo dpkg -i <包的绝对路径>` |
| RPM 系发行版，有管理员权限 | 检查 RPM 元数据，使用系统包管理器安装匹配包 |
| Linux，无管理员权限 | 解压官方二进制到用户可写目录，`chmod 700` 后使用绝对路径 |
| macOS | 使用匹配芯片的 Darwin 资产，解压后设置可执行权限 |
| Windows | 解压匹配架构的资产，保存为固定路径的 `mihomo.exe` |

下载受限时可在其他机器下载、校验后传入。保留 HTTPS 证书校验；不要用 `curl -k` 掩盖下载问题。解压前确认压缩格式，`.gz` 和 `.zip` 使用相应工具。

安装完成执行 `mihomo -v`；若未加入 PATH，则使用可执行文件绝对路径。

安装位置指核心程序的保存目录。用户指定时，核心保存为该目录下的 `mihomo`（Windows 为 `mihomo.exe`）；未单独指定配置目录时，配置、订阅缓存、管理脚本和文件日志也保存在该目录。已由服务管理器接管的日志可以保留在 journal 等原有位置。路径含空格时正确引用，启动命令和服务定义始终使用实际绝对路径。

例如指定安装位置 `/opt/mihomo`，且未另指定配置目录，则核心为 `/opt/mihomo/mihomo`，主配置为 `/opt/mihomo/config.yaml`。这只是新安装的目录规则；已有安装优先沿用实际位置。

指定保存目录时优先使用官方便携二进制；DEB/RPM 的程序位置通常由包决定，不能把下载目录当成程序安装目录。若用户选择自动位置，可参考：

| 场景 | 核心保存位置示例 |
|---|---|
| Linux / macOS 当前用户，便携安装 | `$HOME/.local/bin/mihomo` |
| 管理员管理的 Linux，便携安装 | `/usr/local/bin/mihomo` |
| DEB / RPM 安装 | 使用包实际安装的路径，通过 `command -v mihomo` 核对 |
| Windows 当前用户 | `$env:LOCALAPPDATA\mihomo\mihomo.exe` |

建议使用独立目录，不覆盖 `/etc/mihomo` 或 GUI 的配置：

| 场景 | 配置目录示例 |
|---|---|
| Linux / macOS 当前用户 | `$HOME/.config/mihomo` |
| 管理员管理的 Linux 服务 | `/var/lib/mihomo` 或已规划的专用目录 |
| Windows 当前用户 | `$env:LOCALAPPDATA\mihomo` |

后续命令约定以下变量；每个新终端都需重新设置为实际值。示例使用自动选择的目录；用户指定安装位置或配置目录时，相应修改这些变量。

```bash
# Linux / macOS；MIHOMO_BIN 必须指向实际已安装的核心。
export MIHOMO_DIR="$HOME/.config/mihomo"
export MIHOMO_BIN="$(command -v mihomo)"
test -n "$MIHOMO_BIN" || { printf '请设置 MIHOMO_BIN 为核心绝对路径\n'; false; }
mkdir -p "$MIHOMO_DIR"
chmod 700 "$MIHOMO_DIR"
```

```powershell
# Windows；可执行文件路径按实际安装位置填写。
$MihomoDir = Join-Path $env:LOCALAPPDATA 'mihomo'
$MihomoBin = Join-Path $MihomoDir 'mihomo.exe'
New-Item -ItemType Directory -Force -Path $MihomoDir | Out-Null
```

修改已有配置、启动文件或服务前创建带时间戳的备份。目录和配置文件必须允许实际运行核心的账户读写；不要同时用多个账户操作同一缓存。

## 4. 创建主配置并接入订阅

在配置目录保存 `config.yaml`，将以下两个占位符替换为真实值。YAML 字符串需正确转义，尤其是包含引号的值；可用 JSON 字符串编码得到 YAML 兼容的双引号字符串。

管理密钥可在目标机器使用 Python `secrets.token_hex(32)` 生成。Windows 可使用 `py -3`，Linux/macOS 通常使用 `python3`；先确认 Python 3 可用。不要把生成结果提交到聊天或版本库。

```yaml
mixed-port: 7890
allow-lan: false
bind-address: 127.0.0.1
mode: rule
log-level: warning
ipv6: false

external-controller: 127.0.0.1:9090
secret: "<替换为随机管理密钥>"

proxy-providers:
  subscription:
    type: http
    url: "<替换为完整 HTTPS 订阅链接>"
    path: ./subscription.yaml
    interval: 86400
    proxy: DIRECT
    header:
      User-Agent:
        - clash.meta
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 300
      timeout: 5000
      lazy: false

proxy-groups:
  - name: PROXY
    type: select
    use:
      - subscription

rules:
  - IP-CIDR,127.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR,10.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR,172.16.0.0/12,DIRECT,no-resolve
  - IP-CIDR,192.168.0.0/16,DIRECT,no-resolve
  - MATCH,PROXY

profile:
  store-selected: true
```

该模板仅处理送入代理端口的请求，不自动接管整台机器。`ipv6: false` 也不等于禁用系统 IPv6。规则中的内网网段不是所有网络环境的完整直连清单，有特殊网段时补充。

订阅应返回 Mihomo/Clash 兼容节点数据。检查实际响应格式、节点数量及协议支持，不能只凭 HTTP 200 或 Content-Type 判断；Base64 URI 列表或 HTML 页面不能直接当作节点 YAML。优先向服务商取得兼容订阅，避免将含令牌的订阅交给未知转换服务。

如果已取得完整 Clash YAML，可在目标机器用 YAML 安全解析器提取顶层非空 `proxies` 列表，保存为 `subscription.yaml`（顶层仍为 `proxies:`）；不要直接照搬远端监听、DNS 和路由配置。YAML 解析器属于可选工具依赖，应在目标机器的虚拟环境或系统包中安装，不能假定存在 PyYAML。

`type: http` 会定时更新本地节点缓存；静态离线节点可改为 `type: file` 并移除 `url`、`interval`、`proxy`、`header`。如果需要在线更新，必须确保原始订阅本身也是兼容格式。更多字段见 [官方代理集合说明](https://wiki.metacubex.one/config/proxy-providers/)。

`proxy: DIRECT` 意味着订阅直连下载。若直连受阻，先用离线节点或已有可用代理完成引导，再调整更新路径，避免“必须先有订阅才能更新订阅”的循环。管理接口和代理端口保持回环监听。

Linux/macOS 对主配置、订阅缓存、备份及日志设置仅运行账户可读写的权限，例如 `chmod 600 <文件>`；Windows 使用相应 NTFS ACL。首次运行后检查核心新建文件的权限。

## 5. 校验与前台启动

先校验，成功后再启动：

```bash
"$MIHOMO_BIN" -t -d "$MIHOMO_DIR" -f "$MIHOMO_DIR/config.yaml"
"$MIHOMO_BIN" -d "$MIHOMO_DIR" -f "$MIHOMO_DIR/config.yaml"
```

```powershell
& $MihomoBin -t -d $MihomoDir -f (Join-Path $MihomoDir 'config.yaml')
if ($LASTEXITCODE -ne 0) { throw 'Mihomo 配置校验失败' }
& $MihomoBin -d $MihomoDir -f (Join-Path $MihomoDir 'config.yaml')
```

前台运行会占用当前终端；在另一个终端验证。先确认订阅成功加载、7890/9090 监听正常，再决定常驻方式。`-t` 成功只代表配置校验通过，不证明订阅下载和代理网络可用。

## 6. 后台运行与停止：按环境选择一种

切换到后台管理前，先在前台终端按 Ctrl+C 停止测试实例。只选择一种管理方式，优先复用已有服务。

### 6.1 Linux，systemd 正在运行且有管理权限

只有 PID 1 为 systemd 且服务管理器可用时才走此分支。以下为服务模板，由 Agent 替换所有占位符后保存到 `/etc/systemd/system/mihomo.service`。所有路径使用实际绝对路径，不写 `$HOME`；含空格的路径按 systemd 语法处理。

```ini
[Unit]
Description=Mihomo local proxy
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=<实际运行账户>
WorkingDirectory=<配置目录绝对路径>
ExecStart=<核心绝对路径> -d <配置目录绝对路径> -f <配置文件绝对路径>
Restart=on-failure
RestartSec=3
UMask=0077

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mihomo.service
sudo systemctl status mihomo.service --no-pager
sudo journalctl -u mihomo.service -n 50 --no-pager
# 日常重启 / 停止
sudo systemctl restart mihomo.service
sudo systemctl stop mihomo.service
# 取消开机启动（需要时）
sudo systemctl disable mihomo.service
```

如果同名服务已存在，先检查其定义，不能直接覆盖。普通用户可适配 `systemctl --user` 服务；退出登录后是否保活取决于用户服务和 linger 配置，需单独验证。创建服务的原理参考 [官方运行服务说明](https://wiki.metacubex.one/startup/service/)。

### 6.2 无 systemd 的 Linux / 容器，以及 macOS 临时常驻

优先使用已有 supervisor、容器入口或 macOS launchd。若已安装 tmux，可将前台命令放在独立会话中运行：

```bash
# 如果此会话已存在，先检查它，不要重复创建。
tmux has-session -t mihomo
# 仅在会话不存在、且确认没有现有实例时执行：
tmux new-session -d -s mihomo
# 进入会话，设置第 3 节变量并执行第 5 节前台启动命令。
tmux attach-session -t mihomo
```

运行后用 Ctrl+B，再按 D 脱离会话；停止时重新进入会话并按 Ctrl+C。tmux 不提供开机启动或崩溃自动重启。需要长期托管时，Agent 应根据实际 supervisor/launchd/容器编写服务定义，记录日志、工作目录和重启策略，而不是调用不存在的 systemctl。

### 6.3 Windows 后台运行

确认配置校验通过且无旧实例后，在 PowerShell 启动隐藏进程：

```powershell
$MihomoArgs = '-d "{0}" -f "{1}"' -f $MihomoDir, (Join-Path $MihomoDir 'config.yaml')
$MihomoProcess = Start-Process -FilePath $MihomoBin -ArgumentList $MihomoArgs `
    -WorkingDirectory $MihomoDir -WindowStyle Hidden -PassThru `
    -RedirectStandardOutput (Join-Path $MihomoDir 'stdout.log') `
    -RedirectStandardError (Join-Path $MihomoDir 'stderr.log')
$MihomoProcess.Id | Set-Content (Join-Path $MihomoDir 'mihomo.pid')
```

随后检查进程是否仍存活、日志与端口是否正常。PID 文件只供定位，不能单凭 PID 判断身份；停止前核对实际可执行路径和命令行，避免 PID 复用时误停其他程序：

```powershell
$MihomoPid = [int](Get-Content (Join-Path $MihomoDir 'mihomo.pid'))
Get-CimInstance Win32_Process -Filter "ProcessId = $MihomoPid" |
    Select-Object ProcessId, ExecutablePath, CommandLine
# 核对确为本目录实例后执行：
Stop-Process -Id $MihomoPid
```

需要登录/开机自动启动时，使用计划任务或现有进程管理器，并明确运行账户、工作目录和失败重启策略。隐藏后台进程本身不等于服务，也不会自动崩溃重启。

## 7. 让应用使用代理

按“所有终端启用 / 终端中手动启用”的选择配置。未指定时采用终端中手动启用。Agent 根据实际系统选择后台管理方式，并确保启用代理时核心可用。下面的 URL 需跟随实际端口调整。

### 单个命令

```bash
curl --noproxy '' -x http://127.0.0.1:7890 --max-time 30 https://www.gstatic.com/generate_204
```

Windows 用 `curl.exe`，避免旧 PowerShell 的 `curl` 别名。`--noproxy ''` 显式避免已有 NO_PROXY 影响这次测试。

### 当前 Bash / zsh 终端

```bash
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
export all_proxy=socks5h://127.0.0.1:7890
export HTTP_PROXY="$http_proxy" HTTPS_PROXY="$https_proxy" ALL_PROXY="$all_proxy"
export no_proxy=localhost,127.0.0.1,::1
export NO_PROXY="$no_proxy"
# 停用当前终端的代理环境变量（不会停止核心）
unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY no_proxy NO_PROXY
```

按应用能力扩充 NO_PROXY；不同软件对 CIDR 网段的支持不同，不能假定所有工具都支持内网 CIDR。

### 当前 Windows PowerShell 终端

```powershell
$env:HTTP_PROXY = 'http://127.0.0.1:7890'
$env:HTTPS_PROXY = 'http://127.0.0.1:7890'
$env:ALL_PROXY = 'socks5h://127.0.0.1:7890'
$env:NO_PROXY = 'localhost,127.0.0.1,::1'
# 停用当前终端的代理环境变量
'HTTP_PROXY','HTTPS_PROXY','ALL_PROXY','NO_PROXY' | ForEach-Object {
    Remove-Item -LiteralPath "Env:$_" -ErrorAction SilentlyContinue
}
```

Windows 环境变量名通常不区分大小写。并非所有应用都读取代理环境变量，应按应用文档设置；`HTTPS_PROXY` 使用 `http://` 是连接本地 HTTP 代理并经 CONNECT 访问 HTTPS 的常见方式。

### 终端启用方式

- **所有终端启用**：为目标用户实际使用的各类 shell 配置自动加载，打开新终端时检查或启动核心并设置代理变量。修改 `.bashrc`、`.zshrc`、PowerShell profile 等文件前先备份；已打开的终端需重新加载。此选项作用于目标用户的终端及读取代理设置的程序，不扩展到其他账户或图形系统代理。
- **终端中手动启用**：不自动加载代理变量；用户在需要的终端执行启用命令，检查或启动核心并设置当前终端变量。关闭该终端或执行停用命令后结束该终端的代理设置。

Agent 应提供适合实际 shell 的启用、停用命令；已有核心运行时直接复用，不重复启动。SSH 无法改变另一个已打开终端的环境变量。

代理环境变量不会自动设置 Windows/macOS 图形系统代理。远程服务器上的 `127.0.0.1:7890` 也不会自动成为本地电脑代理；如需使用，应另行配置 SSH 本地转发，并核对本地端口占用。

## 8. 查询、切换节点、更新订阅与重载配置

为避免跨 shell 的 JSON、Unicode 和引号差异，下面提供一个仅依赖 Python 3 标准库的管理工具。将完整代码保存为配置目录内的 `mihomo_api.py`。它只调用本机接口，不启动或停止核心，不读取或输出订阅凭据。

密钥在运行时隐藏输入；自动化时可通过受控环境变量 `MIHOMO_SECRET` 传入，使用后清除。不要将真实密钥作为命令行参数。

```python
import argparse
import getpass
import json
import os
from pathlib import Path
from urllib.error import HTTPError, URLError
from urllib.request import ProxyHandler, Request, build_opener

parser = argparse.ArgumentParser(description="Mihomo localhost API helper")
parser.add_argument("--port", type=int, default=9090)
sub = parser.add_subparsers(dest="action", required=True)
sub.add_parser("version")
sub.add_parser("list")
pick = sub.add_parser("select")
pick.add_argument("name")
sub.add_parser("refresh")
reload_cmd = sub.add_parser("reload")
reload_cmd.add_argument("config", type=Path)
args = parser.parse_args()
if not 1 <= args.port <= 65535:
    parser.error("port must be between 1 and 65535")
secret = os.environ.get("MIHOMO_SECRET") or getpass.getpass("管理密钥: ")
if not secret:
    parser.error("管理密钥不能为空")
opener = build_opener(ProxyHandler({}))  # 管理请求始终直连本机

def request(method, path, payload=None):
    body = None if payload is None else json.dumps(payload, ensure_ascii=False).encode("utf-8")
    req = Request("http://127.0.0.1:%d%s" % (args.port, path), data=body, method=method)
    req.add_header("Authorization", "Bearer " + secret)
    if body is not None:
        req.add_header("Content-Type", "application/json")
    with opener.open(req, timeout=60) as response:
        raw = response.read()
        return json.loads(raw) if raw else None

try:
    if args.action == "version":
        print(json.dumps(request("GET", "/version"), ensure_ascii=False))
    elif args.action == "list":
        group = request("GET", "/proxies/PROXY")
        print("当前:", group["now"])
        print("\n".join(group["all"]))
    elif args.action == "select":
        group = request("GET", "/proxies/PROXY")
        if args.name not in group["all"]:
            raise ValueError("节点不在当前 PROXY 列表中，请先执行 list")
        request("PUT", "/proxies/PROXY", {"name": args.name})
        print("当前:", request("GET", "/proxies/PROXY")["now"])
    elif args.action == "refresh":
        request("PUT", "/providers/proxies/subscription")
        print("已请求刷新订阅；请重新列出节点并检查日志")
    elif args.action == "reload":
        config = args.config.resolve(strict=True)
        request("PUT", "/configs?force=true", {"path": str(config)})
        print("已重载配置；请重新查询节点与版本")
except HTTPError as exc:
    raise SystemExit("管理请求失败，HTTP %s；检查密钥、节点和日志" % exc.code)
except (URLError, OSError, ValueError, KeyError) as exc:
    raise SystemExit("操作失败 (%s)；检查核心、端口、路径及日志" % type(exc).__name__)
```

Linux / macOS：

```bash
python3 "$MIHOMO_DIR/mihomo_api.py" version
python3 "$MIHOMO_DIR/mihomo_api.py" list
python3 "$MIHOMO_DIR/mihomo_api.py" select '从列表复制的完整节点名'
python3 "$MIHOMO_DIR/mihomo_api.py" refresh
# 先执行第 5 节的 -t 校验，再重载：
python3 "$MIHOMO_DIR/mihomo_api.py" reload "$MIHOMO_DIR/config.yaml"
```

Windows PowerShell（示例使用 Python Launcher；若只有 `python`，相应替换 `py -3`）：

```powershell
py -3 "$MihomoDir\mihomo_api.py" version
py -3 "$MihomoDir\mihomo_api.py" list
py -3 "$MihomoDir\mihomo_api.py" select '从列表复制的完整节点名'
py -3 "$MihomoDir\mihomo_api.py" refresh
py -3 "$MihomoDir\mihomo_api.py" reload "$MihomoDir\config.yaml"
```

管理端口变更时，将 `--port <实际端口>` 放在子命令之前。自定义组名或 provider 名时同步调整脚本里的 `PROXY` / `subscription`。

节点名必须完整匹配，包含表情、空格和符号。切换对新连接生效，已建立连接可能沿用旧节点。开启 `store-selected` 后仍需实测重载/重启的保留效果；节点被订阅删除或改名时不能保证原选择继续有效。

重载使用核心所在机器的绝对路径，配置应放在 `-d` 指定的工作目录内；若使用目录外路径，应按官方 SAFE_PATHS 机制处理。API 语义见 [官方管理接口文档](https://wiki.metacubex.one/api/)。

## 9. 日常操作速查

| 目的 | 操作 |
|---|---|
| 启动核心 | 选用第 6 节中实际配置的进程管理方式 |
| 当前终端使用代理 | 执行第 7 节对应 shell 的环境变量设置 |
| 当前终端停用代理 | 清除代理环境变量；不必停止核心 |
| 查询/切换节点 | `mihomo_api.py list` / `select '<完整名称>'` |
| 更新在线订阅 | `mihomo_api.py refresh`，再检查节点和日志 |
| 修改配置 | 备份 → 编辑 → `mihomo -t` → 重载或重启 → 验证 |
| 停止核心 | systemd stop / 前台 Ctrl+C / 核对实例后停止进程 |
| 查看日志 | systemd journal、前台输出、Windows 日志或实际管理器日志 |

重启进程不会自动给现有终端设置代理；清除变量不会自动停止核心。更换端口时同步修改配置、应用变量、测试和管理工具参数。

## 10. 验收与排障

下列 curl 命令适用于 Linux/macOS；Windows 把 `curl` 换为 `curl.exe`。端口按实际值调整。PowerShell 旧版本对空字符串参数的传递可能不同，应确认 `--noproxy` 的参数正确传给 curl.exe。

```bash
# HTTP 代理；期望最终状态码为 204
curl --noproxy '' -sS -o /dev/null --max-time 30 -x http://127.0.0.1:7890 -w '%{http_code}\n' https://www.gstatic.com/generate_204
# SOCKS5，由代理侧解析域名；期望 204
curl --noproxy '' -sS -o /dev/null --max-time 30 --socks5-hostname 127.0.0.1:7890 -w '%{http_code}\n' https://www.gstatic.com/generate_204
# 出口 IP 和地区
curl --noproxy '' -fsS --max-time 30 -x http://127.0.0.1:7890 https://www.cloudflare.com/cdn-cgi/trace
```

Windows 将 `/dev/null` 换为 `NUL`；旧 PowerShell 如无法正确传递空字符串，先临时移除当前进程的 NO_PROXY，或改用 `--noproxy` 的非匹配域名参数，然后恢复原设置。

Agent 交付前逐项记录：

- 核心版本、OS/架构正确，记录下载来源及摘要核对结果。
- 配置校验通过，节点已加载，实际进程和监听端口与配置一致。
- 管理接口需要正确密钥，API 的 list 返回实际节点。
- HTTP 与 SOCKS5 测试成功，出口地区满足用户需求。
- 设置变量后的目标应用实际能走代理；不能只测试显式 `curl -x`。
- 重复执行启动操作不产生第二个核心；重载/重启后检查选中节点。
- 常驻、开机启动、失败重启只在对应功能确实配置并验证后报告完成。
- 未满足项明确标注原因，不把跳过或未测试写成通过。

| 现象 | 优先检查 |
|---|---|
| Exec format error | 操作系统、CPU 架构、ABI、资产是否匹配 |
| Illegal instruction | CPU 指令集与二进制变体是否兼容 |
| systemctl 不可用 | PID 1 和实际进程管理器，改走第 6.2 节 |
| 端口占用 | 现有核心、GUI 客户端或其他程序，不盲目结束进程 |
| 订阅失败/没有节点 | TLS、鉴权、格式、UA、直连条件、缓存权限 |
| API 401 | 管理密钥是否与当前运行实例一致 |
| API 拒绝连接 | 核心、实际管理端口、回环地址及日志 |
| curl 可用但应用不可用 | 应用是否读取变量、代理类型、进程继承和 NO_PROXY |
| 网站 403 / challenge | 代理出口限制或网站验证；不直接等同于代理失效 |
| 旧终端仍不走代理 | 旧终端未加载变量，或应用启动时未继承变量 |

网站首页通过不代表其 API 可用；API 应使用该服务实际接口单独验证。日志可能包含请求信息，报告前按实际内容脱敏。

## 11. 最终交付格式

```text
目标机器 / OS / 架构：
Mihomo 版本 / 可执行文件路径：
配置目录 / 日志位置：
安装来源 / 摘要核对结果：
代理地址 / 管理地址（不含密钥）：
所选节点 / 出口地区：
启动 / 停止 / 重载命令：
应用启用 / 停用代理命令：
后台管理 / 开机启动 / 崩溃重启情况：
配置、HTTP、SOCKS5、应用、选择记忆等测试结果：
备份位置 / 恢复方法：
未完成项及原因：
```

恢复时先识别并停止本次创建的实例或服务，再恢复备份配置及启动文件、撤销本次新增的变量和服务配置；不要删除原有订阅或停掉其他实例。
