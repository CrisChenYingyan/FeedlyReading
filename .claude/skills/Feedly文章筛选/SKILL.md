---
name: Feedly文章筛选
description: 自动操作 Chrome 中已打开的 Feedly 页面，遍历左侧 paper 文件夹下所有订阅源，逐条判断文章标题相关性，相关则点击"稍后阅读（Save for Later）"，不相关则点击"标记为已读并隐藏（Mark as Read & Hide）"。触发命令：`/Feedly文章筛选 [过滤条件]`，不传参时使用默认过滤条件（关键词：navigation / stress / prior / working memory / EEG；语义主题：空间导航、先验知识）。凡涉及 Feedly 文章批量筛选、已读标记、稍后阅读分拣等自动化操作，均应调用此 Skill。
---

# Feedly 文章筛选

当用户输入 `/Feedly文章筛选 [过滤条件]` 时执行。

**前提条件**：Chrome 浏览器已打开 Feedly（feedly.com）且已登录，Claude in Chrome 插件已激活。

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

用户传入的任何内容**完全替换**默认条件，作为本次的过滤逻辑。格式不限，自然语言即可：

- `/Feedly文章筛选 只保留与大脑记忆巩固相关的文章`
- `/Feedly文章筛选 关键词：attention, reward, dopamine；主题：强化学习与神经机制`

---

## 执行步骤

### 步骤 1：确认条件并报告

在开始前输出本次过滤条件，格式：

```
【本次筛选条件】
关键词：navigation, stress, prior, working memory, EEG（含中文对应词）
语义主题：空间导航、先验知识
```

### 步骤 2：获取 paper 文件夹下的子源列表

1. 使用 `read_page` 读取 Feedly 左侧边栏内容
2. 定位路径：Feeds → All → paper 文件夹
3. 展开 paper，提取所有子源（feed source）名称及顺序
4. 报告子源总数：`【paper 文件夹：共 N 个子源】`

### 步骤 3：逐源处理

对每个子源**按顺序**执行以下操作：

#### 3a. 点击子源，等待中央内容区加载完成

#### 3b. 检查是否为空

若中央内容区（main feed panel）出现"All Done"、"全部完成"等提示，报告：

```
[跳过] 子源名 — 无待处理文章
```

并跳至下一个子源。

#### 3c. 处理当前可见的所有文章条目

对每篇文章执行：

1. **读取标题**：提取文章卡片的标题文本
2. **判断相关性**：
   - 先做关键词匹配（任一关键词命中 → 相关）
   - 若无关键词命中，基于标题语义判断是否与指定主题高度相关
3. **执行操作**：
   - **相关** → hover 文章卡片，点击**最左侧**的"稍后阅读"按钮（Save for Later）
   - **不相关** → hover 文章卡片，点击**最右侧**的"标记为已读并隐藏"按钮（Mark as Read & Hide）

#### 3d. 滚动加载更多

处理完当前可见文章后，向下滚动页面，等待新文章加载，继续执行 3c。

**停止条件**：检测到页面底部出现"全部标记为已读（Mark All as Read）"按钮时，停止滚动，进入下一子源。此按钮出现表示该子源所有待处理文章已全部加载完毕。

### 步骤 4：输出摘要

全部子源处理完毕后输出：

```
【筛选完成】
处理子源：N 个（跳过 M 个）
文章总计：X 篇
→ 稍后阅读：Y 篇
→ 标记为已读并隐藏：Z 篇
```

---

## 技术操作要点

### 按钮触发方式

Feedly 文章操作按钮（4 个图标）在 hover 后才显示。推荐方式：

```javascript
// 对文章卡片 dispatch mouseover 以显示工具栏
const card = document.querySelector('.entry[data-entry-id="..."]');
card.dispatchEvent(new MouseEvent('mouseover', {bubbles: true}));
```

之后通过 `find` 工具定位 `.entry-toolbar` 内的具体按钮，点击最左侧（Save for Later）或最右侧（Mark as Read & Hide）。

### 滚动加载

```javascript
window.scrollBy(0, 800);
```

每次滚动后等待约 1 秒，让 Feedly 懒加载完成，再读取新出现的文章。

### 错误处理

若单篇文章操作失败（找不到按钮、点击无响应），记录标题后跳过，继续处理下一篇，不中断整体流程。

### 工具优先级

`read_page` → `find` → `javascript_tool` → `computer`（截图用于调试状态确认）
