# FeedlyReading

## 项目简介

Feedly 文章自动筛选工具。**优先使用 Feedly REST API（PowerShell）** 处理全部订阅源，速度快、稳定，不依赖 Chrome UI。仅当 API 不可用时（token 失效、网络异常）退回 Claude in Chrome UI 操作。

## 工作流

```
Chrome 打开 Feedly（已登录）
     ↓
从 localStorage 提取 token（一次性，见下方步骤）
     ↓
/Feedly文章筛选 [子源=XX] [过滤条件]
     ↓
PowerShell: 拉取 paper 下全部子源的未读文章
     ↓
批量判断标题相关性 → 保存相关文章到 Read Later
     ↓
POST markAsRead 清空全部子源
     ↓
输出处理摘要
```

## 可用命令

| 命令 | 参数 | 说明 |
|------|------|------|
| `/Feedly文章筛选` | `[子源=XX] [过滤条件]` | 读取 paper 下指定子源并按条件筛选；未指定子源则处理全部；不传过滤条件则使用默认条件 |

### 默认过滤条件

**关键词**（不区分大小写）：navigation、stress、prior、working memory、EEG

**语义主题**：空间导航（spatial navigation）、先验知识（prior knowledge）

## Skill

| Skill | 路径 | 触发命令 |
|-------|------|---------|
| Feedly 文章筛选 | `.claude/skills/Feedly文章筛选/SKILL.md` | `/Feedly文章筛选` |

同时注册为全局插件：`feedlyreading-skills@local-desktop-app-uploads`

## GitHub 仓库

| 仓库 | URL | 说明 |
|------|-----|------|
| 主项目 | https://github.com/CrisChenYingyan/FeedlyReading | 完整项目，含 CLAUDE.md、SKILL.md、README、settings.json |
| Skill 独立包 | https://github.com/CrisChenYingyan/FeedlyReadingSkill | 仅含 Skill，供他人直接安装使用 |

## Feedly API 操作方式（首选）

### Token 提取步骤

每次 token 失效后重新提取（token 有效期约数天）：

1. 在 Chrome 中打开 Feedly，确保已登录
2. 打开 DevTools（F12）→ Console，执行：
   ```javascript
   JSON.parse(localStorage.getItem('feedly.session')).feedlyToken
   ```
3. 复制输出的完整 JWT 字符串（以 `eyJ` 开头）
4. 在 PowerShell 脚本中赋值给 `$token`

**userId**：每位用户不同，通过以下命令获取：
```javascript
JSON.parse(localStorage.getItem('feedly.session')).id
```
（在 Chrome DevTools Console 中执行，与 token 提取步骤相同）

### 核心 API 端点

| 操作 | 方法 | URL |
|------|------|-----|
| 获取订阅列表 | GET | `https://feedly.com/v3/subscriptions` |
| 获取未读文章 | GET | `https://feedly.com/v3/streams/contents?streamId={encoded}&count=250&unreadOnly=true` |
| 保存稍后阅读 | PUT | `https://feedly.com/v3/tags/user%2F{userId}%2Ftag%2Fglobal.later` body: `{"entryIds":[...]}` |
| 标记订阅为已读 | POST | `https://feedly.com/v3/markers` body: `{"action":"markAsRead","type":"feeds","feedIds":[...],"asOf":<ms>}` |

**注意**：`/v3/markers/counts` 不可靠（始终返回 0），不要用它判断文章是否存在，改用实际拉取 stream contents。

### 执行模板（PowerShell）

```powershell
$token  = "eyJ..."   # 从 localStorage.feedlyToken 提取
$userId = "YOUR_USER_ID"   # 从 localStorage.id 提取
$headers = @{ Authorization = "OAuth $token"; "Content-Type" = "application/json" }

# 1. 获取 paper 文件夹下所有订阅
$subs   = Invoke-RestMethod "https://feedly.com/v3/subscriptions" -Headers $headers
$paper  = $subs | Where-Object { $_.categories | Where-Object { $_.label -eq "paper" } }

# 2. 拉取各订阅未读文章，写入临时文件
$allData = @()
foreach ($feed in $paper) {
    $encoded = [Uri]::EscapeDataString($feed.id)
    $items = @(); $cont = $null
    do {
        $url = "https://feedly.com/v3/streams/contents?streamId=$encoded&count=250&unreadOnly=true"
        if ($cont) { $url += "&continuation=$cont" }
        $r = Invoke-RestMethod $url -Headers $headers
        $items += $r.items; $cont = $r.continuation
    } while ($cont -and $items.Count -lt 500)
    if ($items.Count -gt 0) {
        $allData += [PSCustomObject]@{ FeedTitle=$feed.title; FeedId=$feed.id; Items=$items }
    }
}
$allData | ConvertTo-Json -Depth 5 | Out-File "$env:TEMP\feedly_unread.json" -Encoding utf8

# 3. 关键词/语义匹配（见 SKILL.md 默认条件）
# → 得到 $relevantIds 列表

# 4. 保存稍后阅读（每批 100 条）
$tagUrl = "https://feedly.com/v3/tags/user%2F$userId%2Ftag%2Fglobal.later"
for ($i=0; $i -lt $relevantIds.Count; $i+=100) {
    $batch = $relevantIds[$i..([Math]::Min($i+99,$relevantIds.Count-1))]
    $body  = @{entryIds=$batch} | ConvertTo-Json -Compress
    Invoke-RestMethod $tagUrl -Method PUT -Headers $headers -Body $body | Out-Null
}

# 5. 全部订阅标记为已读
$asOf = [DateTimeOffset]::UtcNow.ToUnixTimeMilliseconds()
foreach ($feed in $allData) {
    $body = @{action="markAsRead";type="feeds";feedIds=@($feed.FeedId);asOf=$asOf} | ConvertTo-Json -Compress
    Invoke-RestMethod "https://feedly.com/v3/markers" -Method POST -Headers $headers -Body $body | Out-Null
}
```

### 已知误匹配关键词

| 关键词 | 误匹配来源 | 说明 |
|--------|-----------|------|
| `\bstress\b` | DNA拓扑压力、热应激、病原体应激、青光眼神经应激 | 偶有非神经科学文章命中，需人工核查 |
| `\bpriors?\b` | 计算机视觉（6D pose estimation 等） | "prior" 在 CV 中指形状先验，非贝叶斯先验 |

建议在 Feedly Read Later 中快速扫描一遍，剔除误标文章。

## 运行规范

### 前提条件

执行 `/Feedly文章筛选` 前，确认：
- Chrome 已打开 Feedly 且已登录（需要时提取 token）
- PowerShell 可访问 feedly.com（无网络限制）

### 最小改动原则

修改 Skill 或配置时：
- 只改动与当前需求直接相关的部分
- 改动前说明"改了什么、为什么改"

### 执行策略

**API 方式（默认）**：
1. 提取 token → 一次性拉取全部 paper 订阅源未读文章存入临时 JSON
2. 批量评估标题 → PUT 保存相关文章 → POST 标记全部已读
3. 全流程 2-3 分钟完成，无需 UI 逐条操作

**UI 方式（降级备用，仅当 API 不可用时）**：
1. 单篇验证：取第一篇走完完整操作，确认按钮响应正常后再批量处理
2. 根据剩余算例动态分配，每子源完成后输出阶段性摘要
3. 算例不足时主动停止，告知用户剩余未处理子源

### 运行后询问

**每次执行命令结束后**，都询问用户：
> "是否需要更新 CLAUDE.md？（例如：新增规范、修正描述、记录本次发现的问题）"
