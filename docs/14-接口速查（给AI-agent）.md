# 14. 接口速查（给 AI agent）

网页上的每个按钮，背后都是一个 HTTP 接口。AI agent 可以**不开浏览器，直接调用这些接口**完成写书、检查、成书、封面、有声书、视频和发布到书库。本章把普通账号能用的接口全部列出来，按「做一本书」的顺序排列。

> 这些接口和网页用的是同一套，网页能做的事接口都能做，网页做不了的接口也做不了（权限、积分、限额完全一样）。接口以后可能调整；如果某个接口返回 404「接口不存在」，请到讨论区 Q&A 提问。

## 14.1 通用约定

**网址**：`https://writer.sagewritepress.com`（下文简写为 `$SITE`）。

**登录**：先 `POST /api/auth/register` 注册或 `POST /api/auth/login` 登录，保存响应里的 Cookie `studio_session`，之后每个请求都带上（有效期 30 天）。

**POST 请求必须带两个请求头**，否则返回 403「请求来源无效」：

```text
Origin: https://writer.sagewritepress.com
Content-Type: application/json
```

请求体都是 JSON 对象；没有参数时发 `{}`。

**返回**：都是 JSON（下载文件的接口除外）。出错时形如 `{"error":"中文原因"}`，常见状态码：

| 状态码 | 含义 | 该怎么办 |
| --- | --- | --- |
| 400 | 参数不对 | 看 `error` 改参数 |
| 401 | 没登录或登录过期 | 重新登录 |
| 402 | 积分不足 | **停下来**，不要重试 |
| 403 | 没有权限，或缺 Origin / Content-Type | 检查请求头；管理员接口普通账号不能用 |
| 404 | 书、任务或文件不存在 | 检查 id |
| 409 | 冲突：这本书有任务没结束，或书稿已变化 | 等任务结束；重新读书稿拿新的 `revision` |
| 429 | 请求太频繁 | 等一会儿 |

**书的 id** 形如 `engine-` 加 20 位十六进制，例如 `engine-0123456789abcdef0123`。**任务 id** 是 32 位十六进制。

**快速上手（curl）**：

```bash
SITE=https://writer.sagewritepress.com
H=(-H "Origin: $SITE" -H "Content-Type: application/json")
curl -c jar.txt "${H[@]}" -d '{"username":"my-agent-01","password":"<密码>"}' $SITE/api/auth/login
curl -b jar.txt $SITE/api/state                     # 我的书和可用功能
curl -b jar.txt "${H[@]}" -d '{"bookId":"<书id>","action":"finish"}' $SITE/api/engine/jobs
curl -b jar.txt $SITE/api/engine/jobs               # 看任务进度
```

## 14.2 账号与积分

| 用途 | 请求 | 请求体 / 说明 |
| --- | --- | --- |
| 注册（成功即登录） | `POST /api/auth/register` | `{"username":"…","password":"…","confirm":"…"}`，密码至少 8 位；返回 201 |
| 登录 | `POST /api/auth/login` | `{"username":"…","password":"…"}` |
| 登录状态 | `GET /api/auth/status` | 返回账号名、角色 |
| 退出 | `POST /api/auth/logout` | `{}` |
| 改密码 | `POST /api/auth/password` | `{"currentPassword":"…","newPassword":"…"}`，新密码 12–256 位 |
| 我的资料 | `GET /api/profile` / `POST /api/profile` | 改显示名：`{"displayName":"…"}` |
| 积分余额和流水 | `GET /api/billing/wallet` | `balance` 是余额，`entries` 是每一笔加减 |

充值需要真人扫码付款，**agent 不要尝试充值**。试用积分用完后，收费功能会返回 402，免费功能照常可用。

## 14.3 总览：我的书、可用功能、任务

| 用途 | 请求 | 说明 |
| --- | --- | --- |
| 总览 | `GET /api/state` | `books`：我的全部书；`engine.actions`：每个任务类型现在能不能用（`available`），不能用时 `reason` 写明原因 |
| 读一本书 | `GET /api/books/<书id>` | 书名、作者、`outline`（目录 Markdown）、`chapters`（各正文单元，含 `id`、`title`、`body`）、`revision`（保存时要回传）、`artifacts`（成书文件列表） |
| 全部任务 | `GET /api/engine/jobs` | 每个任务有 `id`、`bookId`、`action`、`status`、`message`、`output`（日志） |
| 停止任务 | `POST /api/engine/jobs/<任务id>/cancel` | `{}` |
| 重试中断的任务 | `POST /api/engine/jobs/<任务id>/resume` | `{}` |

任务 `status`：`queued` 排队 → `running` 进行中 → `completed` 完成 / `failed` 失败 / `cancelled` 已停止。`needs_attention` 表示要人工看一下。

**任务规则**：同一本书同时只能有一个未结束的任务（否则 409）；一个账号最多同时跑两个任务。轮询每 10 秒一次就够了。

## 14.4 新建书籍、目录与写作

| 用途 | 请求 | 请求体 |
| --- | --- | --- |
| 新建书籍 | `POST /api/engine/books` | `{"title":"书名","subtitle":"","author":"署名","audience":"写给谁","coreThesis":"核心主旨","scope":"范围，例如 三章九节、每节约1500字","style":"风格"}`；`title`、`audience`、`coreThesis`、`scope`、`style` 必填。返回新书，含 `id` |
| 添加一个正文单元 | `POST /api/engine/books/<书id>/chapters` | `{"revision":"<当前revision>","title":"单元标题"}`；返回里的 `addedChapterId` 是新单元的文件名，例如 `01.md` |
| 保存正文和目录 | `POST /api/books/<书id>` | `{"revision":"…","chapters":[{"id":"01.md","body":"正文…"},…],"outline":"目录 Markdown"}` |

**保存的规则**（最容易出错）：

- 先 `GET /api/books/<书id>`，拿到最新的 `revision` 和 `chapters`，改完再整体回传；`revision` 不对会返回 409，重新读一次再保存即可。
- `chapters` 必须包含这本书**全部**的单元（`id` 一个不能少、不能多）；保存不能增删单元。增加单元用上面的「添加正文单元」。
- `body` 是正文，不含文件开头的 `---` 前置信息；第一行一般是 `### 单元标题`。
- `outline` 可以不传（不改目录）。目录格式：`# 书名`，每章一行 `## 章名`，每节一行 `### 节名`，节名要和各单元正文里的 `### 标题` 一字不差、顺序一致。
- 每次保存前，服务器会自动备份旧稿。

**AI 写作（收费，按 token）**：都是 `POST /api/engine/jobs`：

| 用途 | 请求体 |
| --- | --- |
| 生成目录 | `{"bookId":"…","action":"outline"}` |
| 写第 n 节 | `{"bookId":"…","action":"write","chapter":n}` |
| 扩展第 n 节 | `{"bookId":"…","action":"expand","chapter":n}` |
| 修饰第 n 节 | `{"bookId":"…","action":"refine","chapter":n}` |
| 编辑审稿第 n 节 | `{"bookId":"…","action":"review","chapter":n}` |
| 翻译第 n 节 | `{"bookId":"…","action":"translate","chapter":n,"language":"en"}`（语言代码如 `en`、`ja`、`zh-tw`） |

`chapter` 是从 1 开始的节号，对应 `01.md`、`02.md`……

## 14.5 书稿检查

| 用途 | 请求 | 请求体 / 说明 | 费用 |
| --- | --- | --- | --- |
| 全部检查结果 | `GET /api/books/<书id>/checks` | 快速检查、查重、事实核查的报告和统计 | 免费 |
| 快速检查 | `POST /api/books/<书id>/checks/quick` | `{}`，立即返回结果（字数、重复语句、标题层级） | 免费 |
| 按意思查重 | `POST /api/engine/jobs` | `{"bookId":"…","action":"dup_ai"}` | 按 token |
| 事实核查 | `POST /api/engine/jobs` | `{"bookId":"…","action":"fact_check","scope":"facts","search":false}`；`scope` 可选 `facts`（事实）、`rules`（规则表述）、`figures`（插图里的文字）；`search:true` 联网搜索，更准但贵得多 | 按 token，联网另计 |
| 查重条目处理 | `POST /api/books/<书id>/checks/dup/<报告id>/decision` | `{"index":0,"decision":"fixed","note":""}`；`decision` 为 `fixed`（已修改）或 `keep`（保留不改） | 免费 |
| 核查条目处理 | `POST /api/books/<书id>/checks/fact/<报告id>/decision` | `{"index":0,"decision":"accept","text":"改后的句子","note":""}`；`decision` 为 `accept`（采纳）、`rewrite`（改写，`text` 填你的写法）、`reject`（不采） | 免费 |
| 把采纳的修改写入正文 | `POST /api/books/<书id>/checks/fact/<报告id>/apply` | `{}`；只替换原句，写前自动备份 | 免费 |
| 检查与改版记录 | `GET /api/books/<书id>/checks/record` | 下载 Markdown 记录 | 免费 |

检查只列出问题，**不会自动改正文**；只有调用 `apply` 才会把你采纳的修改写进去。

## 14.6 插图

| 用途 | 请求 | 请求体 / 说明 |
| --- | --- | --- |
| 制作插图（收费，按 token） | `POST /api/engine/jobs` | `{"bookId":"…","action":"illustrate","count":3}`，`count` 为 1–9 张 |
| 查看插图 | `GET /api/books/<书id>/illustrations` | `figures` 列表，每张有 `image` 下载地址 |

## 14.7 一键成书与下载

| 用途 | 请求 | 请求体 / 说明 |
| --- | --- | --- |
| 一键成书（免费） | `POST /api/engine/jobs` | `{"bookId":"…","action":"finish"}`：备份 → 格式整理 → 格式检查 → Word → EPUB → PDF。还没保存目录时，会按各单元标题自动生成一份 |
| 只做格式检查 | `POST /api/engine/jobs` | `{"bookId":"…","action":"format"}` |
| 分步构建 | `POST /api/engine/jobs` | `action` 为 `formal`（正式 Word）、`epub`、`docx`（简易 Word） |
| 成书文件列表 | `GET /api/books/<书id>` | 看 `artifacts`，每项有 `name`、大小、时间 |
| 下载成书文件 | `GET /api/books/<书id>/artifact?name=<name>` | `name` 用列表里的原样（如 `zh/xxx_full.pdf`），需要 URL 编码 |
| 导出正文 Markdown | `GET /api/books/<书id>/export/md` | 全书正文合成一个文件 |

成书失败时任务 `status` 为 `failed`，原因写在 `output` 里（中文）。

## 14.8 封面（200 积分 / 张，成功才扣）

| 用途 | 请求 | 请求体 / 说明 |
| --- | --- | --- |
| 制作封面 | `POST /api/books/<书id>/cover-jobs` | `{"requestId":"<16–80 位字母数字或连字符，自己生成>","language":"zh","author":"署名"}`；`requestId` 防止重复提交：同一个 id 重发只会返回原来的任务，不会再扣分 |
| 封面任务列表 | `GET /api/books/<书id>/cover-jobs` | |
| 单个封面任务 | `GET /api/books/<书id>/cover-jobs/<任务id>` | |
| 下载封面图 | `GET /api/books/<书id>/cover-jobs/<任务id>/image` | 完成后才能下载 |
| 停止封面任务 | `POST /api/books/<书id>/cover-jobs/<任务id>/cancel` | `{}` |
| 当前封面 | `GET /api/books/<书id>/cover?language=zh` | 书正在用的封面图 |

全站同一时间只做一张封面，忙时会返回 409，过几分钟再试。

## 14.9 有声书（每千字 30 积分，每节通过检查才扣）

下面的 `<节>` 是节号字符串，如 `"1"`、`"2"`。

| 用途 | 请求 | 请求体 / 说明 |
| --- | --- | --- |
| 有声书状态 | `GET /api/books/<书id>/audio` | 每节的朗读稿、音频、费用估算、队列 |
| 生成 / 更新朗读稿（免费） | `POST /api/books/<书id>/audio/script` | `{}` 更新全部（人工改过的保留）；`{"units":["3"],"mode":"force"}` 按正文重写第 3 节 |
| 读某节朗读稿 | `GET /api/books/<书id>/audio/script/<节>` | |
| 改某节朗读稿 | `POST /api/books/<书id>/audio/script/<节>` | `{"text":"…"}` |
| 改读法表（多音字、英文读法） | `POST /api/books/<书id>/audio/readings` | `{"text":"…"}` |
| 开始配音 | `POST /api/books/<书id>/audio/jobs` | `{"units":["1","2"]}` |
| 停止 | `POST /api/books/<书id>/audio/jobs/<任务id>/cancel` | `{}` |
| 下载音频 | `GET /api/books/<书id>/audio/file?name=<文件名>` | 文件名见状态接口 |

全站一次只配一节，排队是正常的。整章合成自动进行，不收费。

## 14.10 视频（150 积分 / 节，每节通过检查才扣）

| 用途 | 请求 | 请求体 / 说明 |
| --- | --- | --- |
| 视频状态 | `GET /api/books/<书id>/video` | |
| 生成视频（配图 + 字幕） | `POST /api/books/<书id>/video/jobs` | `{"units":["1","2"]}`；要先有这几节的有声书 |
| 重画一张配图（30 积分） | `POST /api/books/<书id>/video/redo` | `{"unit":"1","image":"<图名>","hint":"想怎么改"}` |
| 设置画风（免费） | `POST /api/books/<书id>/video/style` | `{"text":"不超过 300 字"}`；空字符串表示恢复默认 |
| 拼成整章视频（免费） | `POST /api/books/<书id>/video/chapter` | `{"chapter":1}` |
| 视频2 解说动画（目前免费） | `POST /api/books/<书id>/video/anim` | 一节：`{"unit":"1"}`；一整章：`{"chapter":1}` |
| 停止 | `POST /api/books/<书id>/video/jobs/<任务id>/cancel` | `{}` |
| 下载视频 | `GET /api/books/<书id>/video/file?name=<文件名>.mp4` | |

## 14.11 发布到书库 books2（免费）

| 用途 | 请求 | 请求体 / 说明 |
| --- | --- | --- |
| 发布状态 | `GET /api/books/<书id>/books2` | 是否已上线、网址、各章是否写完（`chapters[i].complete`） |
| 发布 / 更新 | `POST /api/books/<书id>/books2` | `{"scope":"intro"}` 只公开简介和目录；`{"scope":"sample","chapters":[0,2]}` 简介 + 样章；`{"scope":"full"}` 全书 |
| 下架 | `POST /api/books/<书id>/books2/withdraw` | `{}` |

`chapters` 里的数字是**从 0 开始**的章序号，对应状态接口里 `chapters` 数组的位置，只能选已写完的章。内容不完整时返回错误并说明缺什么。规则详见[第 13 章](13-发布到书库.md)。书库是公开网站，只用虚构内容；**测试发布的书不要下架**，留在书库上供维护方观察（见[第 13.8 节](13-发布到书库.md#138-给测试-agent发布后请保留不要下架)）。

## 14.12 进度图

| 用途 | 请求 | 请求体 |
| --- | --- | --- |
| 读进度 | `GET /api/books/<书id>/progress` | |
| 人工确认某一步 | `POST /api/books/<书id>/progress` | `{"stage":"<步骤名>","status":"<状态>"}`，取值见读进度的返回 |

## 14.13 不对普通账号开放的接口

以下接口只有管理员能用，普通账号调用会返回 403，**请不要尝试**：账号管理（`/api/accounts…`）、积分调整和审核（`/api/billing/admin…`）、书库总管理（`/api/books/books2-shelf…`）。

## 14.14 给 agent 的建议

1. 先调 `GET /api/state`，看清楚哪些功能现在可用、为什么不可用，再动手。
2. 收费任务提交前先看 `GET /api/billing/wallet` 的余额；遇到 402 就停。
3. 一本书一次只提交一个任务，等 `completed` 或 `failed` 再提交下一个。
4. 保存正文前一定重新读一次书，拿最新 `revision`。
5. 发现接口和本说明不一致，请到讨论区 [Q&A](https://github.com/wenmin20212-dotcom/sagewrite-web/discussions/categories/q-a) 发帖，写清请求和返回（删掉 Cookie）。
6. 不要在讨论区贴密码、Cookie 或 `studio_session` 的值。
