---
title: "Codex Desktop 代理配置实践"
date: 2026-05-06 23:10:00 +0800
author: Double Young

categories: [技术笔记]
tags: [Codex, OpenAI, GPT, 代理, Windows, Electron, Rust]
description: "分析 Codex Desktop、Rust app-server 与 Chromium 的网络路径，配置仅作用于 Codex 的代理方案并绕过 localhost。"
image:
  path: /assets/img/site/social-preview.png
  alt: 不具名的站点分享图

pin: false
math: false
comments: true
toc: true

published: true
render_with_liquid: false
---

最近用 Codex App 写代码蹬得很爽。唯一不爽的是，Codex App 设置里没有提供代理设置，需要代理软件开 TUN 模式才能正常工作。尽管 TUN 模式可以透明接管网络，使用起来几乎无感，但我不喜欢。代理设置成配置系统代理，Codex 是可以工作的，但我也不喜欢。我希望代理的工作环境是：

- 不打开 TUN。
- 不配置 Windows 系统代理。
- 尽量只让 Codex 相关进程走本地代理 `http://127.0.0.1:10808`

这当然是可以的。尽管 Codex App 没有显式提供代理设置，这不意味着它不进行任何代理规则处理。OpenAI 开源了 [openai/codex](https://github.com/openai/codex) 仓库，其中包含 Codex CLI、Rust core、app-server 等关键运行时部分。我们可以先从这个仓库入手，搞清楚哪些结论来自源码，哪些部分需要结合 Codex App 的桌面应用形态来判断。

这篇文章结合 `openai/codex` 介绍 Codex CLI 和 Codex App，接着探讨 Codex 网络相关请求，最后分析相应的代理设置，做出最适合我们的选择。

## 1. Codex CLI 和 Codex App

讨论代理前，需要先把 `openai/codex` 仓库和 Codex Desktop App 的关系拆开。这个仓库不是 Desktop 前端源码仓库，而是 Codex CLI 和 Rust 本地后端的开源仓库；Desktop App 会复用其中的 Rust 后端，但前端壳来自已安装的 Windows App。

### 1.1 Codex 核心组件和 CLI

从仓库 README 和代码结构看，`openai/codex` 首先是 Codex CLI 的开源仓库。它提供本地运行的 coding agent，也提供一套可被 Desktop、IDE extension 等 rich interface 复用的 Rust 后端能力。

几个关键组件大致是：

- `codex-cli`：npm 包装层，用来分发和启动 `codex`。
- `codex-rs/cli`：Rust CLI 入口，负责命令分发，例如 `codex`、`codex app`、`codex app-server`。
- `codex-rs/core`：agent 核心逻辑，包括会话、任务执行、工具调用、上下文管理等。
- `codex-rs/login`、`codex-rs/chatgpt`、`codex-rs/codex-client`、`codex-rs/backend-client`：负责认证、ChatGPT / Codex 后端请求、模型请求等。
- `codex-rs/app-server`、`codex-rs/app-server-protocol`：给 Desktop / IDE 这类图形界面使用的本地后端接口。它通过 JSON-RPC 暴露 thread、turn、account、config、apps、plugins 等能力。
- `codex-rs/tui`：终端里的交互 UI。
- `codex-rs/exec`、`codex-rs/sandboxing`、`codex-rs/codex-mcp`、`codex-rs/network-proxy` 等：负责命令执行、沙箱、MCP、受控网络代理等底层能力。

可以把仓库核心结构简化成几层：

```text
分发层：codex-cli
入口层：codex-rs/cli、codex-rs/app-server
交互层：tui、Desktop / IDE client
核心层：core
能力层：login、backend-client、exec、sandbox、mcp、plugins、network-proxy
```

其中 CLI 和 app-server 不是上下级关系，而是两个并列入口：

```text
codex CLI
  -> 终端入口
  -> 直接使用 core

codex app-server
  -> 本地 JSON-RPC 后端
  -> 给 Desktop / IDE 调用
  -> 也使用 core
```

所以，能从这个仓库里直接分析出来的是 Codex CLI 和 Rust 后端的网络行为，比如模型请求、auth refresh、account login、app-server 协议、WebSocket / SSE fallback、子进程环境等。

但 Codex Desktop 的前端界面源码不在这个仓库里。仓库里的 `codex app` 命令在 Windows 上只是查找已安装的 Codex AppID，然后通过 `shell:AppsFolder\...` 打开系统里的 Codex Desktop。也就是说，`openai/codex` 能解释 Desktop 背后的 Rust 后端，但不能直接逐个证明 Desktop 前端页面的网络请求。

### 1.2 Codex App：Rust 后端和 Desktop 前端

Codex Desktop App 的前端不在 `openai/codex` 仓库里，但它会通过本地 app-server 使用仓库里的 Rust 后端能力。这里尤其要区分两个可执行文件：

```text
Codex Desktop App

  app\Codex.exe
  ├─ Desktop 外壳进程：Electron / Chromium
  │    └─ Desktop UI 相关请求
  │       Settings / Usage / 账号 / 登录 / OAuth / 内置浏览器
  │
  └─ app\resources\codex.exe
       └─ Rust 后端二进制，来自 openai/codex
          └─ 以 app-server 方式提供本地能力
             模型对话 / auth refresh / rate limits / MCP / 子进程
```

这两个不是同一个进程。`app\Codex.exe` 是 Desktop 外壳，启动时接收 Chromium 参数；`app\resources\codex.exe` 是内置的 Rust 后端，负责 app-server 和会话请求。后端部分可以从开源仓库分析：Desktop 通过本地 app-server 使用 Codex 的 Rust 后端能力；Codex CLI 没有 Desktop 前端，基本直接落在同一套 Rust 会话路径上。因此，Codex CLI 和 Codex Desktop 的后端请求可以放在同一类里理解。

前端部分不能从 `openai/codex` 仓库直接看到源码，但可以从 Windows 安装包判断运行形态。在已安装的 Codex Desktop 目录里，可以看到典型 Electron / Chromium 结构：

```text
app/resources/app.asar
app/LICENSES.chromium.html
app/chrome_100_percent.pak
app/chrome_200_percent.pak
app/v8_context_snapshot.bin
app/resources/node.exe
```

因此，代理问题可以先简化为两类请求：

```text
Desktop UI 相关请求
  app\Codex.exe / Electron / Chromium 发起
  影响 Settings、Usage、账号、登录 / OAuth、内置浏览器 / WebView 等桌面界面

会话请求
  app\resources\codex.exe app-server / Codex CLI 发起
  影响对话、模型流、auth refresh、rate limits、子进程和 MCP
```

### 1.3 会话请求和 fallback

会话请求由 Rust app-server 或 Codex CLI 发起。它覆盖的是 Codex 真正干活的那部分：

- 模型对话。
- auth refresh。
- 模型响应携带的 rate limits / token usage。
- 插件、marketplace、apps 的后端请求。
- Codex CLI。
- shell、MCP、插件子进程。

从源码结构看，这一层和 Desktop UI 相关请求不同：它不经过 Desktop 前端的 Chromium 网络栈，而是由 Rust 进程里的网络客户端发起。Desktop 的 Rust app-server 和 Codex CLI 都会落到这一类请求里。

模型对话内部还有一个很关键的 fallback 机制。默认情况下，Codex 会话请求会优先使用 Responses WebSocket：

```text
wss://chatgpt.com/backend-api/codex/responses
```

如果 WebSocket 连接失败、断流或超时，Codex 不会立刻放弃这一轮会话，而是先按会话重试策略继续尝试。默认重试次数是 5 次。重试耗尽后，会禁用当前 session 的 WebSocket，并 fallback 到 HTTPS/SSE：

```text
POST https://chatgpt.com/backend-api/codex/responses
Accept: text/event-stream
```

所以会话请求内部其实有两条传输路径：

```text
优先：Responses WebSocket
失败：HTTPS/SSE fallback
```

这两条路径都属于会话请求，不属于 Desktop UI 相关请求。后面分析不同代理方式覆盖范围时，这个 fallback 机制非常关键。

### 1.4 localhost 要绕过代理

还有一类不是外部请求，但很重要：Desktop UI 和本地 app-server 之间要通过 `localhost` / `127.0.0.1` 通信。这条本地控制通道不应该走代理。

所以无论采用哪种方案，都要确保本机地址被绕过。具体怎么配置，放到后面的方案对比里展开。

也可以把整个问题简化成一句话：

> 会话请求走 Rust 后端 / Codex CLI，Desktop UI 相关请求走 Desktop 壳 / Chromium，localhost 必须绕过代理。

## 2. 不同代理方式的覆盖范围

下面按几种常见网络配置逐一对比。重点不是评判哪种方式绝对更好，而是看它分别覆盖 Desktop UI 请求、会话请求和 localhost 通信中的哪一部分。

### 2.1 不开代理直连

如果当前网络可以直接访问 `chatgpt.com`、`auth.openai.com`、`api.openai.com`，不开代理当然也可以正常使用。

但在需要代理的网络环境里，直连通常表现为：

| 请求类型 | 预期结果 |
|---|---|
| Desktop UI 请求 | 无法获取用量额度，内置浏览器无法访问外网 |
| 会话 / CLI 请求 | 取决于直连，通常不稳定 |
| localhost 通信 | 正常 |

这个模式下没有哪个代理层会帮 Codex 接管网络。所有外部请求都依赖直连。

### 2.2 TUN

TUN 是我一开始使用的方案。它是透明代理，作用在更底层的网络路由上。在 TUN 规则覆盖的情况下，应用不需要知道代理存在，`https`、`wss`、CLI 子进程、Chromium 请求通常都会被系统网络层接管。

预期结果：

| 请求类型 | 预期结果 |
|---|---|
| Desktop UI 请求 | 用量额度获取和内置浏览器被 TUN 接管 |
| 会话 / CLI 请求 | 被 TUN 接管 |
| localhost 通信 | 取决于 TUN 规则，不应被错误接管 |

TUN 的覆盖范围最完整，接近“让网络环境本身变好”。缺点也明显：它通常是全局影响，不符合“只让 Codex 走代理”的目标。

如果只追求省心，TUN 很强。这也是为什么一开始我用 TUN 时 Codex 基本可用。但如果想精确控制影响范围，尤其不想影响 WSL、Docker 和其他应用，TUN 就不够克制。

### 2.3 设置 Windows 系统代理

系统代理会明显改善 Desktop UI 请求。结合安装包结构和实际现象看，Desktop 壳使用 Chromium 网络栈，而 Chromium 通常会读取系统代理。

预期结果：

| 请求类型 | 预期结果 |
|---|---|
| Desktop UI 请求 | 用量额度和内置浏览器走代理 |
| 会话 / CLI 请求 | 可能先尝试 WebSocket 并 reconnect；失败后再 fallback 到 HTTPS/SSE 走代理，对话成功 |
| localhost 通信 | 通常正常，但仍建议 bypass |

我们之前观察到的现象是：设置系统代理后，对话会先 `Reconnecting...` 5 次，然后能连上。

这和源码逻辑吻合：

```text
默认先走 Responses WebSocket
-> WebSocket 没走通
-> 默认重试 5 次
-> fallback 到 HTTPS/SSE
-> HTTPS/SSE 走可用代理路径后成功
```

也就是说，系统代理能解决很多 UI 问题，但它不是对所有 Rust 网络路径和子进程都可靠。尤其是 WebSocket 和 CLI 工具这类请求，能不能吃到系统代理取决于具体库和工具实现。

### 2.4 `~/.codex/.env` 配置代理

从源码看，Rust 后端 `codex.exe` 启动时会加载 `~/.codex/.env`。因此，可以把代理环境变量写到这里：

```env
HTTP_PROXY=http://127.0.0.1:10808
HTTPS_PROXY=http://127.0.0.1:10808
ALL_PROXY=http://127.0.0.1:10808
NO_PROXY=localhost,127.0.0.1,::1
```

这些环境变量会进入 Rust `codex.exe` 进程，因此主要影响 Rust app-server、Codex CLI 以及它们启动的子进程。

预期结果：

| 请求类型 | 预期结果 |
|---|---|
| Desktop UI 请求 | 无法获取用量额度，内置浏览器无法访问外网 |
| 会话 / CLI 请求 | 被代理接管 |
| localhost 通信 | 正常，前提是 `NO_PROXY` 配好 |

这就是我们实际遇到的状态：

```text
只配 .env
-> 对话可用
-> 主页用量额度直接卡没了，设置里额度一直转圈
```

原因是 `.env` 只覆盖 Rust 进程环境，不会自动变成 Desktop 壳 / Chromium 的代理配置。Desktop UI 里的 Usage、账号、设置页部分请求仍然可能没有走代理。

因此 `.env` 是必要但不充分的方案。

### 2.5 启动项补 `--proxy-server`

只配 `.env` 后，会话 / CLI 请求已经有代理，但 Desktop UI 请求仍不一定覆盖。缺的这一块不是 Rust 环境变量，而是 Desktop 壳 / Chromium 的代理参数。

也就是说，2.4 之后还差的不是另一份环境变量，而是 Codex Desktop 的启动入口：启动真实的 `Codex.exe` 时，把 Chromium 代理参数一并传进去。脚本只是把这个启动方式固定下来，避免每次手工查找安装路径和输入参数。

```powershell
Start-Process -FilePath $CodexExe -ArgumentList @(
  "--proxy-server=http://127.0.0.1:10808",
  "--proxy-bypass-list=localhost;127.0.0.1;::1"
)
```

预期结果：

| 请求类型 | 预期结果 |
|---|---|
| Desktop UI 请求 | 被代理接管 |
| 会话 / CLI 请求 | 被代理接管，来自上一节 `.env` |
| localhost 通信 | 正常，因为显式 bypass localhost |

这个组合的关键是：

```text
~/.codex/.env
  负责会话请求：Rust app-server / Codex CLI / 子进程

--proxy-server
  负责 Desktop UI 请求：Desktop 壳 / Chromium

NO_PROXY / --proxy-bypass-list
  保护 localhost 本地控制通道
```

这里不把 `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY` 写进脚本，是为了让会话层代理只维护在 `~/.codex/.env`，Codex CLI 也能复用同一份配置。脚本只补 Desktop UI 这一层必须的 Chromium 启动参数。

---

## 3. 最终配置

最终我保留两份配置，但它们负责不同层次。

第一份是 `~/.codex/.env`，给 Rust app-server 和 Codex CLI 使用：

```env
HTTP_PROXY=http://127.0.0.1:10808
HTTPS_PROXY=http://127.0.0.1:10808
ALL_PROXY=http://127.0.0.1:10808
NO_PROXY=localhost,127.0.0.1,::1
```

这样即使不启动 Desktop，单独使用 `codex` CLI 时也能走同一份代理配置。

第二份是 Desktop 启动脚本，只给 Desktop 壳 / Chromium 加代理参数。脚本保存为：

```text
scripts/start-codex-desktop-proxy.ps1
```

```powershell
param(
    [string]$Proxy = "http://127.0.0.1:10808",
    [string]$CodexExe = ""
)

$ErrorActionPreference = "Stop"

$scriptRoot = Split-Path -Parent $PSCommandPath
$pathCache = Join-Path $scriptRoot "start-codex-desktop-proxy.path.txt"

function Get-CodexDesktopVersionFromPackageName {
    param([string]$Name)

    if ($Name -match "^OpenAI\.Codex_(?<Version>[^_]+)_") {
        try {
            return [version]$Matches.Version
        } catch {
            return [version]"0.0.0.0"
        }
    }

    return [version]"0.0.0.0"
}

function Get-RunningCodexDesktopExe {
    $processes = @(
        Get-Process -Name "Codex" -ErrorAction SilentlyContinue |
            Where-Object { $_.Path -like "*\WindowsApps\OpenAI.Codex_*\app\Codex.exe" }
    )

    if ($processes.Count -gt 0) {
        return $processes[0].Path
    }

    return $null
}

function Get-CodexDesktopExeFromPackageRepository {
    $packageRoot = "HKLM:\Software\Classes\Local Settings\Software\Microsoft\Windows\CurrentVersion\AppModel\PackageRepository\Packages"
    if (-not (Test-Path -LiteralPath $packageRoot)) {
        return $null
    }

    $packages = @(
        Get-ChildItem -LiteralPath $packageRoot -ErrorAction SilentlyContinue |
            Where-Object { $_.PSChildName -like "OpenAI.Codex_*" } |
            ForEach-Object {
                try {
                    $packagePath = (Get-ItemProperty -LiteralPath $_.PSPath -ErrorAction Stop).Path
                    if ($packagePath) {
                        [pscustomobject]@{
                            Name = $_.PSChildName
                            Version = Get-CodexDesktopVersionFromPackageName $_.PSChildName
                            Path = $packagePath
                        }
                    }
                } catch {
                }
            } |
            Sort-Object Version -Descending
    )

    foreach ($package in $packages) {
        $exe = Join-Path $package.Path "app\Codex.exe"
        if (Test-Path -LiteralPath $exe) {
            return $exe
        }
    }

    return $null
}

function Get-CodexDesktopExeFromWindowsApps {
    $windowsApps = Join-Path $env:ProgramFiles "WindowsApps"
    $packages = @(
        Get-ChildItem -LiteralPath $windowsApps -Directory -Filter "OpenAI.Codex_*" -ErrorAction SilentlyContinue |
            ForEach-Object {
                [pscustomobject]@{
                    Name = $_.Name
                    Version = Get-CodexDesktopVersionFromPackageName $_.Name
                    Path = $_.FullName
                }
            } |
            Sort-Object Version -Descending
    )

    foreach ($package in $packages) {
        $exe = Join-Path $package.Path "app\Codex.exe"
        if (Test-Path -LiteralPath $exe) {
            return $exe
        }
    }

    return $null
}

$runningDesktop = Get-RunningCodexDesktopExe

if ($runningDesktop) {
    Write-Warning "Codex Desktop is already running. Quit it completely first, then run this script again so proxy flags apply to the first process."
    if (-not $CodexExe) {
        $CodexExe = $runningDesktop
        Set-Content -LiteralPath $pathCache -Value $CodexExe -Encoding UTF8
    }
}

if (-not $CodexExe) {
    $CodexExe = Get-CodexDesktopExeFromPackageRepository
}

if (-not $CodexExe) {
    $CodexExe = Get-CodexDesktopExeFromWindowsApps
}

if (-not $CodexExe -and (Test-Path -LiteralPath $pathCache)) {
    $cachedCodexExe = (Get-Content -LiteralPath $pathCache -Encoding UTF8 -TotalCount 1).Trim()
    if ($cachedCodexExe -and (Test-Path -LiteralPath $cachedCodexExe)) {
        $CodexExe = $cachedCodexExe
    }
}

if (-not $CodexExe -or -not (Test-Path -LiteralPath $CodexExe)) {
    throw "Could not find Codex Desktop executable. Pass it explicitly with -CodexExe 'C:\Program Files\WindowsApps\OpenAI.Codex_...\app\Codex.exe'."
}

Set-Content -LiteralPath $pathCache -Value $CodexExe -Encoding UTF8

$args = @(
    "--proxy-server=$Proxy",
    "--proxy-bypass-list=localhost;127.0.0.1;::1"
)

Write-Host "Starting Codex Desktop with Chromium proxy $Proxy"
Write-Host "Executable: $CodexExe"
Start-Process -FilePath $CodexExe -ArgumentList $args -WorkingDirectory $HOME
```

脚本没有绑定某个固定版本号。路径解析顺序是：先尝试从已运行的 `app\Codex.exe` 进程读取路径；再从 Windows AppModel PackageRepository 注册表读取最新 `OpenAI.Codex_*` 包的安装目录；再尝试扫描 WindowsApps；最后才使用缓存路径。这样 Codex Desktop 更新后，即使旧缓存失效，也不会直接导致启动失败；仍然可以通过 `-CodexExe` 手动指定。

可以再给这个脚本创建一个 Windows 快捷方式，比如叫 `Codex Proxy`，以后都从这个入口启动 Codex Desktop。

## 4. 总结

这次问题的核心不是“代理有没有开”，而是“如果不想一直开 TUN，代理配置应该作用到 Codex 的哪一层”。

可以总结成下面这张表：

| 配置方式 | Desktop UI 请求 | 会话 / CLI 请求 | localhost 通信 |
|---|---|---|---|
| 不开代理直连 | 直连 | 直连 | 不需要代理 |
| TUN | 被 TUN 接管 | 被 TUN 接管 | 取决于 TUN 规则 |
| 设置 Windows 系统代理 | 被系统代理接管 | WebSocket 不一定被接管，HTTPS/SSE fallback 可能被接管 | 通常不需要代理，建议 bypass |
| `~/.codex/.env` 配置代理 | 直连 | 被 `.env` 代理接管 | 被 `NO_PROXY` 保护 |
| `.env` + 启动项补 `--proxy-server` | 被 `--proxy-server` 接管 | 被 `.env` 代理接管 | 被 `NO_PROXY` / `--proxy-bypass-list` 保护 |

最终结论：

> Codex CLI 是终端里的 Rust agent；Codex Desktop App 是 Desktop UI 加本地 Rust app-server。TUN 能兜住 Desktop UI 和会话请求，但影响范围太大。迁移到更小范围的方案后，`~/.codex/.env` 负责会话 / CLI 请求；Desktop UI 由 Chromium `--proxy-server` 补齐。最终方案是 `.env` 管会话，脚本管 Desktop UI，并同时绕过 localhost。
