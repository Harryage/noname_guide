# Personal 仓库迁移 · 手动操作手册

> 生成时间：2026-10-03。配套主文档：《Obsidian × Git × GitHub：Windows 与 iPad 双端同步完整实战指南》（下称"主指南"），本文只写差异和步骤，原理不重复。
>
> 目标：用 `F:\Personal` 取代 `C:\Users\17738\Documents\Obsidian Vault`，对应新的 GitHub 私密仓库 `Harryage/personal`，iPad 端照常 Working Copy + Folder Sync 同步，最后删除旧链路。

---

## 〇、当前状态（已由阿枢完成，无需重做）

| 事项 | 状态 |
|---|---|
| `F:\Personal` git 初始化 | ✅ 已完成（分支 `main`，`autocrlf=false`） |
| remote 地址 | ✅ 已指向 `ssh://git@ssh.github.com:443/Harryage/personal.git` |
| `.gitignore` 初版 | ✅ 已写入（本文第二节有最终版，**建议直接覆盖**） |
| 8 个 `.tmp_*` 临时文件 | ✅ 已移到 `C:\Users\17738\.workbuddy\trash_20261003_FPersonal\`（未真删，可随时找回） |
| git 提交身份 | ✅ 全局已有（hurry / jia1773819885@gmail.com） |
| **首次 commit / push** | ❌ 未完成（此前 add 被 24GB 体量拖到超时，且无锁残留，等你手动做） |
| 旧 Vault 内容安全确认 | ✅ 旧 Vault 自 10-01 以来 git 只变动过两个窗口状态文件，无实质笔记更新，内容已全在 F:\Personal |

---

## 一、先决策：F:\Personal 有 24GB，不能整个推 GitHub

### 1.1 为什么不能整推

- **GitHub 单文件硬上限 100MB**：你有 31 个超过 90MB 的文件（15 个宏观课 mp4、8 个雅思视频/软件、2 本教材 PDF、3 本书籍礼包 PDF 等），**push 会被直接拒绝**。
- GitHub 仓库软性建议 ≤1–5GB，24GB 克隆极慢。
- iPad 端 Working Copy 的存储和流量都不现实，同步的对象应该是**笔记**，不是 24GB 资料库。旧 Vault 能跑通，恰恰因为它只有笔记。

### 1.2 顶层大小分布（实测）

| 文件夹 | 大小 | 性质 | 建议 |
|---|---|---|---|
| 宏观经济指标框架/ | **7.4 GB** | 课程视频 mp4 | 排除 |
| temper/ | **7.0 GB** | 雅思视频课+机考软件+书籍礼包+老笔记 | 排除大件，笔记保留 |
| 鸢尾花书/ | 691 MB | PDF 书 | 排除 |
| finacial/ | 485 MB | 笔记 + 2 本百 MB 级教材 PDF | 排除 PDF，笔记保留 |
| 书/ | 23 MB | 书 | 排除（若全是 PDF 自然被扩展名规则挡住） |
| 归档_原始文档_2026-10-01/ | 1 MB | 纯笔记（旧 Vault 原始归档） | **必须进 git** |
| 00~07 编号笔记（8 个 md） | 共 <8 MB | 核心笔记 | **必须进 git** |

处理完只剩**几 MB 级纯笔记仓库**，push 和 iPad 同步都是秒级。

### 1.3 最终版 .gitignore（直接覆盖 `F:\Personal\.gitignore`）

```gitignore
# ===== Obsidian 工作区状态（每台设备各自维护，不同步）=====
.obsidian/workspace.json
.obsidian/workspace-mobile.json

# 插件本地缓存
.obsidian/plugins/*/data.json.bak

# 系统垃圾文件
.DS_Store
Thumbs.db

# 被删除笔记的暂存区
.trash/

# WorkBuddy 助手工作数据（非笔记内容）
.workbuddy/

# 临时文件
.tmp_*

# ===== 大文件：视频 / 音频 / 软件安装包 / 压缩包（任何情况都不进 git）=====
*.mp4
*.m4v
*.wmv
*.avi
*.mkv
*.mov
*.mp3
*.flac
*.dmg
*.exe
*.msi
*.iso
*.zip
*.rar
*.7z

# ===== PDF 全部不入库（教材/书籍体积大、更新少）=====
# 如日后想在 iPad 上同步某本 PDF，单独加例外，例如：
# !finacial/conpany finacial/投资学（第十版）*.pdf
*.pdf
*.epub

# ===== 资料型文件夹整目录排除（防止漏网大文件拖慢 add）=====
宏观经济指标框架/
鸢尾花书/
书/
temper/雅思/
temper/行业研究/【05】金融干货书籍礼包/
```

> 说明：`宏观经济指标框架/` 整目录排除是因为 7.4GB 几乎全是课程视频；如果你在里面写过笔记 md 想同步到 iPad，就把这一行删掉，剩下的 md 会靠上面的扩展名规则正常入库（mp4 依然被挡住）。

### 1.4 完整的 90MB+ 大文件清单（31 个，供核对）

全部会被上面的规则自动排除，无需手动逐个处理：

- `finacial/conpany finacial/`：公司理财（罗斯）.pdf、投资学（博迪）.pdf
- `temper/行业研究/【05】金融干货书籍礼包/【01】金融干货书籍（30本）/12-管理经济学.pdf`，`【02】`下 37-市场营销原理.pdf、58-兼并、收购和公司重组.pdf
- `temper/雅思/`：口语示例P3-6.5.wmv、IELTS_Practice_Test.dmg、IELTS_Practice_Test_1.1.0.zip、小白课×3（口语/听力/阅读）.mp4、雅思7分备考×6（写作/口语/听力/阅读等）.mp4
- `宏观经济指标框架/`：topic1~topic5 共 6 个 mp4 + `基础版合集/video_宏-01~10` 共 10 个 mp4

---

## 二、Windows 端：完成首次提交与推送

打开 **PowerShell**（不要用管理员模式），逐条执行：

### 2.1 建仓库（GitHub 网页操作）

1. 浏览器登录 GitHub → 右上角 `+` → `New repository`
2. Repository name 填：`personal`（全小写，必须与 remote 一致）
3. 选 **Private**
4. **什么都不要勾**：不要 README、不要 .gitignore、不要 License，保持空仓库
5. `Create repository`

### 2.2 覆盖 .gitignore

用 1.3 节的内容覆盖 `F:\Personal\.gitignore`（记事本/VS Code 打开粘贴保存即可）。

### 2.3 提交与推送

```powershell
cd F:\Personal
git status --short          # 确认列表里没有 mp4/pdf/大文件夹（有就回到 1.3 检查 gitignore）
git add -A
git commit -m "初始化 Personal 库（自 obsidian_vault 迁移）"
git push -u origin main
```

> SSH 密钥是现成的（旧 Vault 用的就是本机 `~/.ssh/id_ed25519`），**不需要重新生成、不需要去 GitHub 添加新钥匙**。
> 如果 add 或 push 卡住/报错，直接查主指南第十节；`index.lock` 报错见本文第六节。

### 2.4 验证

```powershell
git log --oneline -1
git ls-files | Measure-Object -Line   # 跟踪文件数，正常应为几十到几百，绝不该上千
```

再去 GitHub 网页刷新 `personal` 仓库，能看到 00~07 笔记和归档文件夹即成功。

---

## 三、Windows 端 Obsidian：Git 插件

1. Obsidian 打开的是 `F:\Personal` 这个库（你已在用）。若当前开的是旧库，先切换。
2. 设置 → 第三方插件 → 浏览 → 搜 `Git`（作者 Vinzent03）→ 安装并启用。
3. 配置照抄主指南 5.2 表格：**Auto commit-and-sync 10 分钟、Pull on startup 开启**，其余默认。
4. 验证：`Ctrl+P` → 执行 `Git: Commit and push`，无报错即通。

---

## 四、iPad 端（流程与主指南第六、七节完全相同，只换地址）

1. **Working Copy**：右上角 `+` → Clone repository，地址填：

   ```
   ssh://git@ssh.github.com:443/Harryage/personal.git
   ```

   协议 SSH、用户 git，SSH 密钥用你当时生成的那把（不用新建）。
2. **Obsidian**：新建本地库（**务必关掉 Store in iCloud**），名字建议就叫 `Personal`，创建后退出。
3. **Folder Sync**：Working Copy 进入 `personal` 仓库 → 点击顶部居中的仓库名字 → `Setup Folder Sync` → 选中「我的 iPad → Obsidian → Personal」这个空库 → 首次同步完成。
4. **验证双端闭环**（顺序做）：
   - iPad：Obsidian 里随便建一篇测试笔记 → Working Copy 里 Commit → Push
   - Windows：Obsidian 命令面板 `Git: Pull`，能看到测试笔记
   - Windows 改一下这篇笔记 → 等自动同步或手动 push → iPad Working Copy Pull → 内容更新
   - 都通了，删掉测试笔记再各推一次

---

## 五、删除旧链路（**全部验证通过后**再执行）

### 5.1 Windows：本地旧 Vault

1. **先关掉 Windows 端 Obsidian 里的旧库**（或直接退出 Obsidian）——旧库还挂着 obsidian-git 插件，每 10 分钟会自动对旧文件夹做 commit，不先关会一直报错。
2. 资源管理器进入 `C:\Users\17738\Documents\`，选中 `Obsidian Vault` 文件夹 → **Delete（进回收站，不要 Shift+Delete）**。回收站保留几天，确认新链路无问题后再清空。

### 5.2 GitHub：删除 obsidian_vault 仓库

1. 打开 `https://github.com/Harryage/obsidian_vault`
2. `Settings` → 拉到最底部 **Danger Zone** → `Delete this repository`
3. 按提示输入 `Harryage/obsidian_vault` 确认删除

### 5.3 iPad：清掉旧痕迹

1. Working Copy：左滑旧的 `obsidian_vault` 仓库 → Delete
2. Obsidian：库切换界面 → 旧的本地库 → 删除

---

## 六、报错速查（本文档特有 + 主指南补充）

| 报错 | 原因与解法 |
|---|---|
| `Unable to create '.git/index.lock': File exists` | 上一次 git 操作被中断留下的锁。确认没有 git 窗口在跑后：`del F:\Personal\.git\index.lock` 再重试 |
| `remote: Invalid username or password` | 不可能出现——本仓库走 SSH。若看到说明 remote 配错，`git remote -v` 核对 |
| push 被拒 `pre-receive hook declined` / 提示文件超过 100MB | gitignore 没生效，某个大文件已被 add。`git rm --cached "路径"` 后重新 commit |
| push 被拒 `(fetch first)` | 远程不为空（建仓库时勾了 README）。`git pull origin main --allow-unrelated-histories` 后再 push，或干脆删掉刚建的远程仓库重建成空仓库 |
| 中文文件名显示成转义数字 | 正常现象，`git config --global core.quotepath false` 可让 `git status` 显示中文原文 |
| iPad clone 报 `failed getting banner` | 网络屏蔽 22 端口，确认用的是 443 通道地址，或切手机热点（主指南 10.3） |
