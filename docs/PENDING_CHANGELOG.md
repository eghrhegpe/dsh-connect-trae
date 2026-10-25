# 待写入 CHANGELOG 的条目（下一个版本）

> **这份文件是一块暂存区，不是文档。** 发版时把下面的内容整段追加到
> `CHANGELOG.md` 顶部（`# Changelog` 之后、最新已发布版本之前），然后删除本文件。
>
> **为什么不能直接写进 `CHANGELOG.md`：** 它是**随包文件**（在 `package.json` 的
> `files` 里）。已发布的 tag `v4.5.2` 之后若改动任何随包文件，
> `pnpm run verify:release` 会（正确地）报出「tag 之后改动了发布内容」——
> 因为那意味着线上包不再等于「tag 指向的源码」。而未发布版本的条目本来就
> 不属于已发布的那份 CHANGELOG。
>
> 放在 `docs/` 下是安全的：`docs/*.md` 不进发布包（只有
> `docs/assets/dsh-connect-trae-usage-card.png` 随包），所以改它不会污染发布内容。

---

## 未发布

> **文生图先只在国内版面世，并且必须由用户亲手勾选。** 企业版的租户网关是另一条
> 计费面，国际版按另一套套餐计费，两者的生图行为都没有测过——所有取证都在
> `trae-api-cn.mchost.guru` 上做的。

### 新增

- **文生图接进会话：`trae_draw_image`**（`feat(tools)`）。
  - 用登录账号画一张图，工具把**图片字节一起取回来**、写进会话目录的 `trae-images/`，并在结果里给
    相对路径 `path`（渲染层据此把图显示出来）。默认关闭、**按面板单独开**：
    卡片模型栏勾上「允许模型自主生图」（`drawImageEnabled`）该面板才注册这个工具，
    取消勾选立即撤下。
  - **协议是实测出来的，不是从二进制里猜的**：`POST /api/ide/v1/tool_text_to_image`
    （另有 SSE 变体），请求体**只认** `{prompt, image_size?}`——多带 `mode_type`/`session_id`/
    `image_urls` 之类的已知字段直接 400 空 body，未知字段反被忽略；成功响应是**扁平**的
    `{"message":"success","uri":"tos-cn-i-…"}`，没有 `{code,msg,data}` 信封。模型 `seedream_5`。
  - **尺寸边界**：总像素必须落在 **3686400 ~ 16777216**（`1024x1024` 报
    `image size must be at least 3686400 pixels`、`4608x4608` 报 at most，两者都**不扣费**）；
    预设 `2k`/`3k`/`4k` 可用，缺省 `1920x1920`。
  - **计费口径写进工具文案**：实测 `1920x1920` 一张约 **8.8 积分**（国内版价），上游不回传
    扣费数字。失败后加 **30 秒冷却**（按张计费，不过冷却的重试等于再买一张）。
  - **那条「取图」GET 是又一次生成，已停用**：同路径带 `?prompt=&image_size=` 的 GET 不是取图。
    决定性探针（CN 个人版）：POST 拿到 `…/1792954795741_44f94116_jpeg_1920x1920` 之后，用同一
    `(prompt, size)` 带全套网关头 + `x-ide-sign` 打 GET，回来的是**另一个** uri
    （`…/1792954810505_b107ac11_jpeg_1920x1920`），耗时 27 秒、**一个像素都没有**——等于又画
    一张、又扣一次（两次共约 17.6）。非法尺寸探针（`100x100`）同样印证：报错直接来自 Ark 的
    `[callArkImageGeneration]`。发版代码**不再发这条 GET**。
  - **真正的取图是两步、零额外计费**（同日实测）：`POST /api/ide/v1/get_resource_url`，body 必须
    正是 `{"uri_list":["<tos key>"]}`——`{uri}`/`{path}`/`{resource_uri}`/`{uris:[…]}`/
    `{resources:[…]}`/`{scene:'report',uri}` 以及 GET `?uri=` 统统只回 200 + 空 `url_map`
    （不报错，也不给地址）；应答是 `{"url_map":{"<key>":"<byteimg 签名 URL>"}}`，再直接 GET 那个
    签名地址（**不带任何 Trae 头**，签名在 query 里、带 `x-expires`）即得字节——实测
    `image/jpeg`、1,155,578 字节、头 `ffd8ffe0`。死路已排除：`get_resource_upload_token` 给的
    TOS STS 凭证读桶一律 `InvalidAccessKeyId`/`NoSuchBucket`。
  - **落盘与显示**：`generate` 返回 `{url, path, credit?}`，图片写在 `<session cwd>/trae-images/`
    下（时间戳 + 随机后缀，png/webp/jpeg 各用各的后缀）；`render` 输出
    `![Trae 文生图](trae-images/…)`——DSH 的聊天按 session cwd 解析相对路径，再过宿主的
    `api/file` 鉴权路由取字节，所以图直接显示。只有字节真的没拿到时（`IMAGE_UNAVAILABLE`：
    图已生成、积分已扣）才退化成「只有对象 key」的行内代码，且该失败码也在冷却集合里。
  - 测试：`tests/image-tool-render.spec.ts` 9 例（字节落盘、png 后缀、credit 回显、hint、
    无 path 时回退 key、写盘失败即 `IMAGE_UNAVAILABLE`、三种冷却）与
    `tests/image-tool-download.spec.ts` 3 例（stub fetch 断言恰好三跳、`uri_list` 请求体、
    CDN 请求不带网关头、空 `url_map` 即失败），钉住「一次生成 = 一次 POST + 一次解析 +
    一次 CDN GET，且不再出现 `?image_size=` 的 GET」。
  - **只有国内版面板提供**（`DRAW_IMAGE_PANELS = ['cn']`，见 `src/status-paths.ts`）。
    企业版走租户网关、国际版按另一套套餐计费，两者的生图行为都**没有测过**：企业版原先
    就因为这一点被拒绝，国际版没有理由凭「面板是消费者版」放行——而且工具文案里的
    8.8 积分是国内价，放到国际版就是错的。两处的勾选框都**置灰并写明原因**（不隐藏：
    隐藏会让人以为插件漏了这个功能），理由与「加进白名单必须先补实测证据」写在同一处注释里。
  - **注册开关是一个可测的纯函数**：`drawImagePanelsOf()` 三道门——面板在白名单内、
    该面板的 provider 开着（opt-out 语义，旧配置仍算开）、用户显式勾选（**只认字面量
    `true`**，`'yes'`、`1` 之类的手改值不算同意）。企业版即使被勾上也**结构性**拒绝，
    不靠白名单列表的书写顺序。首版把这条规则写在 `apply()` 的闭包里，只能靠读代码确认，
    现在由 `tests/image-tool-optin.spec.ts` 的 12 例钉住（缺省/非法形状/非布尔真值/关掉的
    provider/企业版/国际版/多面板顺序/整段 section 与裸 `regions` map 同答）。
  - **开关就在卡片里，不是一个只有 schema 的字段**：真值是 `regions.<panel>.drawImageEnabled`
    （Host 的工具解析按面板读它，所以写的是 `regions` 而不是顶层设置键）。勾选即写、不经过
    「保存」，写后**回读文档**核对——一个谎报授权的勾选框会让「工具没出现」看起来像 bug。
    4 例卡片测试 `tests/card-draw-image-switch.spec.tsx` 断言写入字段是 `regions`、该槽位的
    模型目录/图片模型/上下文预算/账号池一个都不能丢。
  - **范围**：改图（需先走 TOS 上传预签名）与文生视频（异步素材轮询）**不做**，也不留半截入口。

### 文档

- `README.md` / `README.en.md` 新增「文生图」一节与功能列表条目：开关语义（含只有国内版
  提供）、实测协议表、计费、取图两步（`get_resource_url` → 签名地址 → 字节落盘 → 相对路径
  显示），以及为什么那条「取图」GET **是又一次生成**。
