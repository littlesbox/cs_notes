# curl 系统使用手册

## 1. curl 是什么

`curl`：

> Client URL

主要作用是：

```text
客户端
  │
  │ curl
  ▼
URL
  │
  ▼
服务器
```

例如：

```bash
curl https://example.com
```

就是：

```text
curl
  ↓
建立 TCP/TLS/HTTP 连接
  ↓
发送 HTTP Request
  ↓
接收 HTTP Response
  ↓
把 Response Body 输出到终端
```

所以对于后端开发而言，可以把 `curl` 理解成：

> **一个可以手工构造网络请求的命令行 HTTP 客户端。**

这也是为什么开发 API、调试 FastAPI、Nginx、WebSocket、代理、HTTPS、微服务时，`curl` 非常重要。

---

# 2. 基本语法

最基本的形式：

```bash
curl [options] URL
```

例如：

```bash
curl https://example.com
```

多个 URL：

```bash
curl https://example.com https://example.org
```

多个选项：

```bash
curl -v -L https://example.com
```

长选项：

```bash
curl --verbose --location https://example.com
```

`curl` 的命令行参数大致分成两类：

```text
短选项
-d
-H
-X
-o

长选项
--data
--header
--request
--output
```

长选项使用 `--`，短选项使用 `-`。很多短选项可以组合，例如：

```bash
curl -vL https://example.com
```

等价于：

```bash
curl -v -L https://example.com
```

官方文档说明，curl 目前已经有两百多个甚至更多命令行选项，因此真正学习 curl 时，应该按功能体系掌握，而不是死记所有参数。([Everything Curl](https://everything.curl.dev/cmdline/options/index.html?utm_source=chatgpt.com "Command line options - everything curl"))

---

# 3. 第一组：最基本的 HTTP 请求

## 3.1 GET

最简单：

```bash
curl https://example.com
```

默认就是：

```http
GET / HTTP/1.1
Host: example.com
```

因此：

```bash
curl https://api.example.com/users
```

相当于：

```http
GET /users HTTP/1.1
Host: api.example.com
```

---

# 4. 查看响应头

## 4.1 `-I` / `--head`

只发送 HEAD 请求：

```bash
curl -I https://example.com
```

例如：

```text
HTTP/2 200
content-type: text/html
content-length: 1256
server: nginx
```

适合检查：

```text
HTTP 状态码
Content-Type
Content-Length
Server
Location
缓存
Cookie
```

官方 curl 文档也推荐 `-I/--head` 用于获取资源的头部信息。([GitHub](https://github.com/curl/curl/blob/master/docs/MANUAL.md?utm_source=chatgpt.com "curl/docs/MANUAL.md at master · curl/curl · GitHub"))

---

# 5. 同时显示响应头和响应体

使用：

```bash
curl -i https://example.com
```

注意：

```text
-I
```

和

```text
-i
```

完全不同。

### `-I`

只看 Header：

```bash
curl -I URL
```

### `-i`

Header + Body：

```bash
curl -i URL
```

例如：

```text
HTTP/2 200
content-type: text/html

<html>
...
</html>
```

可以记成：

```text
-I → I = Information → 只看头
-i → include → 把 Header include 到输出
```

---

# 6. 显示详细网络过程

这是排查网络问题最重要的参数之一：

```bash
curl -v https://example.com
```

`-v`：

```text
verbose
```

会显示：

```text
DNS
TCP
TLS
HTTP Request
HTTP Response
```

例如可以看到：

```text
* Host example.com:443 was resolved.
* Trying 93.184.216.34:443...
* Connected to example.com
* SSL connection using TLSv1.3
> GET / HTTP/2
> Host: example.com
> User-Agent: curl/...
> Accept: */*
< HTTP/2 200
< content-type: text/html
```

当：

```bash
curl https://example.com
```

失败时，第一反应通常应该：

```bash
curl -v https://example.com
```

官方教程也明确建议，在连接失败、服务器拒绝请求或无法理解发生了什么时使用 `-v`。([GitHub](https://github.com/curl/curl/blob/master/docs/MANUAL.md?utm_source=chatgpt.com "curl/docs/MANUAL.md at master · curl/curl · GitHub"))

---

# 7. 更详细的网络调试

## `--trace`

```bash
curl --trace trace.log https://example.com
```

或者：

```bash
curl --trace-ascii trace.log https://example.com
```

它比：

```bash
-v
```

提供更底层的调试信息。

可以理解为：

```text
普通请求
    ↓
-v
    ↓
--trace
```

排查：

```text
HTTP协议问题
TLS问题
代理问题
连接问题
数据传输问题
```

非常有用。

---

# 8. HTTP 方法

curl 默认：

```http
GET
```

但是可以使用：

```bash
-X
```

或者：

```bash
--request
```

指定 HTTP Method。

---

## GET

```bash
curl -X GET https://api.example.com/users
```

实际上通常不需要写：

```bash
curl https://api.example.com/users
```

即可。

---

## POST

```bash
curl -X POST https://api.example.com/users
```

---

## PUT

```bash
curl -X PUT https://api.example.com/users/1
```

---

## PATCH

```bash
curl -X PATCH https://api.example.com/users/1
```

---

## DELETE

```bash
curl -X DELETE https://api.example.com/users/1
```

---

# 9. 一个非常重要的原则

很多人刚开始学 curl 喜欢：

```bash
curl -X POST ...
curl -X GET ...
```

但实际上：

> `curl` 的很多参数本身就会决定 HTTP Method。

例如：

```bash
curl -d "name=Tom" https://example.com
```

使用 `-d` 后，curl 会使用 POST。

所以：

```bash
curl -X POST -d "name=Tom" ...
```

很多时候：

```bash
curl -d "name=Tom" ...
```

就够了。

---

# 10. POST 表单数据

## 10.1 `-d`

```bash
curl -d "name=Tom&age=20" https://api.example.com/users
```

相当于：

```http
POST /users HTTP/1.1
Content-Type: application/x-www-form-urlencoded

name=Tom&age=20
```

---

# 11. POST JSON

后端 API 开发中非常常见。

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"name":"Tom","age":20}' \
  http://localhost:8000/users
```

服务器收到：

```json
{
    "name": "Tom",
    "age": 20
}
```

这是你使用 FastAPI 时非常重要的一种 curl 用法。

例如：

```bash
curl -X POST \
  http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"你好"}'
```

---

# 12. JSON 文件作为请求体

假设：

```text
request.json
```

内容：

```json
{
    "name": "Tom",
    "age": 20
}
```

可以：

```bash
curl \
  -H "Content-Type: application/json" \
  -d @request.json \
  http://localhost:8000/users
```

这里：

```text
@request.json
```

表示：

> 从文件读取请求数据。

curl 官方文档也提供了使用 `-d @file` 从文件读取请求数据的方式。([Everything Curl](https://everything.curl.dev/cmdline/options/args.html?utm_source=chatgpt.com "Arguments to options - everything curl"))

---

# 13. `--data` 的几个重要变体

常见：

```bash
-d
--data
```

还有：

```bash
--data-raw
--data-binary
--data-urlencode
```

---

## `--data`

```bash
curl -d "name=Tom" URL
```

---

## `--data-raw`

```bash
curl --data-raw '{"name":"Tom"}' URL
```

与 `--data` 类似，但不会把 `@` 特殊处理成文件引用。

---

## `--data-binary`

适合原始二进制数据：

```bash
curl --data-binary @data.bin URL
```

---

## `--data-urlencode`

自动进行 URL 编码：

```bash
curl --data-urlencode "name=张三" URL
```

非常适合：

```text
中文
空格
特殊字符
URL 参数
```

---

# 14. URL Query Parameter

例如：

```text
/users?page=1&size=20
```

直接：

```bash
curl "https://api.example.com/users?page=1&size=20"
```

注意一定要考虑 shell 的特殊字符，因此推荐加：

```bash
"..."
```

---

# 15. Headers

这是 API 调试中最重要的功能之一。

使用：

```bash
-H
```

或者：

```bash
--header
```

例如：

```bash
curl \
  -H "Content-Type: application/json" \
  https://api.example.com
```

多个 Header：

```bash
curl \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN" \
  -H "X-Request-ID: 123456" \
  https://api.example.com
```

---

# 16. Authorization

Bearer Token：

```bash
curl \
  -H "Authorization: Bearer eyJ..." \
  https://api.example.com/users
```

API Key：

```bash
curl \
  -H "Authorization: Bearer $API_KEY" \
  https://api.example.com
```

或者：

```bash
curl \
  -H "X-API-Key: $API_KEY" \
  https://api.example.com
```

建议不要直接把密钥写死：

```bash
curl -H "Authorization: Bearer sk-xxxxx"
```

而是：

```bash
export API_KEY="sk-xxxxx"

curl \
  -H "Authorization: Bearer $API_KEY" \
  https://api.example.com
```

---

# 17. Basic Authentication

使用：

```bash
-u
```

例如：

```bash
curl -u username:password https://example.com
```

也可以：

```bash
curl -u username https://example.com
```

然后 curl 提示输入密码。

推荐后者，因为密码不会直接出现在 shell 历史记录中。

---

# 18. 下载文件

## `-o`

```bash
curl -o file.txt https://example.com/file.txt
```

：

```text
-o
↓
output
```

---

## `-O`

```bash
curl -O https://example.com/file.txt
```

自动使用 URL 中的文件名：

```text
file.txt
```

区别：

```bash
-o myfile.txt
```

你指定文件名。

```bash
-O
```

服务器 URL 文件名决定文件名。

---

# 19. 下载多个文件

```bash
curl -O https://example.com/a.txt \
     -O https://example.com/b.txt
```

---

# 20. 自动跟随重定向

非常重要：

```bash
-L
```

或者：

```bash
--location
```

例如：

```bash
curl -L http://example.com
```

服务器：

```http
HTTP/1.1 301 Moved Permanently
Location: https://example.com/
```

curl 会继续访问：

```text
http://example.com
        ↓
301
        ↓
https://example.com
```

很多情况下：

```bash
curl URL
```

和：

```bash
curl -L URL
```

结果不同。

---

# 21. 下载进度

默认会显示进度条。

如果不想显示：

```bash
-s
```

即：

```text
silent
```

例如：

```bash
curl -s https://example.com
```

---

# 22. 静默模式 + 显示错误

脚本里非常常见：

```bash
curl -sS https://example.com
```

其中：

```text
-s
```

静默。

```text
-S
```

即使静默，也显示错误。

所以：

```bash
-sS
```

非常适合 shell 脚本。

---

# 23. HTTP 错误处理

这是非常重要的一个参数：

```bash
-f
```

或者：

```bash
--fail
```

例如：

```bash
curl -f https://example.com
```

如果服务器返回：

```text
404
500
503
```

curl 会认为请求失败。

脚本里经常：

```bash
curl -fsSL https://example.com/script.sh
```

这个组合你以后会经常看到。

---

# 24. `-fsSL` 到底是什么

例如：

```bash
curl -fsSL https://example.com/install.sh
```

实际上：

```text
-f → HTTP 错误时失败
-s → 静默
-S → 显示错误
-L → 跟随重定向
```

即：

```text
-f
-s
-S
-L
```

这是 Linux 世界非常常见的 curl 组合。

---

# 25. 下载并执行脚本

例如：

```bash
curl -fsSL https://example.com/install.sh | bash
```

原理：

```text
curl
  ↓
HTTP Response Body
  ↓
pipe |
  ↓
bash
```

不过要注意：

> 不要随便执行网上下载的脚本。

更安全的调试方式：

```bash
curl -fsSL https://example.com/install.sh -o install.sh
```

先：

```bash
cat install.sh
```

检查之后：

```bash
bash install.sh
```

---

# 26. 断点续传

使用：

```bash
-C -
```

例如：

```bash
curl -C - -O https://example.com/big.iso
```

如果之前已经下载了一部分，可以尝试从断点继续。

---

# 27. 限速

例如限制到：

```text
1 MB/s
```

```bash
curl --limit-rate 1M URL
```

适合：

```text
服务器下载
大文件
带宽测试
避免占满网络
```

---

# 28. 超时

## 连接超时

```bash
curl --connect-timeout 5 https://example.com
```

表示建立连接最多等待：

```text
5 秒
```

---

## 总超时

```bash
curl --max-time 10 https://example.com
```

整个请求最多：

```text
10 秒
```

生产脚本中经常组合：

```bash
curl \
  --connect-timeout 5 \
  --max-time 30 \
  https://example.com
```

---

# 29. 重试

```bash
curl --retry 3 https://example.com
```

表示失败后尝试重新请求。

常见：

```bash
curl \
  --retry 3 \
  --retry-delay 2 \
  https://example.com
```

适合：

```text
网络抖动
临时 DNS 问题
服务器暂时不可用
CI/CD
脚本
```

---

# 30. 查看 HTTP 状态码

这是 API 调试特别重要的能力。

使用：

```bash
-w
```

或者：

```bash
--write-out
```

例如：

```bash
curl -w "%{http_code}\n" https://example.com
```

返回：

```text
<html>
...
</html>
200
```

如果只想要状态码：

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://example.com
```

结果：

```text
200
```

这个模式非常适合脚本。

---

# 31. 查看更多请求信息

例如：

```bash
curl -w "\nHTTP Code: %{http_code}\nTime: %{time_total}\n" \
  -o /dev/null \
  https://example.com
```

可以获取：

```text
HTTP Code
DNS 时间
连接时间
TLS 时间
首字节时间
总时间
```

常用变量：

```text
%{http_code}
%{time_total}
%{time_connect}
%{time_namelookup}
%{time_starttransfer}
%{size_download}
%{speed_download}
%{url_effective}
```

这对于分析 API 性能非常有用。

---

# 32. Cookie

## 发送 Cookie

```bash
curl -b "sessionid=abc123" https://example.com
```

也可以：

```bash
curl --cookie "sessionid=abc123" https://example.com
```

---

# 33. 保存服务器 Cookie

```bash
curl -c cookies.txt https://example.com/login
```

：

```text
-c
↓
cookie-jar
```

服务器返回：

```http
Set-Cookie: sessionid=abc123
```

curl 保存到：

```text
cookies.txt
```

---

# 34. 使用保存的 Cookie

```bash
curl -b cookies.txt https://example.com/profile
```

因此可以模拟：

```text
登录
 ↓
获得 Cookie
 ↓
访问需要登录的接口
```

例如：

```bash
curl -c cookies.txt \
  -X POST \
  -d "username=tom&password=123" \
  https://example.com/login
```

然后：

```bash
curl -b cookies.txt \
  https://example.com/profile
```

---

# 35. User-Agent

使用：

```bash
-A
```

例如：

```bash
curl -A "Mozilla/5.0" https://example.com
```

即：

```http
User-Agent: Mozilla/5.0
```

也可以：

```bash
curl --user-agent "my-client/1.0" URL
```

---

# 36. Referer

```bash
curl -e "https://google.com" https://example.com
```

发送：

```http
Referer: https://google.com
```

---

# 37. 文件上传

## multipart/form-data

使用：

```bash
-F
```

例如：

```bash
curl -F "file=@test.txt" http://localhost:8000/upload
```

这相当于浏览器：

```html
<form enctype="multipart/form-data">
```

---

# 38. 上传多个字段

```bash
curl \
  -F "username=tom" \
  -F "file=@test.txt" \
  http://localhost:8000/upload
```

FastAPI：

```python
@app.post("/upload")
async def upload(
    username: str,
    file: UploadFile
):
    ...
```

就可以使用这样的 curl 测试。

---

# 39. 上传多个文件

```bash
curl \
  -F "files=@a.txt" \
  -F "files=@b.txt" \
  http://localhost:8000/upload
```

---

# 40. HTTPS

最基本：

```bash
curl https://example.com
```

流程：

```text
DNS
 ↓
TCP
 ↓
TLS
 ↓
HTTP
```

如果 HTTPS 有问题：

```bash
curl -v https://example.com
```

重点观察：

```text
DNS
TCP connect
TLS handshake
certificate
HTTP
```

---

# 41. 不验证 TLS 证书

```bash
-k
```

或者：

```bash
--insecure
```

例如：

```bash
curl -k https://localhost:8443
```

适合：

```text
本地开发
自签名证书
测试环境
```

但不要在生产环境随便使用：

```bash
-k
```

因为这相当于降低 TLS 证书验证安全性。

---

# 42. 指定 CA 证书

如果服务器使用自签名 CA：

```bash
curl --cacert ca.crt https://example.com
```

这比：

```bash
curl -k
```

安全得多。

关系：

```text
-k
↓
关闭证书验证

--cacert
↓
告诉 curl 哪个 CA 可以信任
```

---

# 43. HTTPS 客户端证书

某些服务需要：

```text
mTLS
```

可以：

```bash
curl \
  --cert client.crt \
  --key client.key \
  https://example.com
```

流程：

```text
客户端
  │
  │ client certificate
  ▼
服务器
  │
  │ 验证客户端身份
  ▼
建立 TLS
```

---

# 44. Proxy

这个对你之前 Linux/Clash 网络环境尤其重要。

使用：

```bash
-x
```

或者：

```bash
--proxy
```

例如：

```bash
curl -x http://127.0.0.1:7897 https://example.com
```

也可以：

```bash
curl --proxy http://127.0.0.1:7897 https://example.com
```

---

# 45. HTTP Proxy

例如：

```bash
curl \
  -x http://172.30.112.1:7897 \
  https://example.com
```

这里非常容易搞错：

```text
代理协议是 HTTP
```

所以：

```text
http://172.30.112.1:7897
```

不应该因为目标是 HTTPS 就写成：

```text
https://172.30.112.1:7897
```

即：

```text
目标协议 ≠ 代理协议
```

例如：

```text
curl
 │
 │ HTTP proxy
 ▼
172.30.112.1:7897
 │
 │ HTTPS tunnel
 ▼
https://example.com
```

---

# 46. 环境变量代理

Linux 中非常常见：

```bash
export http_proxy=http://127.0.0.1:7897
export https_proxy=http://127.0.0.1:7897
```

也可以：

```bash
export HTTP_PROXY=http://127.0.0.1:7897
export HTTPS_PROXY=http://127.0.0.1:7897
```

查看：

```bash
env | grep -i proxy
```

---

# 47. 取消代理

```bash
unset http_proxy
unset https_proxy
unset HTTP_PROXY
unset HTTPS_PROXY
```

或者针对单次 curl：

```bash
curl --noproxy '*' https://example.com
```

---

# 48. NO_PROXY

例如：

```bash
export NO_PROXY=localhost,127.0.0.1
```

表示：

```text
localhost
127.0.0.1
```

不要经过代理。

典型开发环境：

```bash
export HTTP_PROXY=http://127.0.0.1:7897
export HTTPS_PROXY=http://127.0.0.1:7897
export NO_PROXY=localhost,127.0.0.1
```

---

# 49. DNS 问题排查

可以：

```bash
curl -v https://example.com
```

观察：

```text
* Host example.com:443 was resolved.
```

如果 DNS 有问题，重点看这里。

---

# 50. 指定 DNS 解析结果

非常实用：

```bash
--resolve
```

例如：

```bash
curl \
  --resolve example.com:443:1.2.3.4 \
  https://example.com
```

意思是：

```text
访问：

example.com:443

但是：

example.com
    ↓
1.2.3.4
```

这对于测试：

```text
Nginx
HTTPS
虚拟主机
CDN
域名切换
服务器迁移
```

非常有用。

---

# 51. 指定 Host

可以：

```bash
-H "Host: example.com"
```

例如：

```bash
curl \
  -H "Host: example.com" \
  http://1.2.3.4/
```

这里：

```text
TCP 连接
    ↓
1.2.3.4

HTTP Host
    ↓
example.com
```

这对 Nginx 虚拟主机调试特别重要。

---

# 52. `--resolve` 和 `Host` 的区别

这是网络调试中值得理解的一个知识点。

### `-H "Host: example.com"`

只改变：

```http
Host:
```

### `--resolve`

同时解决：

```text
DNS
连接目标 IP
TLS SNI
Host
```

所以 HTTPS 测试时：

```bash
curl --resolve example.com:443:1.2.3.4 https://example.com
```

通常比：

```bash
curl -H "Host: example.com" https://1.2.3.4
```

更合理。

---

# 53. 查看 curl 版本

```bash
curl --version
```

例如：

```text
curl 8.x.x
libcurl/8.x.x
OpenSSL/...
zlib/...
Protocols: HTTP HTTPS FTP ...
Features: ...
```

这个信息非常重要。

因为不同 curl 编译版本可能支持不同：

```text
TLS
HTTP/2
HTTP/3
SFTP
Brotli
zstd
IPv6
```

---

# 54. 查看帮助

最常用：

```bash
curl --help
```

或者：

```bash
curl -h
```

查看所有：

```bash
curl -h all
```

查看分类：

```bash
curl -h category
```

查看具体参数：

```bash
curl -h --insecure
```

官方文档特别推荐使用这种方式查询单个选项，而不是直接阅读几千行完整 man page。([Everything Curl](https://everything.curl.dev/cmdline/help.html?utm_source=chatgpt.com "Help - everything curl"))

完整手册：

```bash
curl --manual
```

---

# 55. curl 的配置文件

curl 支持：

```text
~/.curlrc
```

例如：

```text
proxy = http://127.0.0.1:7897
connect-timeout = 5
```

以后：

```bash
curl https://example.com
```

就会自动读取配置。

curl 官方教程说明，curl 启动时会尝试读取用户目录下的 `.curlrc`；也可以通过 `-K/--config` 指定其他配置文件。([GitHub](https://github.com/curl/curl/blob/master/docs/MANUAL.md?utm_source=chatgpt.com "curl/docs/MANUAL.md at master · curl/curl · GitHub"))

---

# 56. 指定配置文件

```bash
curl -K curl.conf https://example.com
```

例如：

```text
# curl.conf

silent
location
fail
header = "Accept: application/json"
```

然后：

```bash
curl -K curl.conf https://example.com
```

---

# 57. JSON API 的完整例子

假设 FastAPI：

```python
@app.post("/users")
async def create_user(user: User):
    ...
```

可以：

```bash
curl \
  -X POST \
  http://localhost:8000/users \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "name": "Tom",
    "age": 20
  }'
```

如果需要 JWT：

```bash
curl \
  -X POST \
  http://localhost:8000/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "name": "Tom",
    "age": 20
  }'
```

---

# 58. SSE 流式接口

对于你做的 FastAPI + SSE / LLM Agent 项目，这个特别重要。

例如：

```bash
curl -N http://localhost:8000/api/chat
```

`-N`：

```text
--no-buffer
```

关闭 curl 输出缓冲。

否则服务器：

```text
chunk
 ↓
chunk
 ↓
chunk
 ↓
chunk
```

可能不能及时显示。

SSE 调试通常：

```bash
curl -N \
  -H "Accept: text/event-stream" \
  http://localhost:8000/api/chat
```

POST SSE：

```bash
curl -N \
  -X POST \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{"message":"你好"}' \
  http://localhost:8000/api/chat
```

---

# 59. 测试 WebSocket

现代 curl 版本也支持 WebSocket 相关能力，但实际 WebSocket 调试时，专门的工具通常更加方便，例如：

```text
websocat
wscat
```

curl 更擅长：

```text
HTTP
HTTPS
SSE
REST API
文件传输
```

---

# 60. 并发请求

curl 支持并行传输。

例如：

```bash
curl --parallel \
  https://example.com/a \
  https://example.com/b \
  https://example.com/c
```

也可以限制并发数量：

```bash
curl \
  --parallel \
  --parallel-max 5 \
  URL1 URL2 URL3
```

适合批量请求。

---

# 61. 多个 URL

```bash
curl \
  https://example.com/a \
  https://example.com/b \
  https://example.com/c
```

也可以利用 URL globbing：

```bash
curl "https://example.com/file[1-10].txt"
```

---

# 62. Range 请求

例如只下载前 1000 字节：

```bash
curl -r 0-999 https://example.com/file
```

对应 HTTP：

```http
Range: bytes=0-999
```

适合：

```text
大文件
断点下载
HTTP Range 测试
```

---

# 63. HTTP 版本

可以指定：

```bash
--http1.1
```

或者：

```bash
--http2
```

或者：

```bash
--http3
```

例如：

```bash
curl --http1.1 https://example.com
```

```bash
curl --http2 https://example.com
```

```bash
curl --http3 https://example.com
```

实际支持哪些协议取决于你的 curl 编译时能力。

查看：

```bash
curl --version
```

---

# 64. IPv4 / IPv6

强制 IPv4：

```bash
curl -4 https://example.com
```

强制 IPv6：

```bash
curl -6 https://example.com
```

排查：

```text
IPv4 能访问
IPv6 不能访问
```

非常有用。

---

# 65. 网络接口

可以指定：

```bash
curl --interface eth0 https://example.com
```

或者：

```bash
curl --interface 192.168.1.100 https://example.com
```

适合多网卡服务器。

---

# 66. 获取响应 Header 到文件

```bash
curl -D headers.txt https://example.com
```

结果：

```text
headers.txt
```

里面：

```http
HTTP/2 200
content-type: text/html
content-length: ...
```

---

# 67. 输出 Body 到文件，Header 到文件

```bash
curl \
  -D headers.txt \
  -o body.html \
  https://example.com
```

这样：

```text
headers.txt
    ↓
响应头

body.html
    ↓
响应体
```

---

# 68. 常见参数总表

| 参数   | 作用                   |
| ---- | -------------------- |
| `-X` | 指定 HTTP Method       |
| `-H` | 设置 Header            |
| `-d` | POST 数据              |
| `-F` | multipart/form-data  |
| `-G` | 将 `-d` 数据作为 Query 参数 |
| `-o` | 指定输出文件               |
| `-O` | 使用 URL 文件名           |
| `-I` | HEAD                 |
| `-i` | 显示响应 Header          |
| `-v` | Verbose              |
| `-s` | Silent               |
| `-S` | Silent 模式显示错误        |
| `-f` | HTTP 错误返回失败          |
| `-L` | 跟随重定向                |
| `-k` | 不验证 TLS 证书           |
| `-u` | Basic Auth           |
| `-b` | Cookie               |
| `-c` | 保存 Cookie            |
| `-A` | User-Agent           |
| `-e` | Referer              |
| `-x` | Proxy                |
| `-4` | IPv4                 |
| `-6` | IPv6                 |
| `-I` | HEAD                 |
| `-C` | 断点续传                 |
| `-r` | Range                |
| `-w` | 输出统计信息               |
| `-N` | 不缓冲输出                |
| `-q` | 不读取默认 curl 配置        |
| `-K` | 指定配置文件               |

---

# 69. 后端开发最重要的 curl 组合

如果你主要做：

```text
FastAPI
React
Nginx
REST API
LLM API
SSE
```

我建议重点掌握下面这些。

### GET

```bash
curl http://localhost:8000/users
```

### POST JSON

```bash
curl \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"name":"Tom"}' \
  http://localhost:8000/users
```

### JWT

```bash
curl \
  -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/users
```

### 文件上传

```bash
curl \
  -F "file=@test.pdf" \
  http://localhost:8000/upload
```

### 查看详细请求

```bash
curl -v http://localhost:8000/users
```

### 跟随重定向

```bash
curl -L http://example.com
```

### HTTP 状态码

```bash
curl -s -o /dev/null -w "%{http_code}\n" URL
```

### SSE

```bash
curl -N \
  -H "Accept: text/event-stream" \
  URL
```

### Proxy

```bash
curl \
  -x http://127.0.0.1:7897 \
  https://example.com
```

### DNS/IP 调试

```bash
curl \
  --resolve example.com:443:1.2.3.4 \
  https://example.com
```

---

# 70. curl 排查网络问题的思维体系

这个比记参数更加重要。

当：

```bash
curl https://example.com
```

失败时，不要一上来乱加参数。

应该按照网络层次排查：

```text
                    curl 请求
                       │
                       ▼
                 ┌───────────┐
                 │ URL 解析  │
                 └─────┬─────┘
                       │
                       ▼
                 ┌───────────┐
                 │ DNS       │
                 └─────┬─────┘
                       │
                       ▼
                 ┌───────────┐
                 │ TCP       │
                 └─────┬─────┘
                       │
                       ▼
                 ┌───────────┐
                 │ TLS       │
                 └─────┬─────┘
                       │
                       ▼
                 ┌───────────┐
                 │ HTTP      │
                 └─────┬─────┘
                       │
                       ▼
                 ┌───────────┐
                 │ Response  │
                 └───────────┘
```

所以：

### 第一步

```bash
curl -v URL
```

### 第二步

检查：

```text
DNS
```

### 第三步

检查：

```text
TCP
```

### 第四步

检查：

```text
TLS
```

### 第五步

检查：

```text
HTTP status
```

### 第六步

检查：

```text
Response body
```

---

# 71. curl 最重要的几个组合

建议你直接记住：

```bash
# 普通请求
curl URL

# 查看 Header
curl -I URL

# 查看完整请求过程
curl -v URL

# POST JSON
curl -H "Content-Type: application/json" \
     -d '{"key":"value"}' \
     URL

# 跟随重定向
curl -L URL

# 下载文件
curl -O URL

# 下载并指定文件名
curl -o file URL

# 忽略 TLS 证书
curl -k URL

# 使用代理
curl -x http://127.0.0.1:7897 URL

# 查看 HTTP 状态码
curl -s -o /dev/null -w "%{http_code}\n" URL

# SSE
curl -N URL

# 网络详细调试
curl -v URL

# 保存 Cookie
curl -c cookies.txt URL

# 使用 Cookie
curl -b cookies.txt URL

# 文件上传
curl -F "file=@test.txt" URL
```

---

# 72. 最后建立一个 curl 知识框架

如果你是为了 Linux + 后端开发系统学习，我建议把 `curl` 按下面这个知识树掌握：

```text
curl
│
├── 1. 基础
│   ├── URL
│   ├── options
│   ├── short option
│   └── long option
│
├── 2. HTTP
│   ├── GET
│   ├── POST
│   ├── PUT
│   ├── PATCH
│   └── DELETE
│
├── 3. HTTP Headers
│   ├── -H
│   ├── Content-Type
│   ├── Authorization
│   ├── User-Agent
│   └── Cookie
│
├── 4. HTTP Body
│   ├── -d
│   ├── --data-raw
│   ├── --data-binary
│   ├── --data-urlencode
│   └── -F
│
├── 5. 文件
│   ├── -o
│   ├── -O
│   ├── -C
│   ├── -r
│   └── -F
│
├── 6. HTTP Response
│   ├── -I
│   ├── -i
│   ├── -D
│   ├── -w
│   └── -f
│
├── 7. HTTPS
│   ├── TLS
│   ├── CA
│   ├── --cacert
│   ├── -k
│   ├── --cert
│   └── --key
│
├── 8. Authentication
│   ├── Basic Auth
│   ├── Bearer Token
│   ├── API Key
│   └── Client Certificate
│
├── 9. Cookie
│   ├── -b
│   ├── -c
│   └── Cookie Jar
│
├── 10. Proxy
│   ├── -x
│   ├── HTTP Proxy
│   ├── HTTPS Proxy
│   ├── SOCKS
│   └── NO_PROXY
│
├── 11. DNS / Network
│   ├── -4
│   ├── -6
│   ├── --resolve
│   ├── --interface
│   └── --connect-to
│
├── 12. Debug
│   ├── -v
│   ├── --trace
│   ├── --trace-ascii
│   └── --write-out
│
├── 13. Performance
│   ├── timeout
│   ├── retry
│   ├── rate limit
│   ├── parallel
│   └── HTTP/2
│
└── 14. 自动化
    ├── shell script
    ├── exit code
    ├── .curlrc
    ├── -K
    └── CI/CD
```

**真正掌握 curl 的关键不是记住几百个参数，而是把它理解成一个“可以手工控制 HTTP/网络请求每个环节的客户端”。** 官方 curl 文档本身也提供了按选项、协议、调试和使用场景组织的完整资料。([Everything Curl](https://everything.curl.dev/cmdline/options/index.html?utm_source=chatgpt.com "Command line options - everything curl"))

如果你接下来是为了**Linux/后端开发**学习，我建议下一步专门深入一遍 **`curl + HTTP 协议`**：把一次 `curl -v` 输出逐行拆开，从 **DNS → TCP 三次握手 → TLS → HTTP Request → HTTP Response → Keep-Alive → Connection 复用**完整串起来。这样 curl 就不再只是“命令参数记忆”，而会和你前面学习的 Linux 网络、防火墙、Nginx、FastAPI、代理真正连成一套。
