# 深入理解 HTTP 协议（含 Cookie 与 JWT）

## 1. 为什么需要 HTTP？

在 Web 环境中，客户端（如浏览器）与服务器需要交换信息。若没有统一规则，不同厂商的软件无法互操作。HTTP（HyperText Transfer Protocol，超文本传输协议）定义了：

*   请求的格式（方法、路径、头部、正文）
*   响应的格式（状态码、头部、正文）
*   资源定位方式（URL）
*   连接管理、缓存控制、状态保持等机制

简单来说：HTTP 是浏览器与服务器之间约定的“通信语言”，确保双方能准确理解对方意图。

## 2. HTTP 版本演进

| 版本       | 核心特性                                                                    | 主要局限                 |
| -------- | ----------------------------------------------------------------------- | -------------------- |
| HTTP/0.9 | <font style="color:rgb(15, 17, 21);">仅 GET 方法，只能返回纯文本（如早期的 HTML）</font> | 功能极简，无法传输图片或样式       |
| HTTP/1.0 | 引入状态码、头部、POST/HEAD 方法                                                   | 短连接：每次请求需重新建立 TCP 连接 |
| HTTP/1.1 | 持久连接（keep-alive）、管道化、Host 头                                             | 队头阻塞：同一连接上的请求必须串行响应  |
| HTTP/2   | 二进制分帧、多路复用、头部压缩、服务器推送                                                   | TCP 层面的队头阻塞仍存在       |
| HTTP/3   | 基于 QUIC（UDP）                                                            | 尚未完全普及               |

每个新版本都在解决前一版本的核心瓶颈。

## 3. HTTP 报文结构

### 3.1 请求报文

```http
POST /api/user HTTP/1.1
Host: example.com
Content-Type: application/json
Content-Length: 18

{"name":"Alice"}
```

组成：

*   请求行：方法、请求目标、HTTP版本
*   头部字段：键值对
*   空行：分隔头部与正文
*   消息正文：可选

### 3.2 响应报文

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 27

{"id":1,"name":"Alice"}
```

组成：

*   状态行：HTTP版本、状态码、原因短语
*   头部字段、空行、正文（同上）

## 4. HTTP 方法（动作语义）

| 方法      | 语义         | 是否携带主体 |
| ------- | ---------- | ------ |
| GET     | 获取资源       | 否      |
| HEAD    | 仅获取响应头部    | 否      |
| POST    | 创建资源       | 是      |
| PUT     | 全量替换资源     | 是      |
| PATCH   | 部分更新资源     | 是      |
| DELETE  | 删除资源       | 可有可无   |
| OPTIONS | 查询服务器支持的方法 | 否      |

**语义核心**：GET 用于读取，POST 用于创建，PUT/PATCH 用于更新，DELETE 用于删除。

## 5. HTTP 状态码分类

| 类别  | 范围      | 含义    | 典型示例                            |
| --- | ------- | ----- | ------------------------------- |
| 1xx | 100-101 | 信息性响应 | 100 Continue                    |
| 2xx | 200-206 | 成功    | 200 OK，204 No Content           |
| 3xx | 300-308 | 重定向   | 301 永久搬家，302 临时跳转，304 未修改       |
| 4xx | 400-451 | 客户端错误 | 400 错误请求，401 未认证，403 禁止，404 未找到 |
| 5xx | 500-511 | 服务器错误 | 500 内部错误，503 服务不可用              |

**速记**：2xx 成功，3xx 去别处，4xx 你错了，5xx 它错了。

## 6. HTTPS 的安全机制

HTTP 是明文传输，存在窃听、篡改、冒充三大风险。HTTPS = HTTP + TLS/SSL，在 TCP 与 HTTP 之间增加安全层。

| 风险       | 解决方案                   |
| -------- | ---------------------- |
| 窃听（内容被看） | 混合加密（非对称交换对称密钥，对称加密数据） |
| 篡改（内容被改） | 消息认证码校验完整性             |
| 冒充（假网站）  | 数字证书（CA 签发），验证服务器身份    |

**结论**：HTTPS 比 HTTP 多了加密、认证、完整性保护三层机制。

## 7. HTTP 缓存机制

缓存可减少重复请求，提升性能。分为两类：

| 类型   | 控制字段                                                   | 行为                      |
| ---- | ------------------------------------------------------ | ----------------------- |
| 强制缓存 | Cache-Control: max-age=3600、Expires                    | 有效期内直接使用本地副本，不发请求       |
| 协商缓存 | ETag / If-None-Match、Last-Modified / If-Modified-Since | 向服务器验证资源是否过期；若未变化返回 304 |

<font style="color:rgb(15, 17, 21);">优先级：</font>Cache-Control  >   Expires；服务器通常优先验证ETag，再验证 Last-Modified

流程图如下：

<!-- 这是一张图片，ocr 内容为： -->

<img width="834" height="1186" alt="image" src="https://github.com/user-attachments/assets/3253181e-3483-4623-b841-7f83453a281c" />


## 8. 一次完整的 HTTP 事务

从输入 URL 到页面展示（HTTP 相关部分）：

1.  **DNS 解析**：域名 → IP 地址。
2.  **TCP 连接**：三次握手建立连接。
3.  **发送请求**：浏览器构建 HTTP 请求报文，通过 TCP 发送。
4.  **服务器处理**：解析请求，执行逻辑，生成响应。
5.  **返回响应**：服务器发送 HTTP 响应报文。
6.  **浏览器处理**： 拿到 HTML 后开始<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">关键渲染</font>。
7.  **连接管理**：若 Connection: keep-alive，连接保持；否则关闭连接。

若为 HTTPS，则在 TCP 连接后增加 TLS 握手（证书验证、密钥协商）。

完整流程如下图：

<!-- 这是一张图片，ocr 内容为： -->

<img width="1080" height="1601" alt="image" src="https://github.com/user-attachments/assets/bd8418d0-9b46-4e98-a6d5-1d5cc9360a14" />


<font style="color:rgb(15, 17, 21);">HTTP 事务完成后，浏览器进入渲染流程（详见《深入理解浏览器渲染流程》），如下图所示：</font>

<!-- 这是一张图片，ocr 内容为： -->

<img width="623" height="1154" alt="image" src="https://github.com/user-attachments/assets/2c15ccc7-72f8-4c87-b28b-70cfdd94e95a" />


## 9. 状态保持机制：Cookie 与 JWT

HTTP 本身是无状态协议——每个请求都是独立的。为了实现登录状态等功能，需要在请求之间传递身份标识。Cookie 和 JWT 是两种主流的解决方案。

### 9.1 Cookie

Cookie 是服务器通过 `Set-Cookie` 头部要求浏览器保存的文本。浏览器在后续同源请求中自动携带。

**工作流程**：

1.  服务器响应：`Set-Cookie: sessionId=abc123; Path=/; HttpOnly; Secure`
2.  浏览器保存该 Cookie。
3.  后续请求自动携带：`Cookie: sessionId=abc123`

**常用属性**：

| 属性                | 作用                                                                                                                                                                                                                                                                                                                                                                   |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Expires / Max-Age | 控制有效期                                                                                                                                                                                                                                                                                                                                                                |
| Path              | 限制作用路径                                                                                                                                                                                                                                                                                                                                                               |
| Domain            | 限制作用域名                                                                                                                                                                                                                                                                                                                                                               |
| HttpOnly          | 禁止 JavaScript 访问（防 XSS：在你页面执行恶意 JS ）                                                                                                                                                                                                                                                                                                                                 |
| Secure            | <font style="color:rgb(15, 17, 21);">设置了 </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">Secure</font>`<font style="color:rgb(15, 17, 21);"> 的 Cookie 仅通过 HTTPS 传输，不会在 HTTP 请求中发送（避免明文泄露</font> ），Secure 不直接防 CSRF，但它是构建安全登录体系的基础，配合 `<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">SameSite</font>` 才能构成完整防御 |
| SameSite          | 限制 Cookie 只在本网站发送，跨站请求不自动带 Cookie，控制跨站请求是否携带（防 CSRF）                                                                                                                                                                                                                                                                                                                 |

**优点**：浏览器自动管理。\
**缺点**：跨域支持复杂，有 CSRF 风险。（**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">攻击者利用你已经在目标网站登录的登录状态</font>**，在别的恶意页面里，偷偷替你发起操作请求）

### 9.2 JWT（JSON Web Token）

JWT 是一种紧凑的令牌格式。服务器登录后返回加密签名字符串，客户端后续请求手动携带。

**JWT 结构**：`xxxxx.yyyyy.zzzzz`，三部分组成：

*   **<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">Header</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：我是什么类型、用什么算法</font>
*   **<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">Payload</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：我带了什么数据（公开可见）</font>
*   **<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">Signature</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：我没被篡改（服务器密钥保证）</font>

**工作流程**：

1.  用户登录，服务器验证成功后生成 JWT 并返回。
2.  客户端保存 JWT（通常存于 localStorage）。
3.  客户端在后续请求头部添加：`Authorization: Bearer <JWT>`
4.  服务器验证签名，读取用户信息。

**优点**：

*   无状态，易于水平扩展。
*   跨域友好。
*   自包含用户信息。

**缺点**：

*   无法主动注销（过期前始终有效）。
*   体积较大。
*   存于 localStorage 时易受 XSS 攻击。

### 9.3 Cookie vs JWT 对比

| 维度       | Cookie                     | JWT                                                                                             |
| -------- | -------------------------- | ----------------------------------------------------------------------------------------------- |
| 存储位置     | 浏览器自动存储（可设 HttpOnly）       | 前端手动存储（通常 localStorage）                                                                         |
| 传输方式     | 自动携带（Cookie 头）             | 手动放入 Authorization 头                                                                            |
| **跨域**支持 | 需配置 withCredentials 和 CORS | **<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">配置简单</font>** (只需基础 CORS) |
| 状态管理     | 服务端存储会话（有状态）               | 服务端无状态（令牌自包含）                                                                                   |
| 安全风险     | CSRF                       | XSS                                                                                             |
| 主动注销     | 服务端删除会话即可                  | 需引入黑名单或缩短有效期                                                                                    |
| 适用场景     | 传统 Web 应用（同域）              | 移动端、微服务、跨域 API                                                                                  |

## 10. 跨域（CORS）

### 10.1 为什么会有跨域？

浏览器的**同源策略**（Same-Origin Policy）限制：**协议、域名、端口**三者任意一个不同，即为跨域。同源策略的目的是防止恶意脚本读取其他网站的数据（如窃取 Cookie、劫持登录状态）。

### 10.2 跨域解决方案

| 方案           | 原理                                             | 适用场景                                |
| ------------ | ---------------------------------------------- | ----------------------------------- |
| CORS（跨域资源共享） | 服务器设置响应头 `Access-Control-Allow-Origin` 允许指定源访问 | 标准方案，支持所有请求方法，需后端配合                 |
| JSONP        | 利用 `<script>` 标签不受同源限制，只支持 GET                 | 老旧浏览器兼容，仅 GET 请求                    |
| 代理服务器        | 前端请求同源代理，代理转发到目标服务器                            | 开发环境（webpack devServer）、生产环境（nginx） |
| postMessage  | 跨窗口通信 API                                      | 不同源的 iframe 或弹出窗口通信                 |
| WebSocket    | 原生支持跨域                                         | 实时双向通信                              |

### 10.3 CORS 核心响应头

| 响应头                                | 作用                                            |
| ---------------------------------- | --------------------------------------------- |
| `Access-Control-Allow-Origin`      | 允许的源（`*` 或具体域名）                               |
| `Access-Control-Allow-Methods`     | 允许的 HTTP 方法（GET, POST, PUT, DELETE 等）         |
| `Access-Control-Allow-Headers`     | 允许的自定义请求头                                     |
| `Access-Control-Allow-Credentials` | 是否允许携带 Cookie（设为 true 时，Allow-Origin 不能为 `*`） |
| `Access-Control-Max-Age`           | 预检请求（OPTIONS）的缓存时间（秒）                         |

### 10.4 预检请求（OPTIONS）

#### 什么是预检请求？

预检请求是在**跨域**的前提下，浏览器在发送**非简单**请求前，自动发起的一个 `OPTIONS` 请求，用于询问服务器是否允许实际请求。代码中无需手动编写。

#### 简单请求 vs 非简单请求

| 类型    | 条件                                                                                                                    | 是否触发预检 |
| ----- | --------------------------------------------------------------------------------------------------------------------- | ------ |
| 简单请求  | 方法为 GET/HEAD/POST，且 Content-Type 为表单类型（`application/x-www-form-urlencoded`、`multipart/form-data`、`text/plain`），且无自定义头 | 不触发    |
| 非简单请求 | 不满足上述任一条件（如 PUT/DELETE 方法、`application/json`、携带 `Authorization` 头等）                                                   | 触发预检   |

> 注：`application/x-www-form-urlencoded` 是默认的普通表单类型，`multipart/form-data` 用于带文件上传的表单，`text/plain` 为纯文本格式；`application/json` 用来发送 JSON 格式数据，它不属于传统表单类型，是 Ajax 常用格式，属于非简单请求。

## 11. 总结

HTTP 定义了请求与响应的格式、方法、状态码、缓存规则。它本身无状态，因此引入 Cookie 和 JWT 维持用户状态。跨域问题由浏览器的同源策略引起，CORS 是标准的解决方案，其中预检请求用于保护非简单跨域请求的安全性。理解这些概念，就能处理跨域、缓存、安全、登录态管理等前端核心问题。
