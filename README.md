OpenAI Image Relay (OpenAI 图像中继)

该 SillyTavern 第三方扩展会监控助手的回复内容，通过用户定义的正则表达式 (Regex) 提取提示词文本，将其发送至兼容 OpenAI 标准的后端，并将匹配到的提示词文本替换为生成的图像。



预设后端

默认设置指向：



服务地址： [http://127.0.0.1:8199/v1/chat/completions](http://127.0.0.1:8199/v1/chat/completions)

API 密钥： sk-any

模型： any

该配置匹配 https://github.com/lumingya/universal-web-api 的本地后端，但该扩展也支持其他任何兼容 OpenAI 接口规范的服务。

## Local Dream 设备互联

在“图片后端配置”中将“后端类型”切换为 `Local Dream 设备互联`，然后把“Local Dream 控制地址”填成设备互联控制端口：

```text
http://127.0.0.1:8808
```

局域网设备则填对应手机或平板的 IP，例如 `http://192.168.1.20:8808`。插件只会：

1. 读取 `GET /info` 确认这是 Local Dream 设备互联端（可选，读不到不阻断）；
2. 读取 `GET /status` 确认后端状态为 `running`；
3. 向同一设备的 `8081` 端口 `POST /generate` 发送生图请求。

插件不会请求 `/models`、`/select`，也不会拉取或远程切换模型。请先在 Local Dream 设备上选择并加载模型，模型进入运行状态后再使用测试或聊天生图。

### 控制端口与生成端口分开处理

Local Dream 有两套端口：`8808` 控制 API 和 `8081` 原生生图后端。部分设备固件版本上，`8808` 不返回 CORS 头，浏览器页面（也就是 SillyTavern 前端）直连会被拦成 `MissingAllowOriginHeader`；而 `8081` 一直带 `Access-Control-Allow-Origin: *`，可以直连。

因此本扩展：

- 生成请求始终走 `8081`，浏览器直连即可；
- 控制端口的 `/info`、`/status` 属于“读得到就用于显示设备名和模型名，读不到就降级”，不再阻断生图；
- 读不到控制端口时，用 `GET 8081/health` 判定后端是否在线。

可选设置：

- **生图地址**：留空则自动推导为同主机 `:8081`；端口被改过时可手动填 `http://IP:PORT`。
- **请求通道**：`自动` 先直连、失败再走 SillyTavern 的 `/proxy/` 转发；`仅直连`；`仅走酒馆转发`。
  酒馆转发需要 `config.yaml` 里 `enableCorsProxy: true` 并重启酒馆。开启后控制端口也能读通，状态检查会显示设备名和已加载的模型。

### 生图参数

面板按手机端「高级参数设置」的布局组织：

| 设置 | 说明 |
|---|---|
| 图像宽高比 | `自动`（0×0，跟随设备当前分辨率）、`1:1`、`3:4`、`4:3`、`自定义`（手填宽高） |
| 调度器 | `Euler A` / `Euler` / `LCM` / `DPM++ 2M` / `DPM++ 2M SDE` |
| Karras | 给调度器加 `_karras` 后缀。`LCM` 没有 Karras 变体，选中时开关自动禁用 |
| 生成步数 / CFG Scale | 直接透传给 `/generate` |
| 重绘强度 Denoise | `denoise_strength`，仅 img2img 生效；纯文生图时后端忽略 |
| Seed | 留空则随机 |
| 超时（秒） | 单张图的等待上限，默认 600 秒 |

调度器实际发送的值会显示在 chips 下方（例如选 `Euler` + 开 Karras 就是 `euler_karras`）。取值来自 local-dream 的 `createScheduler()`，未知值会被后端静默回落到 DPM。

超时单独设成 Local Dream 专用是有必要的：手机 NPU 跑 20 步 1024×1024 常常要 2 分钟以上，用默认的 120 秒会中途中断。

### 角色卡自建前端的生图请求（事件桥）

有些角色卡（例如「诡秘之主：第五纪」）自带前端，不走聊天气泡里的 `<pic prompt="...">`，而是用一对桥接事件请求生图：

```js
// 角色卡前端
eventEmit("generate-image-request", { id, prompt, change, width, height });
eventOn("generate-image-response", (data) => {
    // data = { id, success: true, imageData } 或 { id, success: false, error }
});
```

`eventEmit` / `eventOn` 由 iframe 前端助手（ST-Prompt-Template / JS-Slash-Runner）提供，事件最终落在宿主页面的 `eventSource` 上。

扩展会监听 `generate-image-request`，用同一套后端逻辑生图，再把结果通过 `generate-image-response` 回传，`id` 原样带回。参数处理：

| 字段 | 处理 |
|---|---|
| `id` | 原样回传，前端靠它匹配请求 |
| `prompt` | 主提示词 |
| `change` | 追加到 `prompt` 后面（`prompt, change`） |
| `width` / `height` | 覆盖面板里的宽高；传 0 或省略则按面板设置和设备分辨率 |

注意 `eventEmit` 经 iframe 转发后，handler 收到的是 `[data]` 这种数组形态，扩展已兼容。

**超时必须两边一起调**：角色卡前端有自己的生图等待超时（`imageGenTimeoutSeconds`，默认 120 秒），手机 NPU 跑 20 步 1024×1024 要 250 秒左右，前端会先报超时。请把角色卡设置里的生图超时也调到 300 秒以上，并保证扩展这边的「超时（秒）」不小于它。



### 正文配图持久化

角色卡「诡秘之主：第五纪」把正文配图放在 `storyImageState` 里，源码注释写明「仅内存，不持久化」——刷新页面、切到下一回合再回来都看不到。

扩展补了一层持久化（设置里「正文配图持久化」，默认开启）：

1. 生图成功后，图片通过 `/api/images/upload` 落成服务器文件，返回 `/user/images/<角色名>/oair-*.jpg`。
2. 聊天元数据 `chat_metadata["ST-OpenAI-Image-Relay-story-images"]` 里按「正文指纹」记一条 `{ url, at, prompt }`。
3. 重新进入或翻回某个回合时，扩展算出当前正文的指纹，命中就把图挂回正文区（`#game-text-display` 内的 `#oair-story-image`），点击可看原图。

设计要点：

| 点 | 做法 | 原因 |
|---|---|---|
| 锚点 | 正文指纹 = 去空白后的长度 + 首尾采样哈希 | 不依赖角色卡内部数据结构；剥离空白是因为重渲染时会差一两个换行 |
| 存储 | 只存 URL，不存 base64 | 一张 1024×1024 JPEG 的 base64 约 450 KB，直接塞进聊天会让存档迅速膨胀 |
| 上限 | 元数据保留最近 60 条 | 防止无限增长 |
| 触发 | `CHAT_CHANGED`、`CHARACTER_MESSAGE_RENDERED`、正文区 `MutationObserver`、5 秒兜底轮询 | 角色卡内部换回合不一定改 DOM 结构，光靠观察器会漏 |
| 开销 | 观察器只盯 `childList`，同步做 400ms 节流，页面隐藏时跳过 | 正文区可能有几千字，避免频繁全量读取 |

如果角色卡自己已经把图挂出来了（`#story-image-slot img` 存在），扩展不会重复叠加。

图片文件位于 `data/<user>/user/images/<角色名>/`，同一回合重新生图会生成新文件，旧文件不会自动删除，需要时手动清理。



### 关于进度条步数

local-dream 的 `Pipeline::generate()` 里写死了：

```cpp
int total_run_steps = req.steps + (req.img2img ? 1 : 0) + 2;
```

后端把 **CLIP 文本编码**（第 1 次回调）和 **VAE 解码出图**（最后一次回调）也算进了总步数，所以设置的 20 步会被报成 22，实际 UNet 采样仍然是 20 次。

插件会把回调换回采样语义再显示，因此你会看到：

```text
Local Dream 编码提示词…（采样共 20 步）
Local Dream 采样 1/20
...
Local Dream 采样 19/20
Local Dream 解码图像…（采样 20/20 步已完成）
```

看到 `/20` 就是对的，不用去改步数。

重要设置项

服务 URL (Service URL)： 接收完整的 .../chat/completions 链接，或者像 .../v1 这样的基础路径。

提示词正则 (Prompt regex)： 提取内容中捕获组 1 或名为 "prompt" 的命名组内容并发送至后端。

用户提示词模板 (User prompt template)： 在发送给后端之前，对提取出的提示词进行包装/修饰。

额外正文 JSON (Extra body JSON)： 可选参数，用于覆盖或添加特定后端所需的请求体字段。

响应解析逻辑

扩展会按以下顺序尝试解析返回的图像：



结构化图像字段： 例如返回数据中的 media 或 images 字段。

自定义响应正则： 根据用户配置的“响应图像正则表达式”进行匹配。

Markdown 链接： 在 choices[0].message.content（消息正文）中的 Markdown 格式图片链接。

宽泛匹配： 返回文本中包含的任何普通图像 URL。
