# na_hitokoto_page

`na_hitokoto_page` 是 `na_hitokoto` 系列的静态展示页面。当前仓库不需要构建步骤，直接用 HTML、CSS 和 Vue 3 CDN 渲染页面。

当前页面包含：

- `/`：展示 `na_hitokoto_content` 提供的当前文本。
- `/history/`：展示 `na_hitokoto_history` 的公开历史记录样本。

## 系列关系

相关服务大致分工如下：

- `na_hitokoto_prompt`：聚合外部语料并输出动态 prompt。
- `na_hitokoto_content`：消费 prompt，调用模型生成当前文本，并写入当前文本缓存。
- `na_hitokoto_history`：保存生成历史，并提供历史记录查询接口。
- `na_hitokoto_page`：前端展示页，只负责读取公开接口并展示结果。

## 文件结构

```text
.
├── index.html
├── history/
│   └── index.html
└── README.md
```

`index.html` 是首页。

`history/index.html` 是历史页。静态站点部署后，常见服务器会把 `/history` 重定向或解析到 `/history/`，再读取 `history/index.html`。如果部署平台没有目录索引能力，需要在平台侧配置 `/history` 到 `history/index.html`。

## 首页行为

首页从下面的公开接口读取当前文本：

```text
https://content.hitokoto.natsuki.cloud/
```

响应是纯文本。页面使用 `fetch(..., { cache: "no-cache" })` 避免浏览器复用旧内容。

首页右上角有一个“历史”入口，指向：

```text
/history/
```

## 历史页行为

历史页从下面的公开接口读取历史记录：

```text
https://history.hitokoto.natsuki.cloud/get
```

该接口来自 `na_hitokoto_history` 的 `GET /get`。它是公开接口，不需要 Bearer token。

响应格式是 JSON object：

```json
{
  "0123456789ABCDEF": "stored text"
}
```

页面会把每个键值对渲染成一条记录：

- key：历史内容 ID，通常是 16 位大写 HEX。
- value：历史文本内容。

如果接口返回 `{}`，页面显示“暂无历史记录”。这不是错误状态。`GET /get` 读取的是 history 服务里的公开随机 KV 缓存；刚部署、库为空或缓存尚未刷新时，返回 `{}` 是正常行为。

重要限制：

- `/history/` 当前展示的是 `GET /get` 提供的公开随机历史样本，不是完整历史列表。
- `GET /get` 当前最多返回 128 条历史内容。D1 中不足 128 条时，返回实际存在的数量。
- `GET /get` 不提供时间戳，所以页面不能按创建时间排序。
- `GET /get` 不保证刚写入的内容立刻出现。history 服务的公开随机缓存由 cron 刷新，当前约每小时刷新一次。
- history 服务内部的 `random_history` KV key 保存公开随机样本的 ID 索引，具体内容按 `id -> content` 分别保存为独立 KV 记录；这不改变前端看到的响应格式。

## Token 与安全边界

`na_hitokoto` 系列服务目前约定复用同一个业务 token。受保护接口使用：

```http
Authorization: Bearer <ACCESS_TOKEN>
```

本地环境变量里可能有：

```text
na_hitokoto_token
```

但是这个静态前端页面不应该读取或注入该 token。原因是：

- 浏览器端代码和请求头对访问者可见。
- `GET /get` 本身不需要 token。
- 把 token 放进静态页面会泄露受保护接口权限。

如果后续要在页面里调用 `POST /match`、`POST /delete` 或其他受保护接口，不要直接在前端使用 `na_hitokoto_token`。应该增加一个后端代理或受控管理端，由服务器读取 token 并调用 history 服务。

## 本地预览

这个仓库没有 `package.json`，不需要安装依赖。

可以用任意静态文件服务器预览，例如：

```sh
python3 -m http.server 8000
```

然后访问：

```text
http://localhost:8000/
http://localhost:8000/history/
```

如果只直接打开本地 HTML 文件，首页内容读取通常仍可工作，但 `/history/` 这种目录路由更适合通过静态服务器验证。

## 部署注意事项

部署目标需要支持静态文件托管。

必须能访问这些路径：

- `/index.html`
- `/history/index.html`

推荐让平台支持目录索引：

- `/` 解析到 `index.html`
- `/history/` 解析到 `history/index.html`
- `/history` 重定向到 `/history/`，或直接解析到 `history/index.html`

页面依赖外部 CDN：

```text
https://unpkg.com/vue@3/dist/vue.global.prod.js
https://fonts.googleapis.com/css2?family=Noto+Sans+SC:wght@100..900&display=swap
```

如果未来要离线化或减少第三方依赖，需要把 Vue 和字体资源改为本地托管。

## 给后续集成的上下文

如果下一个对话需要继续集成，请保留以下事实：

1. 本仓库是静态前端，没有构建系统。
2. 首页接口是 `https://content.hitokoto.natsuki.cloud/`，返回纯文本。
3. 历史页接口是 `https://history.hitokoto.natsuki.cloud/get`，返回 JSON object。
4. `/get` 是公开接口，不需要 token。
5. `na_hitokoto_token` 只应用于受保护服务端调用，不能暴露给浏览器。
6. `/get` 返回的是公开随机样本，当前最多 128 条；它不是完整历史、不是按时间排序的列表。
7. 历史页已经处理 loading、错误、空数据和刷新。

## 人工检查清单

修改页面后建议至少检查：

- 首页是否还能显示当前文本。
- 首页“历史”入口是否跳转到 `/history/`。
- `/history` 是否能打开历史页，或被正确重定向到 `/history/`。
- history 接口返回 `{}` 时是否显示空状态。
- history 接口返回记录时是否正确显示 ID 和文本。
- 控制台是否没有 CORS 或 JavaScript 错误。
