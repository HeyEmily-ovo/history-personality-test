# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

"你是哪个历史人物"—— 基于大五人格(OCEAN)的单页面性格测试网站。50 题混合题型评估后，通过欧氏距离算法匹配 16 位中国历史人物，输出 ECharts 雷达图 + 柱状图及特质分析。

## 运行方式

无需构建工具。HTML 文件可直接用浏览器打开，或用 HTTP 服务器托管：

```bash
npx serve -l 7777 -s .
```
或
```bash
python3 -m http.server 7777
```

访问 `http://localhost:7777/index.html`。

## 部署

项目通过 Vercel 部署，项目名 `history-personality-test`，配置在 `.vercel/project.json`。`.vercel/` 目录不应提交到 Git。

## 架构概览

| 文件 | 职责 |
|---|---|
| `index.html` | 页面结构（三层 `<section>`） |
| `style.css` | 全部样式与响应式断点 |
| `script.js` | 题库、计分、匹配算法、人物头像SVG、ECharts 图表渲染 |
| `性格特质测试竞品调研报告.md` | 竞品分析文档（仅参考，不影响运行） |

三层 `<section>` 页面切换：

| 页面 | 元素ID | 触发方式 |
|---|---|---|
| 首页 | `#landing` | 默认显示 |
| 答题 | `#test` | `startTest()` 切换 |
| 结果 | `#results` | 答完全部题目自动切换 |

外部依赖仅 ECharts 5.5 CDN（`cdn.jsdelivr.net`），用于雷达图和柱状图。

## 核心技术细节

### 题库与计分

`questions` 数组定义 50 题，分属 5 个维度（O:开放性、C:尽责性、E:外向性、A:宜人性、N:神经质），每维度 10 题（6 道 Likert 量表 + 4 道情境选择）。

两种题型通过 `type` 字段区分：

| type | 说明 | 计分方式 |
|---|---|---|
| `'likert'` | 五级认同度量表（完全符合→完全不符合） | `key:1` 正向计分，`key:-1` 反向计分（`6 - rawVal`） |
| `'choice'` | 情境选择题（5 个选项对应不同行为倾向） | 每选项自带 `score`（1-5），直接累加到对应维度 |

`likertOptions` 数组控制 Likert 题的选项文字和分值。情境选择题的选项由每题 `choices` 数组自行定义。

计分函数 `calcScores()` 根据 `q.type` 分别处理两种题型，最终将每维度原始分映射到 0–100 百分比（`raw / max × 100`）。

### 神经质 → 情绪稳定性转换

雷达图、柱状图和维度解读中展示的是"情绪稳定性 S = 100 - N"，而非原始 N 值。高分代表情绪稳定，低分代表情绪敏感。

### 人物头像

`mkAvatar(id, color, icon)` 生成函数为每位人物输出内联 SVG 圆形徽章头像。渐变 ID 以 `s-${id}` 格式保证唯一性。SVG 结构：

- 纯色背景圆 + 径向渐变高光（左上角白色半透明）
- 内圈描边（白色 15% 透明度）
- 白色几何图标（每人物的标志性道具）

### 人物匹配算法

`figures` 数组：16 个对象，各有预设的 OCEAN 五维分值（0–100）。匹配使用欧氏距离：

```
dist = √Σ(scores[d] - figure.profile[d])²
similarity = (1 - dist / maxDist) × 100
```

其中 `maxDist = √(100² × 5)`。按相似度降序排列，首名为主要匹配，前 3 名为备选。

### 答题导航

`currentQ` 变量追踪当前题目索引。选答案后自动前进一题；末题选完后 `currentQ` 溢出触发 `renderQuestion()` → `showResults()`。导航栏提供上/下一题按钮（首尾禁用）和 50 题选项卡网格（点击 `goToQuestion(idx)` 直接跳转）。

题目卡片内 `#qHint` 元素根据题型显示不同引导文字：
- Likert 题：`请选择你的认同程度：`
- 情境选择题：`请选择最符合你的选项：`

选项卡（`#qTabs`）中，已答题目加 `answered` 类，当前题目加 `active` 类，情境选择题额外加 `tab-choice` 类以示区分。

### 结果面板

`showResults()` 渲染以下区块：
1. **主匹配人物卡片** (`#topFigure`)：头像、名称、匹配度、描述、名言、特质标签
2. **雷达图** (`#radarChart`)：五维可视化（N 转换为 S）
3. **柱状图** (`#barChart`)：五维得分横向对比
4. **维度解读** (`#dimAnalysis`)：每维度按高(≥70)/中(40-69)/低(<40)给出三档解读
5. **备选匹配** (`#otherMatches`)：第 2-4 名人物

### 响应式布局

| 断点 | 选项卡列数 | 其他调整 |
|---|---|---|
| > 600px | 10 列 | 桌面端默认 |
| ≤ 600px | 6 列 | 人物卡片竖排、维度单列、雷达图缩小 |
| ≤ 380px | 5 列 | 内边距缩减、导航按钮变小 |

## 数据隐私

所有答题数据仅存于浏览器内存（`answers` 对象），不发送网络请求，无 Cookie/本地存储持久化。
