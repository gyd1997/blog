基于ABNF重新学习HTTP报文格式

# 前言

HTTP作为我们熟知的网络协议，熟悉它的报文格式是非常必要的，而网络上常见的文章的描述，大多都不太严谨，本文将从官方规范来解析HTTP的报文格式。

在百度搜索“HTTP报文格式”，选择非常靠前的一篇，他的描述是这样的：

![](D:\blog\docs\md\计算机网络\img\网图.png)

在网络上常见的描述都是类似这幅图，而图中有很多不严谨的地方。以下就来揭晓，严谨、规范的 HTTP 报文格式究竟是什么样的。

# RFC & ABNF

**什么是RFC？**

HTTP 协议的标准是由万维网协会（W3C）、互联网工程任务组（IETF）协调制定，最终发布了一系列的RFC。

RFC（Request For Comments，译为：请求意见稿）。

在 **[RFC5234](https://datatracker.ietf.org/doc/html/rfc5234)** 中有提到，HTTP 现在使用了 ABNF 来描述语法。由此可见，ABNF 是最严谨的 HTTP 报文格式描述形式，脱离 ABNF 谈论 HTTP 格式，往往是片面的、不严谨的。

**什么ABNF？**

> 巴科斯范式的英文缩写为BNF，它是以美国人巴科斯(Backus)和丹麦人诺尔(Naur)的名字命名的一种形式化的语法表示方法，用来描述语法的一种形式体系，是一种典型的元语言。又称巴科斯-诺尔形式(Backus-Naur form)。它不仅能严格地表示语法规则，而且所描述的语法是与上下文无关的。它具有语法简单，表示明确，便于语法分析和编译的特点。

> RFC2234 定义了**增加型巴科斯范式(ABNF)**。近年来在Internet的定义中 ABNF 被广泛使用。ABNF 做了更多的改进。增强型巴科斯-瑙尔范式(ABNF)基于了巴科斯-瑙尔范式(BNF)，但由它自己的语法和推导规则构成。这种元语言的发起原则是描述作为通信协议(双向规范)的语言的形式系统。它建档于 RFC 4234 中通常充当 IETF 通信协议的定义语言。

# ABNF的描述

首先给出一些 ABNF 中用到的规则字符：

| 规则  | 形式定义 | 意义                |
| ----- | -------- | ------------------- |
| DIGIT | %x30-39  | 数字（0~9）         |
| SP    | %x20     | 空格                |
| CRLF  | CR LF    | 互联网标准换行      |
| HTAB  | %x09     | 横向制表符（TAB键） |
| VCHAR | %x21-7E  | 可见（打印）字符    |
| OCTET | $x00-FF  | 8位数据             |

我们来看 ABNF 对 HTTP 报文的基本描述：

```
HTTP-message = start-line
			   *(header-field CRLF)
			   CRLF
			   [message]
```

> 注意这里的换行是为了方便阅读，并不是它有真正的换行符，如果有换行会在描述中体现。

这一段就是 ABNF 对于 HTTP 的描述，可以看出来内容并不多。这只是一个简单的概括，详细的内容下文再来分析。

首先我们先看一些符号的意义：

| 符号 | 意义                                                |
| ---- | --------------------------------------------------- |
| /    | 任选一个                                            |
| *    | 0 个或多个。如：2\* 表示至少 2 个，3\*6 表示 3~6 个 |
| ()   | 组成一个整体                                        |
| []   | 可选（可有可无）                                    |

---

```
start-line = request-line / status-line
```

`start-line` 表示开始行，它可以是 请求行 或者 状态行。

```
 *(header-field CRLF)
```

`header-field` 表示消息头的一行内容，如 `Content-Type: text/javascript`。每一行内容后有一个CRLF（回车换行），一行内容与一个 CRLF 构成一个整体，这个整体可以是 0个 或 多个。

```
CRLF
```

在 0个  或 多个 `header-field CRLF` 后，有专门的一个 `CRLF`。

```
[message]
```

`message` 表示消息体，它是可选的，也就是说是可有可无的。对请求报文来说，消息体就是请求体；对响应报文来说，消息体就是响应体。

---

所以说，一个HTTP报文格式应该包含如下内容：

- `start-line` 开始行（对应请求行或响应行）
- ` *(header-field CRLF)` 0 个或 多个 请求头的一行内容
- `CRLF` 一个 CRLF
- `[message]`可选的消息体

# 更为详细的内容

上文对 报文格式 的整体进行了描述，而里面的 `start-line`，`header-field`还没有详细的解释，下面我们再来看这些更为详细的内容。

## start-line

```
start-line = request-line / status-line
```

**request-line**：

```
request-line = method SP request-target SP HTTP-version
```

```
HTTP-version = HTTP-name "/" DIGIT "." DIGIT
```

```
HTTP-name = %x48.54.54.50;HTTP
```

请求行的组成部分包括 ：

- method：请求方法（GET、POST等）
- SP：空格
- request-target：请求目标
- HTTP-version：HTTP版本

`HTTP-version`的描述为：

HTTP-name   一个 / 字符   一个数字  一个 . 字符   一个数字

`HTTP-name` 的描述为：

`%x48.54.54.50;HTTP`，其中`%x48.54.54.50`表示 ASCII 码的4个字符，`;HTTP`中;号是ABNF的注释。所以说，HTTP-name 其实就是 HTTP 字符。

再来回顾，请求行就是：

> 请求方法    空格    请求目标    空格   HTTP版本

举个例子：

```
GET /hello/ HTTP/1.1
```

这正好和我们上面的描述一一对应了起来。

## status-line

```
status-line = HTTP-version SP status-code SP reason-phrase CRLF
```

```
status-code = 3DIGIT
```

```
reason-phrase = *(HTAB / SP / VCHAR / obs-text)
```

相应报文的第一行，我们叫它状态行，它的内容包括：

- HTTP-version：HTTP版本，同请求行
- SP：一个空格
- status-code：状态码
- reason-phrase：原因短语

`status-code`就是状态码，使用3位数字，也就是我们常见的 404、200等。

`reason-phrase`是原因短语，它可能是0个或多个，内容可能是多样的。一般来说原因短语会是状态码对应的描述，如 200的 `OK`，404的`Not Found`。

综上，状态行就是：

> HTTP版本     空格     3位状态码     空格 0个或多个原因短语   CRLF

举个例子：

```
HTTP/1.1 200 OK
```

也正好和我们上面的一一对应。

## header-field

```
header-field = field-name ":" OWS field-value OWS
```

```
OWS = *(SP / HTAB)
```

`header-field` 表示一个消息头的一行内容（请求头或响应头）。

`OWS` 表示 0个或多个 的 空格或TAB。

所以一个 `header-filed`描述为：

> 字段名 ：OWS 字段值 OWS

举个例子：

```
Content-Type: text/javascript
```

## message-body

```
message-body = *OCTET
```

消息体（请求体或响应体）的描述为0个或多个 OCTET，OCTET就是一个字节。也就是说，消息体可以是0个或多个字节。所以说，消息体既可以为空，也可以是字符串，也可以是图片数据，只要是字节就可以。

# 总结

综上，我们已经基于 ABNF 重新严谨、规范的学习了一遍 HTTP 的报文格式，相信通过本文你一定对 HTTP的报文格式已经有了深入且标准的认识。
