

> 本文是一份**从零到完全跑通**的实操指南，记录的是真实踩坑后总结出来的**正确路径**——跳过了所有走不通的弯路（比如 iPad 上直接"打开文件夹作为库"、SSH 默认 22 端口连不上等），只保留最终验证有效的步骤。照着做，理论上可以一次成功。
> 
> 适用场景：你有一台 **Windows 电脑** 和一台 **iPad**，想用**免费的 GitHub 仓库**在两者之间同步 Obsidian 笔记，而不是花钱订阅官方的 Obsidian Sync。

---

## 目录

1. 整体方案与思路
2. 第一步：创建 GitHub 仓库
3. Windows 端完整配置
4. Windows 端网络优化：SSH over 443 通道（强烈建议直接做）
5. Obsidian Git 插件详细配置
6. iPad 端完整配置
7. iPad 端核心难题：iOS 沙盒限制与 Folder Sync 解决方案
8. 日常使用工作流
9. `.gitignore` 配置
10. 常见报错与排查手册
11. 收尾：如何把这篇文档本身传上 GitHub
12. 总结 Checklist

---

## 一、整体方案与思路

### 1.1 核心原理

把 Obsidian 的库文件夹变成一个 Git 仓库，对应 GitHub 上一个私有的远程仓库：

- **Windows 端**：装 Git 程序 + Obsidian Git 插件，插件自动帮你 `commit → pull → push`，全程不用敲命令行（命令行只在初始化那一次用到）。
- **iPad 端**：iOS 系统不允许任何 App 随便访问其他 App 的沙盒文件夹，Obsidian 也不例外，所以不能像 Windows 一样直接把 Git 仓库"打开"成库。正确做法是用专门的 Git 客户端 **Working Copy** 负责 Git 操作，再通过它的 **Folder Sync（文件夹同步）** 功能，把仓库内容"复制"进 Obsidian 能识别的本地库文件夹里。

### 1.2 两个关键坑，提前说明

这两个坑几乎所有人都会踩到，本文的步骤已经把解法直接嵌入到流程里了，这里先提前说明原理，方便你理解后面为什么要这么做：

**坑一：Git 命令行连 GitHub 经常"时好时坏"、连不上 443 端口。** 这是国内网络访问 GitHub 的常见问题，跟你的账号、仓库配置毫无关系。最彻底的解决办法不是重试，而是换用 GitHub 官方提供的 **"伪装成 443 端口的 SSH 通道"**（地址形如 `ssh://git@ssh.github.com:443/...`），比普通 HTTPS 稳定得多。Windows 和 iPad 两端都建议直接用这个方式，本文步骤里已经默认采用。

**坑二：iPad 上的 Obsidian 经常没有"打开文件夹作为库"这个选项。** 这不是你操作错误，是 Obsidian 在 iOS/iPadOS 上一个长期存在、时有时无的已知问题（Obsidian 官方论坛上有大量类似反馈）。就算入口存在，Obsidian 本质上也无法跨越 iOS 的沙盒机制直接打开 Working Copy 里的文件夹。所以本文不走这条路，而是用 Working Copy 的 **Setup Folder Sync** 功能，从根源上绕开这个限制。

---

## 二、第一步：创建 GitHub 仓库

1. 浏览器登录 [github.com](https://github.com/)，没有账号先注册。
2. 右上角 `+` → `New repository`。
3. 仓库名任意，例如 `obsidian_vault`。
4. **务必设为 Private（私有）**，笔记通常涉及个人隐私，不要公开。
5. **不要**勾选 "Add a README file"，也不要加 `.gitignore` 或 License，保持仓库为空——这样本地初始化后可以直接推送，不用处理"远程有内容、本地没有"导致的额外合并。
6. 点击 `Create repository`，记下仓库地址，后面会反复用到：
    - HTTPS 地址：`https://github.com/你的用户名/obsidian_vault.git`
    - SSH 地址：`git@github.com:你的用户名/obsidian_vault.git`

---

## 三、Windows 端完整配置

### 3.1 安装 Git for Windows（关键：装到默认路径）

1. 访问 `https://git-scm.com/download/win`，下载 64-bit 安装包。
2. 安装时**全程保持默认选项**，尤其注意两点：
    - **安装路径必须是默认的 `C:\Program Files\Git`**，不要改到别的盘符或自定义目录，也不要装完之后再手动移动这个文件夹。曾经实测过装到非默认路径（比如 `F:\Program File\Git`）会导致 Git 内部的 `git-remote-https` 组件"找不到"，push 时报 `remote helper 'https' aborted session` 这种莫名其妙的错误，重装到默认路径后问题立刻消失。
    - "Adjusting your PATH environment" 这一步选**默认推荐项**（Git from the command line and also from 3rd-party software），这样 Obsidian Git 插件才能找到系统里的 Git。
    - 组件勾选保持默认全选即可（Git LFS、Windows Explorer integration 等都不影响功能，装上无副作用）。
3. 安装完成后，**打开一个新的 PowerShell 窗口**（一定要开新窗口，旧窗口不会刷新环境变量），输入：

```powershell
git --version
where.exe git
```

确认版本号能正常显示，且 `where.exe git` 返回的路径是 `C:\Program Files\Git\cmd\git.exe`（标准默认路径）。

### 3.2 配置 Git 全局用户信息

```powershell
git config --global user.name "你的名字"
git config --global user.email "你的邮箱@example.com"
```

邮箱不强制和 GitHub 一致，但建议保持一致，方便以后在 GitHub 上关联提交记录和头像。

### 3.3 找到你的 Obsidian 库文件夹路径

如果你是很久之前安装的 Obsidian，忘了库文件夹在哪，最准确的办法：

1. 打开 Obsidian，随便打开一篇笔记。
2. 顶部菜单 `文件（File）` → 找 **"在系统资源管理器中显示"（Show in system explorer）**（也可能在右键点击左侧文件列表里的某个文件时出现）。
3. 点击后会打开 Windows 资源管理器并定位到笔记所在文件夹，地址栏显示的就是完整路径，比如 `C:\Users\你的用户名\Documents\Obsidian Vault`。

### 3.4 初始化 Git 仓库并首次推送

打开 PowerShell，**务必先确认当前路径是不是你的库文件夹本身**（很容易一不小心 `cd` 到了上一级目录，比如 `Documents` 而不是 `Documents\Obsidian Vault`，这样会把整个 Documents 文件夹都变成 Git 仓库，一定要小心）：

```powershell
cd "C:\Users\你的用户名\Documents\Obsidian Vault"
pwd
```

`pwd` 输出的路径要精确对应你的库文件夹，确认无误后继续：

```powershell
git init
git branch -M main
git remote add origin https://github.com/你的用户名/obsidian_vault.git
git add .
git commit -m "初始化 Obsidian 库"
git push -u origin main
```

推送时会要求登录：**用户名填 GitHub 用户名，密码框填 Personal Access Token（PAT），不是登录密码**。

#### 生成 Personal Access Token（PAT）的方法

1. GitHub 网页右上角头像 → `Settings` → 左侧最下方 `Developer settings`。
2. `Personal access tokens` → `Tokens (classic)` → `Generate new token (classic)`。
3. Note 随便填（比如 "Obsidian Windows"），Expiration 建议 90 天或更长。
4. Scopes 勾选 `repo`（完整仓库读写权限）。
5. 点击 `Generate token`，**立刻复制保存**，页面刷新后就再也看不到了。

如果这一步顺利推送成功，去 GitHub 网页刷新仓库页面能看到你的笔记文件，Windows 端命令行部分就算完成了。**但如果推送时反复出现 `Failed to connect to github.com:443` 这种连接失败或者时好时坏的情况，直接跳到第四节，把地址换成 SSH over 443 通道，一次性解决，不要在这里反复重试浪费时间。**

---

## 四、Windows 端网络优化：SSH over 443 通道（强烈建议直接做）

如果你在国内网络环境下，**几乎一定会遇到 Git push 连接不稳定**的问题——浏览器能正常打开 GitHub 网页，但 `git push` 却经常报错 `Could not connect to server` 或者卡住半天才失败。这是因为 Git 走的网络请求特征跟浏览器不一样，更容易被网络中间设备干扰。

最彻底的解决办法：改用 GitHub 官方提供的、**伪装在 443 端口上的 SSH 协议**，这个通道跟你平时打开网页用的端口一样，基本不会被限制。

### 4.1 生成 Windows 本地的 SSH 密钥

打开 PowerShell（任意目录都可以）：

```powershell
ssh-keygen -t ed25519 -C "你的邮箱@example.com"
```

接下来连续按**三次回车**，全部使用默认值（不设置密码短语，最简单）：

- `Enter file in which to save the key` → 回车
- `Enter passphrase` → 回车
- `Enter same passphrase again` → 回车

生成成功后，密钥文件会保存在 `C:\Users\你的用户名\.ssh\id_ed25519.pub`。

### 4.2 复制公钥并添加到 GitHub

```powershell
Get-Content "$env:USERPROFILE\.ssh\id_ed25519.pub" | Set-Clipboard
```

这行命令会直接把公钥内容复制到剪贴板。然后：

1. GitHub 网页 → 右上角头像 → `Settings` → 左侧 `SSH and GPG keys` → `New SSH key`。
2. Title 填 `Windows PC`。
3. Key type 选 `Authentication Key`。
4. Key 一栏直接 `Ctrl+V` 粘贴。
5. 点击 `Add SSH key` 保存。

### 4.3 修改远程仓库地址为 443 通道

```powershell
cd "C:\Users\你的用户名\Documents\Obsidian Vault"
git remote set-url origin ssh://git@ssh.github.com:443/你的用户名/obsidian_vault.git
```

### 4.4 测试连接

```powershell
ssh -T -p 443 git@ssh.github.com
```

首次连接会提示确认主机指纹（输入 `yes` 继续），看到类似下面这句就说明成功了：

```
Hi 你的用户名! You've successfully authenticated, but GitHub does not provide shell access.
```

如果提示 `Permission denied (publickey)`，说明公钥还没添加成功，回到 4.2 重新确认一遍。

### 4.5 正式推送

```powershell
git push -u origin main
```

这次走的是 443 端口的 SSH 通道，连接稳定性会有明显改善。以后无论是插件自动同步还是手动命令行操作，都会自动沿用这个 remote 地址，不需要重复配置。

---

## 五、Obsidian Git 插件详细配置

命令行部分跑通之后，装上插件让日常使用完全自动化，不用再碰命令行。

### 5.1 安装插件

1. 打开 Obsidian，进入 `设置（Settings）` → `第三方插件（Community plugins）`。
2. 如果安全模式还开着，先关闭（Turn on community plugins）。
3. `浏览（Browse）` → 搜索 `Git` → 找作者为 `Vinzent03` 的 **Obsidian Git** 插件 → 安装并启用。

### 5.2 关键参数配置

进入插件设置页，按下面的值配置（其余保持默认即可）：

|设置项|建议值|说明|
|---|---|---|
|Auto commit‑and‑sync interval (minutes)|`10`|每 10 分钟自动提交并同步一次|
|Auto pull interval (minutes)|`10`|每 10 分钟自动拉取一次远程改动|
|Auto push interval (minutes)|`0`（保持默认）|不需要单独设置，push 已经包含在下面的 commit-and-sync 循环里|
|Push on commit‑and‑sync|开启|每次自动提交后立刻推送|
|Pull on commit‑and‑sync|开启|每次自动同步时先拉取，减少冲突|
|**Pull on startup**|**开启**|**强烈建议打开**：每次打开 Obsidian 时自动先拉一次最新内容，避免基于旧版本编辑|
|Merge strategy|`Merge`（默认）|保持默认即可，够用|

配置完成后，命令面板（`Ctrl+P`）输入 `Git` 能看到 `Git: Commit and push`、`Git: Pull` 等命令，随时可以手动触发。

Advanced 分类下的所有选项（Custom Git binary path、环境变量、Custom base path 等）**保持默认空值/默认值即可**，这些是给特殊仓库结构用的进阶配置，正常场景改了反而容易出问题。

到这里，**Windows 端配置全部完成**。以后正常写笔记，插件会自动帮你提交、拉取、推送。

---

## 六、iPad 端完整配置

### 6.1 安装 Working Copy

1. App Store 搜索下载 **Working Copy**。
2. 免费版可以 clone/pull/查看，**push 推送功能需要内购解锁**，建议直接买断（一次性付费，价格不贵），这是目前 iOS 上最成熟稳定的 Git 客户端。

### 6.2 在 Working Copy 里生成 SSH 密钥

1. 打开 Working Copy，进入设置（齿轮图标）→ **SSH Keys** → 点 **+** 新建密钥，类型选默认的 Ed25519。
2. 生成后点开这把密钥，点击 **复制公钥（Copy Public Key）**。
3. 浏览器登录 GitHub → `Settings` → `SSH and GPG keys` → `New SSH key`：
    - Title 填 `iPad Working Copy`
    - Key type 选 `Authentication Key`
    - Key 一栏粘贴公钥
    - 保存

### 6.3 克隆仓库

1. 回到 Working Copy 主界面，点右上角 `+` → `Clone repository`。
2. 地址栏填写你仓库的 **SSH 443 通道地址**（跟 Windows 端同一个思路，直接用这个更稳，不用先试普通地址再切换）：

```
ssh://git@ssh.github.com:443/你的用户名/obsidian_vault.git
```

> 如果 Working Copy 的地址栏格式识别不了上面这种写法，改用标准 SSH 格式 `git@github.com:你的用户名/obsidian_vault.git`，SSH 密钥选择刚才生成的那一把。如果点击克隆后出现 `failed to start ssh session failed getting banner` 这种网络层报错，说明当前网络屏蔽了标准 22 端口，换成上面 443 通道的写法，或者切换网络（WiFi 换热点）重试。

3. **协议**：SSH；**用户**：git；**主机**：github.com（或 ssh.github.com，取决于你用哪种地址写法）；**端口**：留空使用默认值。
4. **SSH 密钥**选择刚才新建的那把（或者保持"自动"，通常也能匹配上）。
5. 点击右上角**克隆**，等待进度条走完。

克隆成功后，你会在 Working Copy 里看到一个和仓库同名的文件夹（比如 `obsidian_vault`），里面是你所有的笔记文件。

---

## 七、iPad 端核心难题：iOS 沙盒限制与 Folder Sync 解决方案

这是整个 iPad 配置里**最容易卡住、也是最关键**的一步，务必仔细看完。

### 7.1 为什么 Obsidian 打不开 Working Copy 里的文件夹

iOS/iPadOS 系统出于安全考虑，**不允许任何 App 随意访问其他 App 的沙盒空间**。Obsidian 在 iOS 上只能在两个"法定领地"里创建和读写库：

1. iCloud 云盘里的 `iCloud/Obsidian/` 文件夹
2. iPad 本地的 `我的 iPad/Obsidian/` 文件夹

而 Working Copy 克隆下来的仓库属于它自己的沙盒空间，Obsidian 无法跨越沙盒直接把它"打开"成库——这也是为什么很多人发现 Obsidian 的欢迎界面上根本没有"打开文件夹作为库"这个选项（这是 Obsidian 在 iOS 上一个长期存在的已知问题，官方论坛上有多篇类似反馈）。

### 7.2 解决方案：Working Copy 的 Setup Folder Sync

思路反过来：既然 Obsidian 进不去 Working Copy 的沙盒，就让 **Working Copy 把仓库内容"复制同步"到 Obsidian 自己能访问的本地文件夹里**。

#### 第一步：在 Obsidian 里新建一个本地空库

1. 打开 iPad 上的 Obsidian，进入库切换界面（点击当前库名弹出切换列表，或者首次启动看到的欢迎页）。
2. 点击 **Create new vault**（创建新库）。
3. **关闭 "Store in iCloud" 开关**（选择存在 iPad 本地，而不是 iCloud，这一点很关键，选错了以后没法改）。
4. 库名填你仓库对应的名字，比如 `obsidian_vault`，方便识别。
5. 点击 **Create**，Obsidian 会在本地创建一个干净的空库文件夹。
6. 退出这个库（不用管它，后面会被填满内容）。

#### 第二步：在 Working Copy 里设置 Folder Sync

1. 打开 Working Copy，点进你之前克隆好的仓库。
2. **点击屏幕顶部、居中位置的仓库名字本身**（不是返回箭头，不是右上角的功能图标，是标题文字），会弹出一个菜单。
3. 在这个菜单里找到 **"Setup Folder Sync"**（设置文件夹同步）。
    - 如果你的仓库文件夹在 iOS 系统看来是"文档包（document package）"类型、无法被选中，改用旁边的 **"Setup Package Sync"** 选项，效果一样。
4. 点击后会弹出系统文件选择器，导航到：**我的 iPad → Obsidian → 你刚才新建的那个空库文件夹**（比如 `obsidian_vault`）。
5. 选中这个文件夹，确认。
6. Working Copy 会提示建立同步关系，首次同步会把仓库里的所有笔记内容写入这个 Obsidian 文件夹。

#### 第三步：打开 Obsidian 验证

重新打开 iPad 上的 Obsidian，进入刚才新建的那个库——原本空的库现在应该已经出现你所有的笔记、文件夹了，跟 Windows 端内容完全一致。

---

## 八、日常使用工作流

Git 不是实时同步，养成固定的"先同步、再编辑"的习惯是避免冲突最有效的办法。

### 8.1 在 Windows 上

- 因为开了 **Pull on startup**，打开 Obsidian 时会自动拉取一次最新内容，正常写就行。
- 插件每 10 分钟自动提交并推送一次，不需要手动操作。
- 想立刻同步的话，命令面板执行 `Git: Commit and push`。

### 8.2 在 iPad 上（顺序不能错）

**写笔记之前：**

1. 打开 **Working Copy**，进入仓库，点击 **Pull**，拉取 Windows 端最新推送的内容。
2. 因为设置了 Folder Sync，拉取后的内容会自动同步进 Obsidian 对应的本地库文件夹。
3. 确认无冲突提示后，打开 **Obsidian** 开始编辑。

**写完笔记之后：**

1. 回到 **Working Copy**，Folder Sync 会检测到 Obsidian 库里的文件变化并同步回仓库文件夹。
2. 进入仓库，能看到修改的文件列表，确认无误后点击 **Commit**，填写一句提交说明。
3. 点击 **Push**，推送到 GitHub。

**核心原则**：iPad 端"编辑前先 Pull、编辑后立刻 Commit + Push"，Windows 端交给插件自动处理。真正容易出问题的场景是——iPad 上编辑了很久都没推送，同时 Windows 上又改了同一篇笔记，这种"双端离线并发修改"才会导致合并冲突。

---

## 九、`.gitignore` 配置

Obsidian 库里有些内容不需要同步（比如每台设备各自的窗口布局状态），避免多余的提交干扰、也避免两端互相覆盖界面设置。

在 Windows 库根目录新建 `.gitignore` 文件：

```gitignore
# Obsidian 工作区状态（每台设备的窗口布局、最近打开的文件等，不需要同步）
.obsidian/workspace.json
.obsidian/workspace-mobile.json

# 插件产生的本地缓存
.obsidian/plugins/*/data.json.bak

# 系统垃圾文件
.DS_Store
Thumbs.db

# 如果不想同步被删除笔记的历史
.trash/
```

写好后执行：

```powershell
git rm -r --cached .obsidian/workspace.json
git add .gitignore
git commit -m "添加 gitignore 忽略工作区状态文件"
git push
```

---

## 十、常见报错与排查手册

### 10.1 `git: 'remote-https' is not a git command`

Git 安装不完整或者装到了非默认路径。解决：卸载 → 删除残留文件夹 → 重装到默认路径 `C:\Program Files\Git` → 重启电脑 → 新开窗口验证 `where.exe git`。

### 10.2 `Failed to connect to github.com:443`（时好时坏）

网络层问题，跟配置无关。直接用本文第四节的方法，切换到 SSH over 443 通道（`ssh://git@ssh.github.com:443/...`），一次性解决，比反复重试或者临时开关代理更省心。

### 10.3 `failed to start ssh session failed getting banner`（iPad 端）

标准 22 端口的 SSH 被当前网络屏蔽。换成 443 通道地址，或者切换网络（WiFi 换手机热点）重试。

### 10.4 `error: remote origin already exists`

说明之前已经 `git remote add` 过一次了，不需要重复添加，直接跳过这条命令继续下一步即可。

### 10.5 `! [rejected] main -> main (fetch first)`

远程仓库上有本地没有的内容（常见于两端都推送过、或者网页端有过操作）。解决：

```powershell
git pull origin main --allow-unrelated-histories
```

如果弹出 Vim 编辑器界面（大片文字加 `~`）要求确认合并提交信息：按 `Esc`，输入 `:wq`，回车保存退出即可（或者按两下 `Shift+Z` 快捷键）。之后再执行一次 `git push -u origin main`。

如果提示有冲突（conflict），需要手动打开冲突文件，删掉 `<<<<<<<`、`=======`、`>>>>>>>` 标记，保留想要的内容，再 `git add . && git commit -m "解决冲突" && git push`。

### 10.6 误操作在错误的目录执行了 `git init`

比如把上一级目录（如 `Documents`）误初始化成了 Git 仓库。解决：`cd` 到那个错误的父目录，执行 `Remove-Item -Recurse -Force .git` 删除误建的隐藏文件夹（只删 `.git`，不影响任何实际笔记文件），再 `cd` 到正确的库文件夹重新走一遍初始化流程。

### 10.7 iPad 上 Obsidian 编辑的内容"消失"了

最常见原因：编辑前忘了先在 Working Copy 里 Pull，导致基于旧版本编辑，随后同步时的合并把改动覆盖掉了。**严格执行"编辑前先 Pull"的顺序**可以完全避免。

