# FeedlyReading

## 项目简介

Feedly 文章自动筛选工具。通过 Claude in Chrome 操作已登录的 Feedly 网页，遍历指定订阅文件夹下的所有子源，逐条判断文章标题相关性，自动标记"稍后阅读"或"标记为已读并隐藏"。

## 工作流

```
Chrome 打开 Feedly（已登录）
     ↓
/Feedly文章筛选 [过滤条件]
     ↓
遍历 paper 文件夹下所有子源
     ↓
逐条判断文章标题 → 标记为"稍后阅读"或"已读并隐藏"
     ↓
输出处理摘要
```

## 可用命令

| 命令 | 参数 | 说明 |
|------|------|------|
| `/Feedly文章筛选` | `[过滤条件]` | 按指定条件筛选 paper 文件夹下所有子源；不传参则使用默认条件 |

### 默认过滤条件

**关键词**（不区分大小写）：navigation、stress、prior、working memory、EEG

**语义主题**：空间导航（spatial navigation）、先验知识（prior knowledge）

## Skill

| Skill | 路径 | 触发命令 |
|-------|------|---------|
| Feedly 文章筛选 | `.claude/skills/Feedly文章筛选/SKILL.md` | `/Feedly文章筛选` |

同时注册为全局插件：`feedlyreading-skills@local-desktop-app-uploads`

## 运行规范

### 前提条件

执行 `/Feedly文章筛选` 前，确认：
- Chrome 已打开 Feedly 且已登录
- Claude in Chrome 插件已激活
- 当前页面在 feedly.com

### 最小改动原则

修改 Skill 或配置时：
- 只改动与当前需求直接相关的部分
- 改动前说明"改了什么、为什么改"

### 运行后询问

**每次执行命令结束后**，都询问用户：
> "是否需要更新 CLAUDE.md？（例如：新增规范、修正描述、记录本次发现的问题）"
