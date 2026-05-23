---
name: Feedly文章筛选
description: 自动筛选 Feedly paper 文件夹下的未读文章：优先调用 Feedly REST API（PowerShell），批量拉取标题 → 判断相关性 → 保存到"稍后阅读" → 标记全部已读。API 不可用时退回 Claude in Chrome UI 操作。触发命令：`/Feedly文章筛选 [子源=XX] [过滤条件]`，不传参时使用默认过滤条件（关键词：navigation / stress / prior / working memory / EEG；语义主题：空间导航、先验知识）并处理全部子源。
---

# Feedly 文章筛选

当用户输入 `/Feedly文章筛选 [子源=XX] [过滤条件]` 时执行。

**前提条件**：Chrome 浏览器已打开 feedly.com 且已登录，Claude in Chrome 插件已激活。

---

## 过滤条件

### 默认条件（不传参时始终使用）

**关键词**（不区分大小写，中英文均适用）：

- navigation / 导航
- stress / 压力
- prior / 先验
- working memory / 工作记忆
- EEG / 脑电图

**语义主题**（标题即使不含上述关键词，但语义与以下主题高度相关也判为"相关"）：

- 空间导航（spatial navigation、place cells、grid cells、海马体空间表征等）
- 先验知识（prior knowledge、Bayesian prior、top-down expectation、predictive coding 等）

### 自定义条件

用户传入的过滤内容**完全替换**默认条件，作为本次的过滤逻辑。格式不限，自然语言即可：

- `/Feedly文章筛选 只保留与大脑记忆巩固相关的文章`
- `/Feedly文章筛选 关键词：attention, reward, dopamine；主题：强化学习与神经机制`

---

## 目标子源

命令参数中可通过 `子源=<名称>` 指定 paper 下的某一个子源：

- **已指定**：只处理该子源，跳过其他所有子源
- **未指定**（默认）：按顺序处理 paper 下的**全部**子源

---

## 执行步骤（API 优先）

### 步骤 0：提取 Token 与 userId

使用 `javascript_tool` 从已打开的 Feedly 标签页提取认证信息：

```javascript
const s = JSON.parse(localStorage.getItem('feedly.session'));
JSON.stringify({ token: s.feedlyToken, userId: s.id });
```

- 若成功获取 token 和 userId，进入 **API 方式**（步骤 1A）
- 若失败（页面未登录或 localStorage 无数据），进入 **UI 降级方式**（步骤 1B）

---

### 步骤 1：确认条件并报告

在开始前输出本次过滤条件及目标子源，格式：

```
【本次筛选条件】
关键词：navigation, stress, prior, working memory, EEG（含中文对应词）
语义主题：空间导航、先验知识
目标子源：全部（或：<子源名称>）
执行方式：API（或：UI 降级）
```

---

## API 方式（步骤 2A–6A）

### 步骤 2A：获取 paper 文件夹订阅列表（PowerShell）

```powershell
$token   = "<从步骤0提取>"
$userId  = "<从步骤0提取>"
$headers = @{ Authorization = "OAuth $token"; "Content-Type" = "application/json" }

$subs  = Invoke-RestMethod "https://feedly.com/v3/subscriptions" -Headers $headers
$paper = $subs | Where-Object { $_.categories | Where-Object { $_.label -eq "paper" } }
Write-Host "paper 文件夹共 $($paper.Count) 个子源"
$paper | Select-Object title, id | Format-Table -AutoSize
```

根据 `子源=XX` 参数决定处理范围（全部或指定子源）。

### 步骤 3A：拉取全部未读文章（PowerShell）

```powershell
$allData = @()
foreach ($feed in $targetFeeds) {
    $encoded = [Uri]::EscapeDataString($feed.id)
    $items = @(); $cont = $null
    do {
        $url = "https://feedly.com/v3/streams/contents?streamId=$encoded&count=250&unreadOnly=true"
        if ($cont) { $url += "&continuation=$cont" }
        $r    = Invoke-RestMethod $url -Headers $headers
        $items += $r.items
        $cont  = $r.continuation
    } while ($cont -and $items.Count -lt 500)
    if ($items.Count -gt 0) {
        $allData += [PSCustomObject]@{ FeedTitle=$feed.title; FeedId=$feed.id; Items=$items }
        Write-Host "  $($feed.title): $($items.Count) 篇"
    }
}
$allData | ConvertTo-Json -Depth 5 | Out-File "$env:TEMP\feedly_unread.json" -Encoding utf8
```

**注意**：`/v3/markers/counts` 端点不可靠（始终返回 0），不用于判断是否有未读文章，改用实际拉取 stream contents。

### 步骤 4A：评估标题相关性

对临时 JSON 中所有文章标题执行关键词 + 语义过滤：

```powershell
$kwPatterns = @(
    'EEG|electroencephalograph',
    '\bnavigation\b',
    '\bstress\b',
    'working memory',
    '\bpriors?\b',
    'predictive coding',
    'predictive processing'
)
$semanticPatterns = @(
    'grid map', 'entorhinal spatial map', 'septo.?entorhinal',
    'hippocampal ripples', 'place cell', 'grid cell',
    'hippocampal CA3.*sequence', 'hippocampal theta',
    'predictive map.*entorhinal|entorhinal.*predictive map',
    'cortical knowledge structure', 'vestibular heading',
    'hippocampal.retrosplenial', 'hippocampal sequences',
    'sticky.beliefs|belief stickiness', 'prediction error',
    'alpha.*working memory|working memory.*alpha', 'alpha rhythm',
    'remapping.*hippocampal|hippocampal.*remapping'
)

$data = Get-Content "$env:TEMP\feedly_unread.json" -Raw | ConvertFrom-Json
$relevantIds = [System.Collections.Generic.List[string]]::new()
foreach ($feed in $data) {
    foreach ($item in $feed.Items) {
        $title = $item.title; if (-not $title) { continue }
        $matched = $false
        foreach ($pat in $kwPatterns) { if ($title -match $pat) { $matched=$true; break } }
        if (-not $matched) { foreach ($pat in $semanticPatterns) { if ($title -match $pat) { $matched=$true; break } } }
        if ($matched -and $item.id -notin $relevantIds) { $relevantIds.Add($item.id) }
    }
}
Write-Host "相关文章：$($relevantIds.Count) 篇"
```

输出匹配标题列表，供用户确认无误后继续。

### 步骤 5A：保存稍后阅读 + 标记全部已读（PowerShell）

```powershell
# 保存相关文章（每批 100 条）
$tagUrl = "https://feedly.com/v3/tags/user%2F$userId%2Ftag%2Fglobal.later"
for ($i=0; $i -lt $relevantIds.Count; $i+=100) {
    $batch = $relevantIds[$i..([Math]::Min($i+99, $relevantIds.Count-1))]
    $body  = @{entryIds=$batch} | ConvertTo-Json -Compress
    Invoke-RestMethod $tagUrl -Method PUT -Headers $headers -Body $body | Out-Null
}
Write-Host "已保存 $($relevantIds.Count) 篇到稍后阅读"

# 标记全部订阅为已读
$asOf = [DateTimeOffset]::UtcNow.ToUnixTimeMilliseconds()
foreach ($feed in $data) {
    $body = @{action="markAsRead";type="feeds";feedIds=@($feed.FeedId);asOf=$asOf} | ConvertTo-Json -Compress
    Invoke-RestMethod "https://feedly.com/v3/markers" -Method POST -Headers $headers -Body $body | Out-Null
    Write-Host "  [已读] $($feed.FeedTitle)"
}
```

### 步骤 6A：输出摘要

```
【筛选完成】
处理子源：N 个
文章总计：X 篇
→ 稍后阅读：Y 篇
→ 标记为已读：Z 篇
```

---

## UI 降级方式（步骤 1B–4B）

仅在 API 不可用时（token 提取失败、网络受限）使用此方式。

### 步骤 1B：确定目标子源

1. 使用 `read_page` 读取 Feedly 左侧边栏
2. 定位 Feeds → All → paper 文件夹，提取所有子源名称
3. 报告：`【paper 文件夹：共 N 个子源，本次处理 M 个】`

### 步骤 2B：单篇全流程验证

对第一个目标子源的**第一篇文章**执行完整操作：
1. hover 文章卡片 → 判断相关性 → 点击对应按钮
2. 确认文章已消失或归档（操作生效）
3. 验证通过后继续批量处理；若失败，报告错误原因并停止

### 步骤 3B：逐源处理

对每个子源按顺序执行：
1. 点击子源，等待中央内容区加载
2. 若出现"All Done"则跳过，报告 `[跳过] 子源名 — 无待处理文章`
3. 逐篇读取标题 → 判断相关性 → hover → 点击按钮：
   - **相关** → 点击最左侧"稍后阅读"（Save for Later）
   - **不相关** → 点击最右侧"标记为已读并隐藏"（Mark as Read & Hide）
4. 向下滚动加载更多，直到出现"Mark All as Read"按钮

### 步骤 4B：输出摘要

同步骤 6A 格式。

---

## UI 操作技术要点

```javascript
// hover 显示工具栏
const card = document.querySelector('.entry[data-entry-id="..."]');
card.dispatchEvent(new MouseEvent('mouseover', {bubbles: true}));

// 滚动加载
window.scrollBy(0, 800);
```

- 通过 `find` 工具定位 `.entry-toolbar` 内的具体按钮
- 若单篇操作失败，记录标题后跳过，继续下一篇

---

## 工具优先级

API 方式：`javascript_tool`（提取 token）→ PowerShell（所有 API 调用）

UI 降级：`read_page` → `find` → `javascript_tool` → `computer`（截图调试）
