# FeedlyReading

> 一个 Claude Code Skill，自动筛选 Feedly 订阅文章——相关文章存入"稍后阅读"，其余全部标记为已读。

---

## 它能做什么

1. 读取 Feedly `paper` 文件夹下所有订阅源的未读文章（~1000+ 篇也能处理）
2. 逐篇判断标题相关性（关键词匹配 + 语义主题判断）
3. 相关 → 保存到 Feedly **Read Later（稍后阅读）**
4. 其余 → 批量标记为已读，清空 Unread 计数
5. 全流程约 2–3 分钟，输出处理摘要

**默认过滤条件**（可自定义）：

| 类型 | 内容 |
|------|------|
| 关键词 | navigation、stress、prior、working memory、EEG（及中文对应词） |
| 语义主题 | 空间导航（spatial navigation、place cells、grid cells 等）、先验知识（Bayesian prior、predictive coding 等） |

---

## 环境要求

| 依赖 | 说明 |
|------|------|
| [Claude Code](https://claude.ai/code) | Desktop 版或 VS Code 扩展 |
| [Claude in Chrome 扩展](https://chromewebstore.google.com/detail/claude-in-chrome/...）| 用于自动提取 Feedly token |
| Chrome + Feedly | 已登录 feedly.com |
| PowerShell | Windows 内置；macOS/Linux 用户可将脚本改写为 `curl` |
| Feedly 账号 | 免费账号即可；订阅列表位于名为 `paper` 的文件夹下 |

---

## 安装

```bash
git clone https://github.com/YOUR_USERNAME/FeedlyReading.git
cd FeedlyReading
claude .   # 用 Claude Code 打开
```

Claude Code 会自动识别 `.claude/skills/` 下的 Skill，无需额外配置。

---

## 首次使用：获取你的 userId

Token 由 Skill 自动提取，**userId 需要手动查一次**（之后不会变）：

1. 在 Chrome 中打开 feedly.com，确保已登录
2. 按 `F12` 打开 DevTools → Console 标签
3. 执行以下命令，复制输出的字符串：

```javascript
JSON.parse(localStorage.getItem('feedly.session')).id
```

4. 在 CLAUDE.md 的 PowerShell 模板中，将 `YOUR_USER_ID` 替换为你的实际值。

> Token 的提取方式相同，将 `.id` 改为 `.feedlyToken`。但 **Token 有时效**，Skill 会在每次执行时自动提取，无需手动操作。

---

## 使用方法

在 Claude Code 终端输入：

```
/Feedly文章筛选
```

### 常用变体

| 命令 | 说明 |
|------|------|
| `/Feedly文章筛选` | 处理 paper 下全部子源，使用默认过滤条件 |
| `/Feedly文章筛选 子源=Nature Neuroscience` | 只处理指定子源 |
| `/Feedly文章筛选 只保留与记忆巩固相关的文章` | 全部子源 + 自定义过滤条件（替换默认条件） |
| `/Feedly文章筛选 子源=bioRxiv 关键词：attention, reward` | 指定子源 + 自定义过滤条件 |

---

## 自定义过滤条件

传入自然语言描述即可，无需特定格式。自定义条件**完全替换**默认条件：

```
/Feedly文章筛选 关键词：attention, reward, dopamine；主题：强化学习与神经机制
```

---

## 工作原理

```
Chrome（已登录 Feedly）
        ↓
javascript_tool 提取 localStorage.feedlyToken + .id
        ↓
PowerShell: GET /v3/subscriptions → 筛出 paper 文件夹订阅列表
        ↓
PowerShell: GET /v3/streams/contents（分页，支持 1000+ 篇）
        ↓
Claude 批量评估标题相关性（关键词 + 语义）
        ↓
PowerShell: PUT /v3/tags/global.later（保存相关文章）
        ↓
PowerShell: POST /v3/markers markAsRead（清空全部未读）
        ↓
输出摘要
```

API 不可用时（token 失效等）自动降级为 Chrome UI 操作。

---

## 已知误匹配

| 关键词 | 误匹配来源 | 建议 |
|--------|-----------|------|
| `stress` | DNA拓扑压力、热应激、病原体感染等非神经科学论文 | 处理完后在 Read Later 快速扫一遍 |
| `prior` | 计算机视觉中的形状先验（pose estimation 等） | 同上 |

---

## 项目结构

```
FeedlyReading/
├── .claude/
│   ├── settings.json              # Claude in Chrome 工具权限（自动授权）
│   └── skills/
│       └── Feedly文章筛选/
│           └── SKILL.md           # Skill 定义（Claude Code 自动加载）
├── .gitignore                     # 排除 settings.local.json（含个人 token）
├── CLAUDE.md                      # 项目规范（API 模板、已知问题等）
└── README.md                      # 本文件
```

---

## 常见问题

**Q: 执行时提示找不到 Feedly 标签页？**  
确认 Chrome 中已打开 feedly.com，且 Claude in Chrome 扩展已激活（工具栏图标亮起）。

**Q: PowerShell 报 401 Unauthorized？**  
Token 已失效，重新从 DevTools Console 提取：`JSON.parse(localStorage.getItem('feedly.session')).feedlyToken`

**Q: 我的订阅不在 `paper` 文件夹下怎么办？**  
修改 SKILL.md 中的 `$_.label -eq "paper"` 为你的文件夹名称。

**Q: macOS / Linux 可以用吗？**  
Skill 中的 PowerShell 脚本需改写为 `curl` + `jq`；token 提取和相关性判断逻辑不变。

---

## License

MIT
