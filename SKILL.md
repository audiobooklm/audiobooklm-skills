---
name: audiobooklm
description: 基于新音剪创作平台以及珠峰AI自研音色，提供播客创作、内容管理、音频生成、专辑上架等功能。适用于用户提出"播客生成"、"播客制作"等需求。
homepage: https://aigc.ximalaya.com
---

# audiobooklm

`audiobooklm_mcp` 的标准播客/有声书生产链路。用户提供文本(或让用户先生成),最终产出可上架的章节音频。

---

## 核心原则

1. **每一步可选择的操作都要征求用户意见**
2. **工具调用前必须先问用户确认，禁止自动决定**
3. **使用播客类音色时优先选择 category 为"播客"的音色**
4. **整章合成用 chapter_task_create，不要逐段 task_create**
5. **先混音再上架，顺序不能乱**

---

## Step 0 · 环境自检（强制前置，必须最先执行）

⚠️ **在询问用户任何业务问题（文本来源、书籍、章节名等）之前，必须先确认 MCP 服务可用。** 

**检查方式（二选一即可）：**
1. 查看当前会话已注册的 MCP 工具列表，是否存在 `mcp__audiobooklm_mcp__*` 前缀的工具（例如 `mcp__audiobooklm_mcp__book_create`、`mcp__audiobooklm_mcp__list_tts_voices`）。
2. 或调用一个轻量只读工具试探，例如 `mcp__audiobooklm_mcp__list_tts_voices()`。

**判定与处理：**

| 情况 | 处理 |
|---|---|
| 工具列表中存在 `mcp__audiobooklm_mcp__*` 且调用正常 | ✅ 进入第 1 步 |
| 找不到 `mcp__audiobooklm_mcp__*` 工具 | ❌ 输出下方配置指引，**停下来等用户配置并重启**，不要继续问业务问题 |
| 工具存在但返回 401/403/unauthorized/token expired/connection refused | ❌ 同上，Token 失效或服务不可达，提示用户检查 Token 并重启 |

**配置指引（未配置时输出给用户）：**
> 检测到当前会话尚未接入 `audiobooklm_mcp`，需要先完成 MCP 配置再继续。按以下 4 步操作：
> 1. 注册/登录 https://aigc.ximalaya.com/user
> 2. 创建制作组，生成 API Token
> 3. 把 MCP 服务写入配置：
> ```jsonc
> {
>   "mcpServers": {
>     "audiobooklm_mcp": {
>       "type": "http",
>       "url": "https://aigc.ximalaya.com/audiobooklm/mcp",
>       "headers": {
>         "Authorization": "Bearer <Token>"
>       }
>     }
>   }
> }
> ```
> 4. 重启服务，确认mcp已连接成功(需要结合当前环境给出具体的重启提示)
>
> 配置并重启后告诉我，我再继续帮你做播客。

**重要：** 在用户确认 MCP 配置完成并重启之前，**不要**询问文本来源、书籍、章节等任何业务信息，避免做无用功。
**提示信息：** 结合当前环境，给出用户简单的MCP配置方法，并提示用户如何重启能够生效。  

---

## 标准流程

### 1. 确认信息

开始前必须确认以下信息：

| 字段 | 是否必需 | 处理方式                                        |
|---|---|---------------------------------------------|
| 文本内容 | 必需 | 用户已给直接用；让你生成则先出文案确认                         |
| 书籍 | 必需 | 询问：新建还是用已有？                                 |
| 章节名 | 必需 | 询问用户确认                                      |
| 文本形态 | 必需 | 普通旁白/单人播客 vs 多人对话（决定导入工具）                   |
| 音色 | 必需 | 调用 list_tts_voices 向用户展示候选,询问用户使用哪个音色还是自动推荐 |
| 专辑 | 可选 | 仅上架时需要，询问：新建/已有/不上架                         |

---

### 2. 口语化润色(可选)

**询问用户:是否需要对文本进行口语化润色?**

- **否** → 跳过本步骤,使用原文进入下一步
- **是** → 使用下方 prompt 对**全部文本**进行改写,展示润色后的文本给用户,并**询问用户是否满意**:
  - **满意** → 使用润色后的文本进入下一步
  - **不满意** → 根据用户反馈再次调整,直到用户确认满意

**润色 prompt:**

```
你是一个专业的播客脚本润色专家。请对用户输入的文本进行改写,使其听起来像自然口语风格。严格遵循以下要求:

【风格要求】
1. 适量使用以下语气词,适当插入到句子开头、中间或结尾:
   - 呃、嗯、好吧、好、啊、呢、哦
   - 你注意、什么意思呢、也就是说、你看、重点来了
2. 增加口语化连接词和停顿,例如"那"、"所以"、"但是"、"其实"、"说白了"、"换句话说"。
3. 保持原文的所有事实、数据、逻辑、专有名词(人名、地名、机构名、数字、百分比、日期、价格等)完全不变,不得增删实质性内容。
4. 不要改变原文的段落结构和论述顺序,但可以通过添加语气词使长句更易读。

【格式要求】
1. 输出文本必须适合TTS(文字转语音)朗读,特殊符号进行处理(比如"安娜·卡列宁"改为"安娜卡列宁","x%"改为"百分之x")
2. 英文单词保持不变,数字根据上下文转为对应的汉字。

【输出示例风格】
原文:"黄金跌了10%。这是1983年以来最狠的一次。"
改写:"呃,黄金跌了百分之十。你注意哦,这是一九八三年以来最狠的一次。"
```

⚠️ **多人对话**:逐角色润色每行台词内容,保留 `[角色名]` 前缀不变。

---

### 3. 确定书籍

**询问用户：新建书籍还是用已有书籍？**

- **新建** → 向用户确认书名后调用：
  ```
  book_create(book_name="<书名>", book_type="podcast")
  ```

- **已有** → 用 `read_abs(scope={"domain":"books"})` 拉取书架，列出最近 10 本让用户选择

---

### 4. 创建章节

根据文本形态选择工具：

**普通旁白/单人播客 → `podcast_import`**
```
podcast_import(
  book_id=<book_id>,
  chapter_name="<章节名>",  # 新建章节时必填
  script_lines=["文本1", "文本2", ...], # 单人播客只需要传入文本列表，不需要角色标识
)
```

**多人对话 → `podcast_import`**
```
podcast_import(
  book_id=<book_id>,
  chapter_name="<章节名>",  # 新建章节时必填
  script_lines=["[角色名] 文本1", "[角色名] 文本2", ...] # 多人播客需要给出每段文本的角色名
)
```

**注意：**
- 多人播客每行格式必须为 `[角色名] 文本`
- 单人播客每行只需要文本，不需要角色名，格式为`文本`
- chapter_name 和 chapter_id 二选一：新建章节传 chapter_name，追加到已有章节传 chapter_id
- 返回 chapter_id 和 characters (角色名与 character_id 映射)

---

### 5. 列出音色

调用 `list_tts_voices()` 获取音色列表。

**筛选建议：**
- 播客类优先选 category 包含"播客"的音色
- 根据角色特征推荐合适的音色给用户选择
- 向用户展示 3-5 个候选 + 简短描述

**询问用户**：每个角色分别选哪个音色？

---

### 6. 绑定音色到角色

```
mcp__audiobooklm_mcp__patch_abs(
  scope={"domain": "book", "book_id": <book_id>},
  operations=[
    {
      "op_id": "op-<角色序号>",
      "type": "update_character",
      "character_id": <character_id>,
      "patch": {"speaker_id": <speaker_id>},
      "reason": "绑定音色"
    }
    // 多个角色用逗号分隔
  ]
)
```

⚠️ **注意**：
- 每个 operation 都要有唯一 op_id
- type 是 "update_character" 不是 "op"

---

### 7. 整章合成

使用 `chapter_task_create` 一次提交整章：

```
mcp__audiobooklm_mcp__chapter_task_create(
  book_id=<book_id>,
  chapter_id=<chapter_id>
)
```

返回 task_ids 列表。

---

### 8. 查询任务结果

```
mcp__audiobooklm_mcp__task_query(
  task_ids=[<task_ids列表>],
  wait_timeout=120  # 可根据段落数调整
)
```

所有段落 status=succeeded 后继续。

---

### 9. 章节混音

```
mcp__audiobooklm_mcp__chapter_audio_mix(
  book_id=<book_id>,
  chapter_id=<chapter_id>
)
```

⚠️ **顺序要求**：必须先完成第 8 步，再调用混音。

返回 `audio_url` 即章节完整音频。

---

### 10. 上架专辑（可选）

**询问用户**：需要上架到专辑吗？新建/已有/不上架？

- **新建专辑** → 询问专辑名后调用：
  ```
  mcp__audiobooklm_mcp__create_album(
    album_name="<专辑名>",
    book_content="<简介>"  # 可选，让 AI 生成封面
  )
  ```

- **已有专辑** → 用户提供 album_id

- **上架**：
  ```
  mcp__audiobooklm_mcp__upload_audio_to_album(
    album_id=<album_id>,
    audio_url=<第9步返回的audio_url>，
    title=<专辑音频标题>
  )
  ```

  上架完成后，交付用户看的站内地址目前只要求返回专辑 URL：
  - 专辑 URL：`https://www.ximalaya.com/album/<album_id>`

  注意：`uploaded_tracks[].audio_url` 如果是 COS 或 mix 文件地址，只能作为底层音频文件地址，不等同于用户可访问的喜马拉雅站内页面 URL。当前 `upload_audio_to_album` 未稳定返回 `sound_id`，不要编造或推断 `https://www.ximalaya.com/sound/<sound_id>`。

---

## 交付内容

完成后告诉用户：

- 书籍 ID / 章节 ID
- 选用的音色 (speakerId + 名称)
- **章节完整音频文件 URL**：第 8 步混音返回的 `audio_url`，通常是 COS/mix 文件地址
- **专辑 URL**：如果执行了上架，返回 `https://www.ximalaya.com/album/<album_id>`
- **编辑页面**：https://aigc.ximalaya.com/edit/<book_id>/<chapter_id>
- **账单或充值页面**: https://aigc.ximalaya.com/payment
- **编辑或充值页面需要在用户中心切换制作组身份登录操作 https://aigc.ximalaya.com/user**

---

## 常见问题

| 现象 | 原因 | 解决 |
|---|---|---|
| 找不到 `mcp__audiobooklm_mcp__*` | MCP 未注册 | 配置 MCP |
| 401/403/unauthorized | Token 无效/过期 | 重新生成 Token |
| patch_abs validation_failed | 缺 op_id 或 type | 检查 operation 格式 |
| task_query 返回 processing | 异步任务未完成 | 继续轮询 task_query |
| 音色不对 | character 未绑定 speaker | 重新绑定步骤 |
| create_album 失败 | 需要 book_content | 传入简介文本 |

---

## 多人对话模板

适用于用户已提供：book_id + 章节名 + 对话脚本（格式为 `[角色名] 文本`）

### 流程
1. 询问用户**是否口语化润色** → 选"是"则按润色 prompt 改写每行台词并请用户确认满意
2. `podcast_import` 导入章节 → 获得 chapter_id 和 characters
3. `list_tts_voices` 列出音色 → 用户为每个角色选择
4. `patch_abs` 绑定音色 → 每个角色都要绑
5. `chapter_task_create` 提交整章
6. `task_query` 查询结果
7. `chapter_audio_mix` 混音


