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
