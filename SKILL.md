---
name: video-to-script
description: YouTube 视频转口播稿：用户提供视频链接后，完成字幕提取、要点提炼、口播化改写的完整流程。触发词：视频转口播稿、口播稿生成、YouTube 字幕提取、视频文案提取、口播化改写。
---

# Video to Script Skill

用户提供 YouTube 视频链接，完成从字幕获取、内容提炼到口播稿生成的完整流程。

---

## 受众画像

### 博主定位
中文互联网 AI 博主，平台覆盖 Bilibili、小红书、微信视频号、抖音。

### 核心受众
1. **小白用户（70%）** - 25-40岁，对 AI/科技感兴趣但缺乏专业背景，追求"听得懂的干货"
2. **互联网/AI 从业者（30%）** - 产品、研发、运营，追求深度和效率

### 受众特征
- 时间宝贵，厌恶废话
- 追求"可执行"的干货
- 喜欢有深度但不学术化
- 对新事物开放，但警惕割韭菜
- 渴望"别人不知道的信息差"

### 内容调性
- 深入浅出：有质量的 insight 用通俗方式讲出来
- 专业但不装逼
- 直接但不粗暴
- 有料但不啰嗦
- 真诚但不煽情

---

## 执行流程

### Phase 1: 字幕获取

#### Step 1: 解析视频 URL

从用户提供的 YouTube URL 中提取 video ID。支持以下格式：
- `https://www.youtube.com/watch?v=xxx`
- `https://youtu.be/xxx`
- `https://www.youtube.com/live/xxx`

#### Step 2: 用 yt-dlp 获取字幕

优先使用 yt-dlp，语言优先级：**英文优先，中文兜底**。

```bash
VIDEO_ID="<extracted_video_id>"
OUTPUT_DIR="~/yt-insight/subtitles"
mkdir -p "$OUTPUT_DIR"

# 第一优先：英文手动字幕
yt-dlp --write-sub --sub-lang "en,en-US,en-GB" --skip-download \
  --sub-format "srt/vtt/best" \
  -o "$OUTPUT_DIR/$VIDEO_ID" "<youtube_url>" 2>/dev/null

# 检查是否拿到
SUB_FILE=$(ls "$OUTPUT_DIR/$VIDEO_ID"*.{srt,vtt} 2>/dev/null | head -1)

# 第二优先：英文自动生成字幕
if [ -z "$SUB_FILE" ]; then
  yt-dlp --write-auto-sub --sub-lang "en,en-US,en-GB" --skip-download \
    --sub-format "srt/vtt/best" \
    -o "$OUTPUT_DIR/$VIDEO_ID" "<youtube_url>" 2>/dev/null
  SUB_FILE=$(ls "$OUTPUT_DIR/$VIDEO_ID"*.{srt,vtt} 2>/dev/null | head -1)
fi

# 第三优先：中文手动字幕
if [ -z "$SUB_FILE" ]; then
  yt-dlp --write-sub --sub-lang "zh,zh-Hans,zh-CN" --skip-download \
    --sub-format "srt/vtt/best" \
    -o "$OUTPUT_DIR/$VIDEO_ID" "<youtube_url>" 2>/dev/null
  SUB_FILE=$(ls "$OUTPUT_DIR/$VIDEO_ID"*.{srt,vtt} 2>/dev/null | head -1)
fi
```

同时获取视频元信息：
```bash
yt-dlp --print "%(title)s|||%(duration_string)s|||%(channel)s|||%(upload_date)s|||%(thumbnail)s" "<youtube_url>" 2>/dev/null
```

#### Step 3: Supadata 兜底

仅当 yt-dlp 完全无法获取字幕时，才调用 Supadata API：

```bash
curl -s "https://api.supadata.ai/v1/youtube/transcript?video_id=$VIDEO_ID" \
  -H "x-api-key: $(grep SUPADATA_API_KEY ~/yt-insight/.env | cut -d'=' -f2)"
```

如果 Supadata 也失败，告知用户该视频无可用字幕，流程终止。

#### Step 4: 解析字幕内容

对 yt-dlp 获取的字幕文件进行清洗：
- VTT 格式：去除头部、时间戳、定位标签、空行、重复行
- SRT 格式：去除序号、时间戳、格式标签、空行、重复行

保留纯文本内容用于后续分析。

---

### Phase 2: 内容提炼

#### Step 5: 结构化分析

对字幕内容进行深度分析，输出 **5 个模块**：

##### 模块1: 内容速览 (overview)
```json
{
  "oneSentenceSummary": "一句话总结核心价值",
  "coreInsightsSummary": ["3-5个核心观点的一句话总结"],
  "targetAudience": "适合谁看",
  "creationValue": {
    "score": "1-5",
    "reason": "评分理由：是否适合做口播稿"
  }
}
```

##### 模块2: 嘉宾背景 (guest)
```json
{
  "name": "嘉宾姓名",
  "title": "身份头衔",
  "uniqueness": "为什么值得听这个人讲"
}
```
> 如果视频没有明确嘉宾（如纯教程、独白），此模块可为 null。

##### 模块3: 核心观点 (insights)
```json
[
  {
    "id": "insight-1",
    "title": "观点标题",
    "timeRange": { "start": "MM:SS", "end": "MM:SS" },
    "coreArgument": "核心论点",
    "evidences": [
      { "type": "data|case|quote|analogy|logic", "content": "内容" }
    ],
    "goldenQuote": {
      "original": "英文原文",
      "translation": "中文翻译",
      "timestamp": "MM:SS"
    },
    "plainExplanation": "大白话解释，初中生能懂",
    "conceptExplanations": [
      { "term": "术语", "explanation": "通俗解释" }
    ]
  }
]
```
> 术语解释直接内嵌在对应观点中，不单独成模块。

##### 模块4: 金句收集 (goldenQuotes)
```json
[
  {
    "quote": "英文原文",
    "translation": "中文翻译",
    "timestamp": "MM:SS",
    "context": "金句出现的语境",
    "useCase": "口播稿中可以怎么用"
  }
]
```

##### 模块5: 质量评估 (qualityAssessment)
```json
{
  "scores": {
    "informationDensity": 4,
    "uniqueness": 3,
    "evidenceStrength": 4,
    "accessibility": 5
  },
  "overallScore": 4,
  "overallRecommendation": "总体建议：是否值得做口播稿，为什么",
  "topInsightsToExpand": ["insight-1", "insight-3"]
}
```

#### Step 6: 保存分析结果

组装完整 JSON，保存到：
```
youtube-insight-dashboard/data/analyses/{videoId}.json
```

完整 JSON 结构：
```json
{
  "meta": {
    "videoId": "",
    "videoTitle": "",
    "videoUrl": "",
    "channelName": "",
    "thumbnailUrl": "",
    "duration": "",
    "publishedAt": "",
    "analyzedAt": "",
    "subtitleSource": "yt-dlp | supadata",
    "subtitleLang": "en | zh"
  },
  "overview": {},
  "guest": {},
  "insights": [],
  "goldenQuotes": [],
  "qualityAssessment": {}
}
```

#### Step 7: 展示分析概览 ✋ 确认点 1

向用户展示：

```
## 内容分析完成

**视频**: {videoTitle}
**频道**: {channelName} | **时长**: {duration}
**字幕来源**: {subtitleSource} ({subtitleLang})

### 一句话总结
> {oneSentenceSummary}

### 核心观点 ({n} 个)
1. {insight-1 title} — {plainExplanation 摘要}
2. {insight-2 title} — {plainExplanation 摘要}
...

### 精选金句
> "{quote translation}" — {context}

### 创作价值评分: {overallScore}/5
{overallRecommendation}

---
**是否继续生成口播稿？**
```

使用 AskUserQuestion 工具等待用户确认。如果用户说不继续，流程结束。

---

### Phase 3: 口播稿生成

#### Step 8: 二次提炼

对 Phase 2 的分析结果进行口播稿导向的二次加工：

**8.1 价值密度评估**
- 哪些观点是"常识复述"可以略过？
- 哪些是"真正的洞察"必须保留？
- 对于 70% 小白受众，哪些需要额外解释？

**8.2 差异化角度挖掘**
1. 这个话题别人怎么讲的？我们有什么不同视角？
2. 有没有被忽视的"反常识"点？
3. 能否加入独家案例/数据/经历？

**8.3 冲突点识别**
- 有没有挑战主流观点的内容？
- 有没有"你以为...其实..."的转折？
- 有没有行业痛点可以戳？

#### Step 9: 选择叙事框架

根据内容类型选择最适合的结构：

| 框架 | 适用场景 | 结构 |
|------|---------|------|
| 问题-方案型 | 工具/方法论 | 痛点共鸣 → 常见误区 → 正确方法 → 实操步骤 → 效果预期 |
| 认知颠覆型 | 新观点/趋势 | 主流认知 → 反常识冲击 → 深层原因 → 新的理解 → 行动启示 |
| 案例拆解型 | 成功/失败案例 | 结果展示 → 背景铺垫 → 关键转折 → 方法提炼 → 适用边界 |
| 信息差型 | 行业内幕/趋势 | 信息钩子 → 信息来源 → 深度解读 → 影响分析 → 应对策略 |

#### Step 10: 设计情绪曲线

```
情绪强度
   ↑
   │    ╭─╮ 高潮1    ╭──╮ 高潮2(最高)
   │   ╱   ╲        ╱    ╲
   │  ╱     ╲      ╱      ╲    ╭─
   │ ╱       ╲    ╱        ╲  ╱  升华
   │╱ 钩子    ╲  ╱          ╲╱
   └────────────────────────────→ 时间
     开场   铺垫  冲突   解决  收尾
```

每个部分标注：
- 情绪目标（好奇/焦虑/恍然/兴奋/踏实）
- 语速建议（快/中/慢）
- 停顿位置（制造张力）

#### Step 11: 生成标题与结构概览 ✋ 确认点 2

生成 3-5 个标题选项：

| 类型 | 公式 | 示例 |
|------|------|------|
| 数字型 | [数字] + [核心价值] | "Claude Code的3个核心设计原则" |
| 悬念型 | [反常识现象] + 为什么 | "为什么最简单的架构反而最强？" |
| 对比型 | [A] vs [B]：[结论] | "RAG vs Grep：Claude Code选择了更笨的方案" |
| 揭秘型 | [权威来源] + 揭秘 + [主题] | "Anthropic工程师揭秘：Claude Code架构设计" |
| 痛点型 | [痛点] + 的解决方案 | "AI Agent总是不稳定？看看Claude Code怎么做的" |

**标题红线**：不超过25个汉字、不用"震惊""必看"等低质词、有信息增量。

向用户展示：

```
## 口播稿概览

**叙事框架**: {框架名} — {选择原因}
**差异化角度**: {我们的独特视角}
**预估时长**: {X分钟} / {X字}

### 推荐标题
> {推荐的标题}

### 备选标题
1. {标题1} — {类型}
2. {标题2} — {类型}
3. {标题3} — {类型}

### 结构概览
| 段落 | 主题 | 情绪目标 | 时长 |
|------|------|---------|------|
| 开场 | 钩子 | 好奇 | 30秒 |
| 1 | xxx | xxx | 2分钟 |
| ... | ... | ... | ... |
| 收尾 | 升华 | 踏实/激励 | 30秒 |

### 金句引用计划
- "{quote}" → 用在{section}，效果：{purpose}

---
**框架和标题是否OK？确认后生成完整文案。**
```

使用 AskUserQuestion 工具等待用户确认。用户可能会要求调整标题或结构。

#### Step 12: 撰写完整口播稿

基于确认的结构，撰写逐字稿。

**文案格式规范**：
```
【段落标题】

一句话作为一行。
不要把多个句子挤在一行。
每行控制在15-25字之间。
需要停顿的地方，单独成行。

[画面提示：xxx]
[字幕强调：xxx]

---
```

**文案写作原则**：

1. **口语化** — 用"你"不用"您"，用短句，用具体词，像和朋友聊天
2. **节奏感** — 重要观点前停顿，金句后留白，长短句交替，反问制造互动感
3. **信息密度** — 每30秒一个信息点，删掉废话连接词，用具体数字，一段一个核心点
4. **情绪设计** — 开场好奇/焦虑，中间"恍然大悟"，结尾踏实/激励
5. **深入浅出** — 术语出现时立刻用大白话翻译，用类比降低理解门槛（照顾70%小白受众）
6. **金句引用** — 在适当位置自然引入原视频金句，给出中文翻译，增加权威感和记忆点
7. **结尾升华** — 从具体案例提升到普遍规律，从操作方法到底层逻辑，给出行动启示

**开场钩子模板库**：

| 类型 | 模板 |
|------|------|
| 问题型 | "你有没有想过，为什么...却总是..." |
| 数据型 | "最新数据显示...[反直觉的数据]" |
| 场景型 | "上周我看到一个视频，彻底刷新了我对...的认知" |
| 反常识型 | "关于[话题]，大多数人都理解错了" |
| 信息差型 | "这个信息很多人还不知道..." |

**结尾升华公式**：
- 认知层面：从具体案例 → 普遍规律 → 跨域启发
- 情感层面：从焦虑 → 踏实（明确路径）/ 从困惑 → 清晰（认知框架）/ 从旁观 → 行动
- 结尾模板："说到底，[核心洞察]" / "与其[常见做法]，不如[新视角]" / "记住，[可以带走的一句话]"

#### Step 13: 保存口播稿

将口播稿数据追加到已有的分析 JSON 文件中，新增 `scriptStructure` 字段：

```json
{
  "meta": { "..." },
  "overview": { "..." },
  "guest": { "..." },
  "insights": [],
  "goldenQuotes": [],
  "qualityAssessment": {},
  "scriptStructure": {
    "meta": {
      "narrativeFramework": "问题-方案型 | 认知颠覆型 | 案例拆解型 | 信息差型",
      "coreHook": "一句话说清楚为什么要看这个视频",
      "uniqueAngle": "差异化视角",
      "targetDuration": "预估时长",
      "difficulty": "入门 | 进阶 | 深度"
    },
    "titles": {
      "recommended": "推荐标题",
      "options": [
        {
          "title": "标题文案",
          "type": "数字型 | 悬念型 | 对比型 | 揭秘型 | 痛点型",
          "strength": "优势"
        }
      ]
    },
    "opening": {
      "type": "问题型 | 数据型 | 场景型 | 反常识型",
      "hook": "开场钩子文案",
      "emotionGoal": "好奇/焦虑/共鸣",
      "duration": "30-45秒"
    },
    "structure": [
      {
        "order": 1,
        "section": "段落主题",
        "purpose": "这一段要达成什么目的",
        "keyPoints": ["核心要点"],
        "quotesToUse": ["原视频金句引用"],
        "transition": "到下一段的过渡句",
        "emotionCurve": "情绪走向",
        "pacing": "快/中/慢",
        "duration": "预估时长"
      }
    ],
    "closing": {
      "elevation": "升华的核心信息",
      "callToAction": "希望观众做什么",
      "memorableEnd": "留下的最后一句话",
      "emotionGoal": "踏实/激励/思考"
    },
    "fullScript": {
      "totalWordCount": "总字数",
      "estimatedDuration": "预估时长（按150字/分钟）",
      "sections": [
        {
          "sectionTitle": "段落标题",
          "lines": [
            "第一句话。",
            "第二句话。",
            "",
            "停顿后的下一句。"
          ],
          "visualCues": ["画面/字幕提示"],
          "duration": "本段时长"
        }
      ]
    },
    "productionNotes": {
      "visualCues": ["需要画面配合的节点"],
      "textOverlays": ["需要字幕强调的金句"],
      "paceChanges": ["节奏变化位置"]
    },
    "spreadStrategy": {
      "thumbnailConcept": "封面创意建议",
      "keywordsToPlant": ["自然植入的关键词"],
      "hookForShorts": "切片钩子"
    },
    "qualityChecklist": {
      "infoIncrement": "是否有足够的信息增量？",
      "logicChain": "逻辑链条是否完整？",
      "emotionRhythm": "情绪节奏是否有起伏？",
      "actionable": "观众看完能做什么？",
      "memorable": "有没有能记住的金句？"
    }
  }
}
```

---

## 输出展示

生成完成后，输出完整口播稿文案：

```
---
# {推荐标题}
---

## 【开场】

第一句话。
第二句话。

[画面：xxx]

---

## 【第一部分：xxx】

第一句话。
第二句话。

[字幕强调：xxx]

---

## 【第二部分：xxx】

...

---

## 【结尾】

升华句。
行动号召。
记忆点金句。

---
```

---

## 质量红线

最终输出前自检，以下任何一条不通过必须修改：

1. **前30秒法则**：开场有没有给出"为什么要看完"的理由？
2. **信息增量**：去掉常识后，还剩多少干货？
3. **情绪波动**：至少有2个情绪高点？
4. **逻辑闭环**：论点-论据-结论自洽？
5. **行动触发**：观众看完知道该做什么？
6. **记忆点**：有没有一句能被记住的话？
7. **小白友好**：术语都有通俗解释？类比是否到位？
