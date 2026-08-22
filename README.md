
<img width="2560" height="1280" alt="deepseek-harness" src="https://github.com/user-attachments/assets/904062ea-a929-4915-a7e9-af5d408d0e39" />

# DeepSeek Harness desktop 一键安装器（桌面版客户端+插件市场+多模态视觉）使用说明
deepseek harness桌面客户端(含插件市场和多模态视觉) 一键安装包，双击DeepSeekHarnessSetup-desktop-3.0.exe就可以一键安装，适合小白！

---
DeepSeek Harness 一键安装器（桌面版）稳定版本下载地址：https://github.com/yu-wenchao/deepseek-harness-desktop-Install/releases/download/v.30/DeepSeekHarnessSetup-desktop-3.0.exe

DeepSeek Harness 一键安装器（桌面版）（最终交付版，修复了插件市场部分插件拦截问题）客户端反向拦截：代理劫持 DSH CLI 调用（不用改插件市场源码）下载地址：https://github.com/yu-wenchao/deepseek-harness-desktop-Install/releases/download/v.30/DeepSeekHarnessSetup-desktop-4.0.exe

---

# Agent Note: 拦截 GitHub Release 压缩包 URL，修复插件市场一键安装

Status: implemented

[English](2026-08-22-plugin-market-github-release-install.md) | 中文

## 问题

第三方插件市场（dshmarket）不由我们维护，也无法修改。用户点击其「确认安装」后，它会执行
`dsh plugin --profile <p> add
"<https://github.com/owner/repo/releases/download/tag/pkg-ver.tgz>"`。`dsh` 命令行把这个
spec 原样转发给 pnpm，而 pnpm 的 URL 分类器匹配到 `github.com/...` 主机后，会走 Git 解析路径
（`resolveGit` / `resolveRef`），试图解析一个静态 Release 资源并不存在的 tag ref，于是在下载
发生之前就抛错，导致从 GitHub Release 托管的插件无法一键安装。

## 决策

在 harness CLI 入口（`harness/apps/cli/lib/bin.js`）顶部、commander 解析 `argv` 之前，加入一层
命令拦截。垫片（`dsh-market-github-release-shim.mjs`）会识别 `plugin ... add` 命令中任何
GitHub `/releases/download/*.tgz` 参数，把该资源下载一次到 `<root>/market-auto-plugin-cache/`，
并将 `process.argv` 改写为指向本地文件。随后 pnpm 会像安装普通预构建包一样安装，从而完全绕开
那条会崩溃的 Git 解析路径。安装器把该垫片作为资源嵌入，并在解压 harness 之后立即把垫片写到
`bin.js` 旁边，同时在 `bin.js` 顶部注入 `import "./dsh-market-github-release-shim.mjs";`
（幂等：已存在标记则跳过注入）。由于市场会通过 `process.argv[1]` 直接重新调用宿主 `bin.js`，
拦截会被自动触发；第三方市场的 UI/源码保持不变。

## 备选方案

**用 PATH 上的 `dsh` 启动器外壳或 spawn 包装器。** 不予采用，因为市场是直接重新调用宿主
`bin.js`，而不是通过 PATH 上的 `dsh` 查找，因此 PATH 层面的包装器会被绕过。

**修改第三方市场的 UI/源码。** 不予采用，因为我们无权修改第三方市场，且任何改动都无法在
市场更新后保留。

**把 URL 改写成 `github:owner/repo#tag` 形式的 git spec。** 不予采用，因为这会解析源代码仓库
而非预构建的 Release 资源，很可能失败或装错构建产物。

**在 harness 仓库里永久修改 `bin.js`。** 不予采用，因为 harness 是单独发布并被重新下载的，
静态改动会丢失。在安装时注入可以让 harness 源码保持未改，并在每次安装时重新应用修复。

## 影响

现在通过市场从 GitHub Release URL 一键安装可以正常工作。新增了一个基于 `fetch` 的下载逻辑，
并带本地缓存，重复安装时复用（通过 `.downloading` 临时文件做原子写入）。若下载失败，垫片会打印
中文错误并以非零状态退出，而不是在 pnpm 深处崩溃。拦截仅作用于引用了 GitHub Release 压缩包的
`plugin ... add` 命令；其他命令与安装源均原样透传。harness 的 `bin.js` 只在最终用户机器上的
安装期被修改，源码始终不变。

---
# 安装器 / 运行时 修复清单（2026-08-22）

本次修复涉及两部分：

- **安装器（本仓库 `release/installer`）**：插件市场 GitHub Release 一键安装失败 —— 已写入 Agent Note。
- **运行时（gitee harness）**：卸载插件后启动崩溃、重装覆盖丢失用户数据。

---

## 1. 插件卸载后程序启动崩溃（运行时 / gitee harness）

- **现象**：在插件面板卸载某个插件后，再次启动程序直接崩溃，无法进入主界面。
- **根因**：卸载只删除了插件目录，但插件清单 / 启用配置里仍残留对已删除插件的引用；harness 启动时按清单加载插件失败并抛出，导致整个进程退出。
- **修复**：启动自愈（boot self-heal）。启动时校验插件清单，对缺失或无效的插件条目做「跳过 + 清理」（写回修正后的清单），而不是让进程崩溃；缺失的运行时依赖自动修复而非终止。
- **验证**：卸载任意插件后重启，程序正常进入；清单中已无失效引用。

---

## 2. 重装 / 覆盖安装导致用户数据丢失（运行时 / gitee harness）

- **现象**：覆盖安装新版本或重装后，用户的配置、会话、已安装插件数据全部丢失，回到初始状态。
- **根因**：安装器 / 更新流程直接整体覆盖程序目录，没有保留用户数据目录与配置。
- **修复**：重装流程在写入前先备份用户数据（配置、会话、插件数据目录），安装完成后再还原；只更新程序文件，保留 `version.json` 与用户目录，做到「升级程序、保留数据」。
- **验证**：覆盖安装后用户配置与历史会话仍在；全新安装（无旧数据）不受影响。

---

## 3. 插件市场 GitHub Release 一键安装失败（安装器 / 本仓库）✅ 已写 Agent Note

- **Agent Note**：[`.agents/notes/implemented/bug-fix/2026-08-22-plugin-market-github-release-install.md`](../../.agents/notes/implemented/bug-fix/2026-08-22-plugin-market-github-release-install.md)（含中文版 `.zh.md`）
- **现象**：在第三方插件市场（dshmarket）点击「确认安装」，地址是 `https://github.com/.../releases/download/.../*.tgz` 时安装失败。
- **根因**：`dsh plugin --profile <p> add "<github release tgz>"` 被原样转发给 pnpm；pnpm 把 `github.com/...` 误判为 Git 仓库，进入 `resolveGit / resolveRef` 去解析并不存在的 tag ref，下载前即抛错。
- **修复**：安装器注入一层命令拦截垫片（`dsh-market-github-release-shim.mjs`）。它在 harness CLI 入口 `bin.js` 顶部、commander 解析 `argv` 之前运行，识别 `plugin ... add` 中的 GitHub Release `.tgz` 参数，先把资源下载一次到 `<root>/market-auto-plugin-cache/`，再把 `process.argv` 改写为本地文件路径；pnpm 随即像安装普通预构建包一样安装，绕开崩溃的 Git 解析路径。第三方市场 UI / 源码保持不变；垫片随安装器资源嵌入，解压 harness 后幂等注入，重装后自动生效。
- **验证**：市场一键安装 GitHub Release 插件成功；非 Release 的其它安装源与其它命令均原样透传，不受影响。

---

DeepSeek Harness（DeepSeek 智能体生产环境）支持一键安装：`DeepSeekHarnessSetup-desktop-3.0.exe` 会替你完成**下载源码 → 安装全部依赖 → 配置模型网关 → 启动桌面客户端**的全部工作，全程无需命令行，装完即用。

---

## 1. 系统要求

| 项目 | 要求 |
|---|---|
| 操作系统 | **Windows 10 / Windows 11**（64 位） |
| WebView2 Runtime | 系统自带（Windows 10/11 已内置），无需额外安装 |
| 磁盘空间 | 安装目录约需 **4 GB**（运行时 + 源码 + 依赖 + 数据） |
| 网络 | 需可访问 `registry.npmmirror.com`（国内源）下载 900+ 依赖包 |
| 硬件 | 建议 8 GB 内存以上（首次建索引与依赖解析较吃资源） |

> 仅支持 Windows。macOS 版后续提供。
>
> Windows 7 需另装 WebView2 Evergreen 且不保证兼容，建议使用 Win10/11。

---

杀毒软件误报处理（重要）

本安装器未做商业数字签名（需要付费证书），360 等国内杀毒软件常会误报为"病毒/木马"并自动删除或拦截安装文件。程序本身完全安全，请放心。

请在安装前先关闭杀毒软件，安装完成后再重新开启：

右键任务栏右下角的 360 图标 → 选择 退出 / 退出并关机时启动? 取消，确认完全退出（部分版本需在设置里关闭"开机自启防护后再退出"）
右键安装器 → 选择「以管理员身份运行」，正常完成整个安装过程
安装完成后，重新打开 360，并在 恢复区 恢复被误删的文件（若有）
若未关闭杀毒导致安装失败：

打开 360 → 木马查杀 → 恢复区 → 恢复本安装器文件
360 弹窗提示时选 「允许运行」/「信任」，将该文件加入信任白名单
重新运行安装器即可继续
文件官方校验值（SHA256）可向发布方索取核对，确保文件没有被真正篡改、只是误报。

---

## 2. 一键安装流程

### 2.1 双击运行

双击 `DeepSeekHarnessSetup-desktop-3.0.exe`，打开安装器窗口：

![安装器界面：安装目录 / 源码地址 / 模型网关 / 开始安装]

- **安装目录**：默认 `D:\DeepSeekHarness`（无 D 盘时回退到系统用户数据目录），可点"浏览…"更改。
- **源码包地址**：内置 **gitee 国内主源 + GitHub 官方 codeload 备源**，自动按顺序尝试，一般无需改动。
- **默认模型网关**：指向模型网关服务地址，一般无需改动。

点 **"开始安装"**。

### 2.2 安装过程（6 步）

| 步骤 | 说明 |
|---|---|
| `[1/6]` | 释放内置 **Node v24 运行时 + pnpm** 到 `<安装目录>\runtime\` |
| `[2/6]` | 从源码站**下载 Harness 源码包**（约 31 MB） |
| `[3/6]` | 解压源码到 `<安装目录>\harness\`（自动归一化目录布局） |
| `[4/6]` | **在线解析全部依赖**（约 923 个包，首次 **10–20 分钟**，视网络而定） |
| `[5/6]` | 写入独立数据目录 **`<安装目录>\dsh-home\`** 和默认模型网关配置 |
| `[6/6]` | 释放**桌面客户端**、**在桌面创建 "DeepSeek Harness" 快捷方式**，并自动启动 |

窗口下方进度条和日志实时显示当前状态。期间可随时点"取消"中止。

> **安装较慢是正常现象**：首次需拉取 900+ 依赖包，进度取决于你的网络。日志出现 "Progress: ... added N" 表示解析进度。

### 2.3 安装完成

弹窗提示**"安装完成！"**，随后**桌面客户端窗口自动打开**，即可直接使用。

---

## 3. 使用

### 3.1 启动

三种方式任选其一：

1. **桌面快捷方式** —— 双击桌面上的 **"DeepSeek Harness"** 图标（推荐）；
2. **安装目录** —— 双击 `<安装目录>\DeepSeekHarnessSetup-desktop-3.0.exe`；
3. **start.bat** —— 双击安装目录下的 `start.bat`（同为桌面模式快捷入口）。

启动后自动拉起模型网关服务，并在**内嵌窗口**中显示工作台界面（地址为 `http://127.0.0.1:3080`，无需手动打开浏览器）。

### 3.2 退出

**直接关闭桌面客户端窗口**即可——服务会随之停止并释放端口，不会留有后台进程。

### 3.3 首次使用步骤

1. **打开设置 → 模型**，输入 DeepSeek API 密钥保存（见第 4 节"模型网关配置"）；
2. 点击 **"选择工作区"**，添加你的项目目录并选中；
3. 在底部输入框给 agent 派发任务即可。

---

## 4. 模型网关配置

安装器已写入默认配置（`<安装目录>\dsh-home\settings.yaml`）：

```yaml
llm-pi-ai:
  providers:
    remote-gateway:
      apiKeyEnv: DSH_GATEWAY_KEY
      api: openai-completions
      baseURL: https://deepseek-harness.239.ccwu.cc
```

为了让模型真正可用，**二选一**：

- **方式一**：把模型网关访问密钥写入环境变量 `DSH_GATEWAY_KEY`，然后重启桌面客户端；
- **方式二**：打开 **设置 → 模型**，直接填入 API 密钥并保存（即时生效，无需重启）。

> 在客户端界面里改模型配置同样持久化到 `dsh-home`，之后重开仍保留。

---

## 5. 常用文件与目录

| 路径 | 作用 |
|---|---|
| `<安装目录>\DeepSeekHarnessSetup-desktop-3.0.exe` | 桌面客户端（入口程序） |
| `<安装目录>\runtime\` | Node v24 便携运行时 + pnpm |
| `<安装目录>\harness\` | Harness 源码与依赖（`node_modules`） |
| `<安装目录>\dsh-home\` | **你的数据目录**：配置、工作区、会话记录、模型设置 |
| `<安装目录>\dsh-home\settings.yaml` | 模型网关等配置 |
| `<安装目录>\start.bat` | 启动入口（桌面模式） |

> **升级/备份**：备份 `dsh-home` 即可保留全部个人数据。`harness` 与 `runtime` 可随时重装。

---

## 6. 卸载

无独立卸载程序，手工三步即可彻底移除：

1. 关闭桌面客户端；
2. 删除安装目录（如 `D:\DeepSeekHarness`）；
3. 删除桌面 "DeepSeek Harness" 快捷方式。

如需保留模型设置，可只删 `runtime`、`harness` 和桌面快捷方式，保留 `dsh-home`。

---

## 7. 静默安装（自动化 / 企业分发）

安装器支持命令行静默安装，适合脚本、CI、批量装机：

```powershell
DeepSeekHarnessSetup-desktop-3.0.exe D:\DeepSeekHarness
```

- 自动使用命令行指定的目录，**跳过所有 GUI 交互**；
- 日志与安装结果写入 exe 所在目录的 `install-result.txt`：
  - `OK` + 说明 → 安装成功，桌面客户端已自动启动；
  - `FAIL` + 原因 → 安装失败。

---

## 8. 常见问题（FAQ）

### Q1：下载源码时报"网络问题 / 来源失败"
多源自动换源已内置（gitee → GitHub）。若全部失败：
1. 检查网络能否访问网络与 npm 国内源；
2. 在安装器"源码包地址"栏粘贴可用地址（支持逗号分隔多个，自动按序尝试）；
3. 重新点击"开始安装"。

### Q2：安装很慢，卡在 ≥ 10 分钟
属正常。首次需解析 900+ 依赖包，进度由网络决定。请勿关闭窗口；观察日志中 "added N" 数字是否在增长即视为正常推进。

### Q3：桌面客户端打不开（白屏 / 报 WebView2 相关错误）
桌面客户端依赖系统 WebView2 Runtime。请确认系统为 Windows 10/11；若被精简系统移除了 Runtime，可安装 Microsoft WebView2 Runtime 后重试。

### Q4：模型一直不回复 / 报密钥错误
模型网关需要有效密钥：设置 `DSH_GATEWAY_KEY` 环境变量，或在 设置 → 模型 中填写 API 密钥，然后重启客户端。

### Q5：更换电脑后如何迁移？
在新机器一键安装完成后，把旧机器的 `<安装目录>\dsh-home` 复制覆盖到新安装目录，即恢复配置、工作区与历史数据。

### Q6：想要用浏览器打开而不是内嵌窗口？ 
本安装器为桌面客户端形态。若需浏览器形态，可使用浏览器版安装器（`DeepSeekHarnessSetup-desktop-3.0.exe`，安装后自动打开 `http://127.0.0.1:3080`）。

---

## 9. 技术原理（简述）

安装器为单个自包含 EXE（C# 编译，零外部依赖），内嵌：

- **Node v24 便携运行时**（`node.zip`）
- **pnpm**（`pnpm-dist.zip`）
- **桌面客户端**（`DeepSeekHarness.exe` + WebView2 托管/原生 DLL 三件套）

安装时逐步释放以上资源；下载源码包后以 **PK\03\04 头校验**防止网页内容被误当压缩包；依赖解析走国内 npm 镜像；最后写入独立数据目录、创建桌面快捷方式并启动桌面客户端。

----------------------------------------------------------------------------------------------------------------------------


# deepseek-harness-Install

deepseek harness 一键安装，双击deepseekharnesssetup.exe就可以一键安装，适合小白！

安装包下载地址：https://github.com/yu-wenchao/deepseek-harness-Install/releases/tag/v1.0

<img width="622" height="457" alt="安装说明" src="https://github.com/user-attachments/assets/e10a513d-d657-46f6-91a4-ffa61de9b808" />


# DeepSeek Harness 一键安装说明

感谢使用 DeepSeekHarnessSetup.exe。本安装器会自动完成全部部署，全程约 **10 ~ 30 分钟**（取决于网速），期间请勿关闭窗口。

---

## 一、安装前准备

| 项目 | 要求 |
|------|------|
| 操作系统 | Windows 10 / 11（64 位） |
| 磁盘空间 | 至少 **3 GB** 空闲空间（建议 5 GB） |
| 网络 | 需要联网下载依赖 |
| 杀毒软件 | **强烈建议先关闭 360 等杀毒软件再安装**，否则可能被误报拦截（详见第六节） |
| 浏览器 | 任意主流浏览器（将自动打开） |

> 无需安装 Node.js、无需安装 pnpm，安装器内置完整运行环境。

---

## 二、开始安装

1. 双击运行 **DeepSeekHarnessSetup.exe**
2. 选择安装目录（默认 `C:\Users\<用户名>\AppData\Local\DeepSeekHarness`；建议装到空间充足的盘，例如 `D:\DeepSeekHarness`）
3. 保持"发布服务器地址"和"模型网关地址"为默认值，点击 **开始安装**
4. 窗口左侧会滚动显示安装日志，下方进度条会随时间前进

---

## 三、安装流程（6 个步骤，分别做什么）

| 步骤 | 内容 | 消耗时长 |
|------|------|----------|
| [1/6] 释放内置 Node 运行时 | 解开自带的 Node.js 运行环境（约 40 MB） | 约半分钟 |
| [2/6] 下载源码 | 从发布站点下载程序源码包（约 30 MB） | 视网速，可能数分钟 |
| [3/6] 解压源码 | 展开全部程序代码 | 约半分钟 |
| [4/6] 在线解析依赖 | **下载 900+ 个第三方依赖包并安装**（最耗时的一步） | **约 5 ~ 20 分钟** |
| [5/6] 初始化配置 | 生成个人数据目录、模型网关配置、一键启动脚本 | 几秒钟 |
| [6/6] 启动 Web | 启动本地服务，自动打开浏览器 | 几秒钟 |

> 进度条走到第 [4/6] 步时会比较久，这是正常的，请耐心等待。
> - 日志中出现 `[WARN]`（下载较慢提示）或 `Updated available` 等文字，均属正常，无需处理。
> - 日志中出现 "Will retry in …"（下载失败自动重试），安装器会自动重试，请勿关闭窗口。

---

## 四、安装完成后

- 浏览器会自动打开 **本机界面**：`http://127.0.0.1:3080`
- 同时会尝试打开 **模型网关首页**：`https://deepseek-harness.239.ccwu.cc`（如未弹出，可手动在浏览器输入）
- 安装目录下会生成 **start.bat**，以后每次使用双击它即可启动
- 默认模型网关已配置，首次使用时按提示填入网关访问密钥即可

---

## 五、为什么安装慢？（常见疑问）

本程序是完整的 AI 智能体工作台，不是轻量小工具，需要安装 **900 多个第三方依赖包**（如 TypeScript、Web 框架、原生二进制组件等），总计约 **1 GB** 数据通过网络下载。这一步在网络不佳时会较慢，安装器会自动重试下载失败的包，属于正常保护机制。

**提速建议：**
- 使用有线网络 / 更稳定的宽带
- 避开用网高峰时段
- 关闭其他占用带宽的下载任务

---

## 六、杀毒软件误报处理（重要）

本安装器未做商业数字签名（需要付费证书），**360 等国内杀毒软件常会误报为"病毒/木马"并自动删除或拦截安装文件**。程序本身完全安全，请放心。

**请在安装前先关闭杀毒软件，安装完成后再重新开启：**

1. 右键任务栏右下角的 **360 图标** → 选择 **退出** / **退出并关机时启动? 取消**，确认完全退出（部分版本需在设置里关闭"开机自启防护后再退出"）
2. 右键安装器 → 选择「以管理员身份运行」，正常完成整个安装过程
3. 安装完成后，重新打开 360，并在 **恢复区** 恢复被误删的文件（若有）

**若未关闭杀毒导致安装失败：**
- 打开 360 → **木马查杀 → 恢复区** → 恢复本安装器文件
- 360 弹窗提示时选 **「允许运行」/「信任」**，将该文件加入信任白名单
- 重新运行安装器即可继续

> 文件官方校验值（SHA256）可向发布方索取核对，确保文件没有被真正篡改、只是误报。

---

## 七、常见问题

| 现象 | 处理 |
|------|------|
| 停在某个步骤不动 | 多在下载中，耐心等待，最多等 30 分钟 |
| 进度条长时间不走 | 网络阻塞，可稍后重试安装（重装前先删掉上次残留的安装目录） |
| 浏览器没自动打开 | 手动打开 `http://127.0.0.1:3080`；服务未就绪时先双击 `start.bat` 再打开 |
| 安装目录损坏/想重装 | 删除原安装目录，再运行安装器重新安装 |


特别感谢
特别感谢 DeepSeek Harness 原始仓库 和 DeepSeek AI 团队。DSH Desktop 基于固定版本的上游源码构建，核心的智能体、模型、工具、会话、Web UI 和插件生态都来自这个项目。

同时感谢 Cordis 项目提供的插件化基础。没有这些开源项目，就不会有 DSH Desktop。

也感谢 Koishi.js 项目和社区长期积累的插件化实践、工具与经验，以及所有参与讨论、测试、反馈和插件开发的社区成员。

---
本项目遵循 MIT License。

本项目是基于 DeepSeek Harness 构建的一键安装器，并非 DeepSeek 官方产品。

本项目完全开源免费。如果有人向您以任何形式出售此软件，请拒绝交易。

如有任何问题，请将安装窗口的**完整日志文字**反馈给发布方即可快速定位。
