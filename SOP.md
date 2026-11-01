# 读书流水线 SOP

> 一条从「找书」到「建成个人书库」的完整流程。逐章精读交给 AI 子代理，主对话只收讲解；读完整理成四件套，导入 FlowUs 做自己的书库。

---

## 0. 一次性准备

1. **装 zlib**（Z-Library 命令行）：
   ```bash
   brew install heartleo/tap/zlib   # macOS / Linux
   # 或：curl -fsSL https://raw.githubusercontent.com/heartleo/zlib/main/install.sh | sh
   ```
2. **登录 Z-Library**（在自己终端里做，密码不要发进 AI 对话）：
   ```bash
   zlib doctor                    # 找可用域名
   zlib login --domain https://z-lib.gd
   zlib profile --json            # 确认已登录
   ```
3. **装深度阅读插件**：
   ```bash
   dsh plugin --profile <profile> add github:Madao1210/dsh-deep-reading
   ```
   插件会带来 `dr_import / dr_read / dr_write / dr_status / dr_usage / dr_export_epub / reading_chapter` 七个工具，以及 `deep-reading` 技能。
4. **装 pnpm**（`dsh plugin` 依赖）：
   ```bash
   npm install -g pnpm
   ```
5. **准备 Chrome**（渲染思维导图用，macOS 自带路径见后）。

---

## 1. 找书与选版本

```bash
zlib search "书名 作者" --json --count 20
```

- 解析 JSON 的 `books[].name / authors / year / extension / size / id`，列成表格让用户选。
- 规则：**epub 优先于 pdf**（pdf 必须有文字层，扫描版会被拒）；中文书选 epub 最省事。
- 同一本书多版本时先问用户；用户只说"搜"就只展示，不要擅自下载。

## 2. 下载

```bash
zlib download "<id:hash>" --json
```

- `id` 保留完整 `数字:hash` 两段。
- 从 stdout 的 JSON 取 `path`，作为下一步导入的 `src`。
- 退出码：`3`=需重新登录；`4`=网络/镜像问题（跑 `zlib doctor`）；`5`=书已存但 Kindle 投递失败。

## 3. 导入拆章

```
dr_import(src="<绝对路径>", book="<kebab-id>", title="<书名>")
```

- `book` 用简短拼音/英文，如 `zhi-shen-shi-nei`，之后所有工具都用它。
- 导入报告只说**书名和章数**，不转述正文。
- EPUB 按 spine 拆章；PDF/txt 先转 Markdown（转换全文留 `books/<ID>/source.md`）。

## 4. 逐章精读

先看状态，再逐章读：

```
dr_status(book="<ID>")
reading_chapter(book="<ID>", chapter=N, purpose="<阅读目的>")
```

- **原文只在子代理里读**，主对话只收「讲解 + 状态行」；绝不要用 `dr_read` 把整章拉进主对话。
- `reusable=true` 的章**不重读**，直接基于已有记忆讲解，注明"（基于已有记忆，未重读原文）"。
- `state=partial` 说明本章没读完，从 `covered` 断点续读。
- 章节几十上百章时，**只精读主干章**（各章正文小节），跳过目录、版权页、参考文献、"扩展阅读"书目；在笔记开头写明跳过了哪些。

## 5. 续读与复用

- 新会话说「继续」：`dr_status` 全书扫一遍，从第一个 `reusable=false` 的章接着读。
- 换视角重讲已读章节，直接复用记忆，必要时 `dr_read(mode="locate")` 定位原文短句核对。

## 6. 产出四件套

读完后在 `读书/<ID>/`（或用户指定目录）产出四件套：

| 文件 | 内容 | 来源 |
|---|---|---|
| `00 大纲.md` | 全书主线、结构图、逐章大纲、论证链、作者立场与局限 | 各章记忆 |
| `01 读书笔记.md` | 逐章笔记（机制 + 证据 + 我的看法）、跨章线索、金句、读后总结 | 讲解 + 记忆 |
| `02 知识卡片.md` | 30–40 张可复用卡片（概念/一句话/机制/易混/出处） | 记忆提炼 |
| `03 思维导图.png` | 核心命题 + 各章一句话 + 全书主线 | 大纲 |

**思维导图渲染**（HTML + Chrome 无头截图，不依赖 Python 绘图库）：

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --no-sandbox --hide-scrollbars --no-first-run \
  --user-data-dir=/tmp/dsh-chrome --force-device-scale-factor=2 \
  --window-size=2000,1080 --screenshot="03 思维导图.png" "file://$PWD/mindmap.html"
```

- HTML 用固定像素画布（如 2000×1080），字体写 `"PingFang SC", "Hiragino Sans GB", sans-serif`。
- 渲染后**必须**用 `read_image` 检查：是否溢出、是否留大片空白；尺寸不对就调画布高度重截。

## 7. 落盘与同步

把四件套同步到用户指定位置（默认 `~/Downloads/<书名>-拆解/`）：

```bash
B="$HOME/.dsh/deep-reading/books/<ID>"
mkdir -p "读书/<ID>"
for f in "$B"/lecture/*.md; do cp "$f" "读书/<ID>/讲解-$(basename "$f")"; done
for f in "$B"/memory/*.md;  do cp "$f" "读书/<ID>/章节记忆-$(basename "$f")"; done
```

- 只复制讲解/记忆/四件套；`source.md`、`chapters/`、`reader.md` 是内部数据不搬。
- 复制是单向快照，书库仍是唯一真相。

## 8. 建 FlowUs 书库

```
📚 我的书库（主页）
├── 数据库「书籍总表」：书名/作者/状态/评分/标签/链接
└── 每本书详情页
    ├── 封面 + 基本信息
    ├── 00 大纲（子页）
    ├── 01 读书笔记（子页）
    ├── 02 知识卡片（子页，或做成"卡片数据库"：主题/出处 两个字段）
    └── 03 思维导图（PNG 图片块）
```

- FlowUs 支持粘贴 Markdown / 导入文件，标题、列表、表格自动识别。
- 卡片最值得做成数据库：每条一行 + 「主题 / 出处章节」字段，方便筛选和跨书串联。

---

## 故障排查

| 症状 | 处理 |
|---|---|
| `subagents.start is not a function` | DSH 0.2.1+ 把 subagent API 换成 `startActivation`；用 `pnpm patch dsh-deep-reading` 适配（把 `subagents.start('spawn', {...})` 改成 `subagents.startActivation({provider:'spawn', request:{...}, signal, delivery:'caller'})`，并把章节技能拼进 prompt），改完 `pnpm patch-commit` 再触发 HMR 或重启 |
| 扫描版 PDF 被拒 | 提示先 OCR；不要反复重试 |
| zlib 退出码 3 | 让用户在自己的终端 `zlib login`，不要把密码要进对话 |
| zlib 退出码 4 | `zlib doctor` 找可用镜像 |
| 中文 PDF 抽出 0 字 | 无文字层（扫描件），需 OCR |
| 改了 `source.md`/`chapters` 后记忆失效 | 章节哈希变了，`dr_status` 会判过期；要拆章就趁没读之前 |

---

## 版权边界（重要）

- **方法、模板、SOP**：你自己的原创，可开源、可授权、可出售。
- **书籍拆解产物**：是原书的衍生内容（含大量转述与直接引语），**只适合自用**，不要公开售卖或随方法开源。
- 想卖"方法"就卖 SOP + 模板；想卖"内容"就用公版书或自己原创，或做会员社群/带读服务。
- 正式商业化前，涉及有版权书的内容请咨询版权律师。

## 工具速查

| 工具 | 作用 |
|---|---|
| `zlib search/download/profile/doctor` | 找书、下载、账户、镜像 |
| `dr_import` | 导入 EPUB/md/txt/pdf，拆章建索引 |
| `dr_status` | 每章记忆状态与 `reusable` 判定 |
| `dr_read` | 只读检索：index / range / locate / find |
| `dr_write` | 写 memory / lecture / user-note / reader / export |
| `reading_chapter` | 派生章节子代理精读一章 |
| `dr_usage` | 用量记账 / 汇总 |
| `dr_export_epub` | 把讲解拼成 EPUB |
