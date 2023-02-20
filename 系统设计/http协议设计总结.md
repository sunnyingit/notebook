HTTP笔记

## HTTP概述

HTTP协议是基于TCP/IP协议的应用层协议。主要规定客户端和服务端通信的格式，默认端口是80。

支持HTTP协议的客户端通常指的是浏览器，支持HTTP协议的服务器有很多，常用的服务器有nginx。

客户端与服务器端相互通信，必须关注如下几个问题：

1. 资源信息，例如资源位置，类型，长度，编码等
2. 资源传送效率，例如资源压缩，续传，缓存，持久链接，管道化
3. 资源传送安全，例如跨域，报文加密，服务器认证，报文完整性保证
4. 客户端和服务端状态维护，例如cookie，session
5. 请求响应结果，例如状态码

HTTP协议就是为了解决以上问题。


## HTTP报文
用于HTTP协议交互的信息被称为HTTP报文，请求端的HTTP报文叫做请求报文，响应端的报文称为响应报文。

HTTP协议规定了请求报文和响应的格式。

请求报文格式如下：

```
request_method uri http_version     # 请求方法 资源地址 http版本
request_headers                     # 请求头部信息
空行(CR+LF)                         # 空行 (解析请求数据流时读取到空行说明请求头部解析完成)
request_body                        # 请求参数
```

响应报文格式如下：

```
status_code http_version message    # 响应状态 http版本 响应消息
response_headers                    # 响应头部信息
空行(CR+LF)                         # 空行解析响应数据流时读取到空行说明响应头部解析完成
response_body                       # 响应数据
```

一个完整请求和响应的实例如下：

```
// 请求
GET /chat HTTP/1.0
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_5)
Host: www.baidu.com
// 空行
// 请求参数
uid:1

// 响应
HTTP/1.0 200 OK
Content-Type: text/plain
Content-Length: 137582

// 空行
<html>
  <body>Hello World</body>
</html>
```

## 请求行和响应行
HTTP支持的请求方法有：GET，POST，DELETE，HEAD，OPTIONS，PUT，HEAD，在RestFull Api设计中，常使用到GET，POST，DELETE，PUT。

HTTP目前主流版本有：HTTP/1.0，HTTP/1.1， 目前默认http版本是HTTP/1.1。

常见响应状态码：
1. 200 OK 请求成功
2. 204 No Content 服务端没有返回信息，在RestFull 设计中可用于delete，update操作的返回状态
3. 206 Partial Content 返回部分信息 在RESTFull设计中可用于获取资源一部分信息的响应状态
4. 301 Moved Permanently
5. 302 Found 如果服务器端返回此状态，一般会响应头里面有Location:uri 进行重定向
6. 303 See Other 和302状态码一样的功能，只是明确了请求重定向的方法必须是GET
7. 304 Not Modified 服务器端的资源没有变化（缓存）
8. 400 Bad Reqeust 请求报文有误，在RestFull设计中，当API请求参数有误，可返回此参数
9. 401 UnAuthorized 未授权 在RESTFull设计中，未登录的状态下请求需要登录的API，可返回此状态
10. 403 Forbidden 服务器拒绝此请求并不告知具体原因，一般是因为权限不够导致
11. 404 Not Found  资源不存在，在RestFull设计中，如果请求的资源不存在，可以返回404
12. 500 Internal Server Error 服务器挂了，在RESTFull设计中，如果服务器处理有问题，则可以返回500
13. 503 Service Unavailable 停机维护


## 请求参数实体

常用的请求方法有GET, POST。 GET和POST主要区别是幂等性。

对于GET请求来讲，请求参数拼接在请求的URL中。

对于POST请求来讲，有常见的4种请求格式:

### application/x-www-form-urlencoded

这应该是最常见的 POST 提交数据的方式，表单如果不设置 enctype 属性，就会以 application/x-www-from-urlencoded 方式提交数据url
```
POST http://www.example.com HTTP/1.1
Content-Type: application/x-www-form-urlencoded;charset=utf-8

title=test&sub%5B%5D=1&sub%5B%5D=2&sub%5B%5D=3
```

首先，Content-Type 被指定为application/x-www-form-urlencoded，其次，提交的数据按照 key1=val1&key2=val2 的方式进行编码，key 和 val 都进行url转码

### multipart/form-data

使用表单上传文件时，必须让表单的 enctype 等于multipart/form-data

### application/json

现在越来越多的人把它作为请求头，用来告诉服务端消息主体是序列化后的 JSON 字符串：
```

POST http://www.example.com HTTP/1.1
Content-Type: application/json;charset=utf-8

{"title":"test","sub":[1,2,3]}
php 无法通过 $_POST 对象从上面的请求中获得内容，需要从php原始字节流获取数据，服务器端接受到的其实是字符串

```

php判断是否为json请求：

```

public function isJson()
{
    return str_contains($_SERVER['CONTENT_TYPE'], '/json');
}
```

### text/xml
它是一种使用 HTTP 作为传输协议，XML 作为编码方式的远程调用规范，目前用的不多。


## HTTP 请求和响应首部

HTTP请求首部是客户端告知服务端请求报文的相关信息

常用指令如下：

Accept: 通知服务器端，客户端能处理的文件类型(MIME) Accept: text/html, text/plain, text/css, image/jpeg

Accept-Charset: 通知服务器端，客户端能处理的字符编码(unicode utf8) Accept-Charset: iso-8895-8, unicode-1-1;q=0.8

Accept-Encoding: 通知服务器端，客户端能处理的内容编码(内容压缩算法) Accept-Encoding: gzip, compress

Accept-Language: 通知客户端，客户端能处理的语言(中文，英文) Accept-Language: zh-cn, zh

Host: 在一台主机上可以虚拟出多台域名不同的主机(通过DNS或者配置/etc/hosts文件)，host告知服务器请求的对应虚拟服务器 Host:www.hafefd.com

Referer: 告知服务器，本次请求是从哪个页面发起的，如果这个页面有些敏感信息，则可能导致信息泄露，再者Referer的准确拼写是Referrer，验证Referer可以一定程度上放在csrf攻击 Referer: http://www.fads.jp/index.html

User-Agent: 告知服务器，发起请求的浏览器信息 User-Agent: Mozilla/5.0

Connection：告知服务器端和客户端保持持久连接，在HTTP/1.1版本中，默认开启 Connection: Keep-alive

Date: 创建请求报文的时间 Date: Fri, 25 Jan 2019 12:31:00 GMT

Upgarde: 告知服务器协议升级，如果http协议升级到websocket协议，则需要此参数 Upgrade: Web-Socket


## HTTP响应首部

响应头部是服务器端告知客户端响应报文的相关信息。常用指令如下：

Connection：告知服务器端和客户端保持持久连接，在HTTP/1.1版本中，默认开启 Connection: Keep-alive | close

Date: 创建请求报文的时间 Date: Fri, 25 Jan 2019 12:31:00 GMT

Content-Encoding: 告知客户端，服务器对响应实体的【主体】使用的编码方式 Content-Encoding: gzip

Content-Language: 告知客户端实体使用的语言 (中文，英文) Content-Languaget: zh-CN

Content-Type: 告知客户端实体对象的类型(MIME) Content-Type:Application/json

Location: 指定浏览器的跳转链接 Location: http://fis.com

Server: 告知客户端服务器的信息 Server: nginx/1.1

Transfer-Encoding:告知客户端服务器数据已经传送完毕。 传输编码 Transfer-Encoding: chunked

Content-Length: 告知客户端响应实体数据的长度 Content-Length: 19885

### HTTP缓存控制

HTTP缓存是指代理服务器(CDN)或者客户端(浏览器)本地磁盘内保存的资源副本。

当代理服务器缓存资源后，可以减少对源服务器的通信流量和通信时间。在web开发中css，js文件资源可以被浏览器和代理服务器缓存。

当浏览器缓存了资源后，可以支持本地【前进】【后退】功能，减少对代理服务器的通信次数。

总之，需要理解的是，客户端和服务端都可以通过指令去控制缓存，这个指令就是Cache-Control。 Cache-Control指令的参数是可选的，多个指令时间使用,分割，例如Cache-Control:no-cache，max-age=0，客户端和服务器端分别有不同的参数。


### 客户端缓存请求参数

客户端缓存请求参数，主要是为了告知服务器端(一般是代理服务器CDN)缓存处理逻辑

no-cache: 表示客户端不会【接受】缓存过的资源，代理服务器(CDN)必须强制向源服务器发起请求拿到最新数据。

no-store: 表示不要缓存【请求数据】，一般如果请求里面有机密数据，可以使用no-store指令告知服务器

max-age: max-age 指定的是缓存有效时间

当缓存时间没有超过max-age指定的值(单位是秒)，就把它给我，否则代理服务器(CDN)必须强制向源服务器发起请求拿到最新数据。所以max-age=0则表示代理服务器通常需要重新向源服务器请求资源,

min-fresh: 当缓存【再】过min-fresh指定的值(单位是秒)，如果还没有过期，则把它返回给客户端，例如min-fresh=60表示 再过一分钟，缓存还没有失效，就把缓存给我

only-if-cached: 只有代理服务器本地缓存了目标资源，才返回，否则不返回。

must-revalidate: 要求代理服务器必须向源服务器再次验证即将返回的资源是否过期，如果代理代理服务器无法连接到源服务器，则返回状态码504(gateway timeout)

no-transform: 要求代理服务器不可以改变请求主体的类型，这样可以防止缓存或代理服务器压缩图片等操作。


### 服务器端缓存响应参数

服务器端缓存响应参数告知客户端缓存处理逻辑，这里客户端分为两种情况:

服务器端(源服务器) --------> 客户端(缓存代理服务器)
服务器端(缓存代理服务器) -------> 客户端(浏览器)
no-cache: 不使用过期的资源。告诉客户端，不论本地缓存是否过期，都不能直接使用本地缓存。换句话说，在使用本地缓存之前，必须先通过 ETag 或 Last-Modified 向服务器发起二次验证，如果服务器响应 304则可用该本地缓存，否则不可。 千万注意，并不是不缓存资源

no-store: 告知客户端，不要缓存资源。

public: 表明响应的数据主体可以被任何对象（包括：发送请求的客户端，代理服务器，等等）缓存

private: 表明响应只能被单个用户缓存，不能作为共享缓存（即 CDN 或代理服务器不能缓存它）, 只能被浏览器缓存

must-revalidate: 缓存必须在使用之前验证旧资源的状态，并且不可使用过期资源

no-transform: 要求客户端不可以改变响应主体的类型

max-age: 告知客户端，不超过max-age指定的时间，就不需要向服务器验证，也就是告知客户端可以缓存多长时间，超过这个值后就需要重新验证了


### Etag指令和缓存验证过程
tag: 告知客户端资源的唯一标识，服务器会为每一份资源都设置一个唯一的tag etag: "vnAm1Q"

nginx中Etag的相关配置如下：

Syntax:     etag on | off;
Default:    etag on;
Context:    http, server, location
This directive appeared in version 1.3.3.
Enables or disables automatic generation of the ETag response header field for stat
ic resources.
在上文中提到，客户端(包括代理服务器)都会验证缓存是否过期

假定在首次获取资源 120 秒后，浏览器又对该资源发起了新的请求。首先，浏览器会检查本地缓存并找到之前的响应。遗憾的是，该响应现已过期，浏览器无法使用。此时，浏览器可以直接发出新的请求并获取新的完整响应。不过，这样做效率较低，因为如果资源未发生变化，那么下载与缓存中已有的完全相同的信息就毫无道理可言！

这正是验证令牌（在 ETag 标头中指定）旨在解决的问题。服务器生成并返回的随机令牌通常是文件内容的哈希值或某个其他指纹。客户端不需要了解指纹是如何生成的，只需在下一次请求时将其发送至服务器。

浏览器会替我们完成所有工作：它会自动检测之前是否指定了验证令牌，它会将验证令牌追加到发出的请求上，并且它会根据从服务器接收的响应在必要时更新缓存时间戳。我们唯一要做的就是确保服务器提供必要的 ETag 令牌。

If-Match: 形如If-xxxx这样的请求首部字段，都可以称为条件请求，服务器接受到这样的请求后，只有判断条件为真时，才会执行请求， 否则会返回412 （Precondition Failed）

If-Match:'122345': 匹配Etag=122345的实体值


## HTTP续传
所谓断点续传, 也就是要从文件已经下载的地方开始继续下载。类似于大文件下载如果支持断点续传，会有更好的用户体验。

请求首部指令： Range: 指定第一个字节的位置和最后一个字节的位置 Range: bytes=100-1212 #告知服务器，请求100-1212的字节

响应首部指令： Content-Range: 告知客户端，响应的实体的那部分范围是符合请求的范围，字段值以字节为单位, 同时还需要告知客户端整个实体的大小

Content-Range: bytes=1-1212/10000



## HTTP COOKIE

cookie是网站为了辨别用户身份而储存在用户终端(浏览器)上的数据。

比如用户登录网站成功后，服务器发送一个指令，要求用户使用的浏览器记录一份数据(cookie), 下次用户访问该网站时，浏览器会【自动】把对应的cookie发送到网站，网站就可以识别登录用户，不需要用户重复登录。

注意，因为同源策略的限制，浏览器不会把A网址的cookie发送给B网址。

常用指令如下：

Set-Cookie: 服务器告知客户端设置对应的cookies信息 Set-Cookie: expires=true, 05 GMT; path=/; deomain=.hacker.jp; HttpOnly; qwerty=219ffwef9w0f;

<cookie-name>=<cookie-value>: 一个 cookie 开始于一个名称/值对, 多个cookie使用";"分割。例如: qwerty=219ffwef9w0f;uid=11111

expires: cookie在保存在客户端的有效时间。在代理服务器中，需要关注服务器自身的时区设定。否则cookie无效。

path: 指定一个URL路径，这个路径必须出现在要请求的资源的路径中才可以发送cookie。 例如，path=/docs, 那么 "/docs", "/docs/Web/" 或者 "/docs/Web/HTTP" 都满足匹配的条件，皆可以发送cookie, 请求urll是"/file"，则不会发送cookie

Domain: 指定 cookie可以送达的主机名。假如没有指定，那么默认值为当前文档访问地址中的主机部分（但是不包含子域名）,不指定域名更安全

HttpOnly: 设置了 HttpOnly 属性的 cookie 不能使用 JavaScript 经由Document.cookie 属性、XMLHttpRequest 和 Request APIs 进行访问，以防范跨站脚本攻击（XSS）

Secure: 一个带有安全属性的 cookie 只有在请求使用SSL和HTTPS协议的时候才会被发送到服务器。然而，保密或敏感信息永远不要在 HTTP cookie 中存储或传输

## HTTP 跨域请求

首先必须了解[同源策略](!http://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html)

浏览器有同源策略的限制, 来源是A服务器的网页中的javascript不可以发送AJAX请求到B服务器。

同源策略，是为了保证用户信息的安全，防止恶意的网址窃取用户信息。

设想这样一种情况：A网站是一家银行，用户登录以后，又去浏览其他网站。如果其他网站可以读取A网站的 Cookie，会发生什么。

但有这样的一种使用场景，网站采用前后端分离的部署方式，前端服务器负责静态资源的响应(包括js，css)，后端服务器负责提供api接口，那么前端js发起ajax请求到后端服务器，必定需要跨域请求。

HTTP协议规定一些指令用于绕开同源策略，允许浏览器请求跨域的接口。

对客户端来讲, 需要添加指令告知服务器端是否允许某个【域】请求服务器的资源。


如果服务器同意了请求，也需要通过指令告知客户端，服务器端常用指令如下：

Access-Control-Allow-Origin: origin | *:指定了允许访问该资源的域名。”*“表示允许来自所有域的请求。 Access-Control-Allow-Origin: http://mozilla.com 将允许来自 http://mozilla.com 的请求

Access-Control-Allow-Credentials:该字段可选。它的值是一个布尔值，表示是否允许发送Cookie。默认情况下，Cookie不包括在CORS请求之中。设为true，即表示服务器明确许可，Cookie可以包含在请求中，一起发给服务器。这个值也只能设为true，如果服务器不要浏览器发送Cookie，删除该字段即可。 如果浏览器发送了cookie， 但是服务端端响应中未携带Access-Control-Allow-Credentials，则浏览器不会把响应的内容返回给请求者 Access-Control-Allow-Credentials: true

Access-Control-Allow-Methods:首部字段用于预检请求的响应。其指明了实际请求所允许使用的 HTTP 方法。 Access-Control-Allow-Method: PUT

Access-Control-Expose-Headers: 在跨域访问中，浏览器只能拿到一些最基本的响应头信息，例如Cache-Control, 如果还需要拿一些其他响应头，则需要设置服务器的此值

Access-Control-Expose-Headers: X-My-Custom-Header, X-Another-Custom-Header



## HTTPS
我们都知道HTTPS能够加密信息，以免敏感信息被第三方获取。所以很多银行网站或电子邮箱等等安全级别较高的服务都会采用HTTPS协议， 过程如下：
1. 客户端向服务端发送请求。
2. 服务端发送公钥(证书)给客户端。
3. 客户端收到公钥后验证公钥是否合法，如果合法则生成一个随机数并使用公钥对随机数加密传输给服务端
4. 服务端收到加密的随机数后，使用私钥解密得到随机数
5. 后续双发传递数据都可以使用这个随机数来加密和解密了(对称加密算法)

## Web服务安全

### xss（cross-site scripting） 跨站脚本攻击
利用网站漏洞，注入非法的html或者js，当用户浏览被注入的网页时，攻击就会发生，可能会造成如下影响:

1. 利用虚假的输入表单骗取用户个人信息
2. 利用js获取用户cookie

防止措施：

1. 严格验证用户输入 php可使用htmlspecialchars，不允许输入html或者js
2. cookie可以设置HttpOnly属性，不允许js获取到cookie信息
3. 开启X-Xss-Protection， 用于控制浏览器xss开关。 例如 X-Xss-Protection: 1开启xss保护开关

### sql注入
通过运行非法的sql产生的风险, 非法的sql往往是因为服务器端代码没有校验用户输入导致。

防止措施：

1. 使用占位符方式, 占位符避免了sql拼接
2. 严格校验用户输入

### csrf 跨站点请求伪造攻击 （cross-site reqeust forgeries）

通过设置好的陷阱, 强制对已经登录用户进行非预期的个人信息等某些状态的更新。

比如用户登录了A网站，攻击者发送一个伪装的链接(此链接是访问A网站，并删除用户的购买信息)点击此链接后，因为用户处于登录状态，所以会删除用户在A网站的购买记录。但用户并不期望删除他的购买记录。

防止措施:

由于CSRF的本质在于攻击者欺骗用户去访问伪造的URL，验证URL是否合法，那么攻击者就无法再运行CSRF攻击。
一般采用TOEKN的方式验证是否正常访问URL。


验证TOEKN的过程和注意事项如下：

1. 用户完成登录后，服务器使用登录用户信息(uid)，通过算法生成唯一token (保证每个用户生成的token都不一样)，然后通过set-cookie把token放在cookie中

2. 在提交Ajax请求，或者表单前，通过js获取到cookie中的token，把token动态的加入到请求参数中(或者把token加入到请求头信息里面)

3. 服务器获取到URL参数中的token(或者请求头) 和cookie上传的token对比，判断是否为正常请求

4. Cookie不能设置为HTTP Only，因为需要被js访问，因为同源策略的限制，伪造的请求是无法获取到cookie，所以伪造请求是无法把cookie中的token加入到请求参数里面，点击伪造请求时，浏览器会把同源下的cookie都发送给服务器，但是伪造请求参数里面并没有token，所以验证失败

5. HTTP头中有一个Referer字段，这个字段用以标明请求来源于哪个地址。在处理敏感数据请求时，通常来说，Referer字段应和请求的地址位于同一域名下。以上文银行操作为例，Referer字段地址通常应该是转账按钮所在的网页地址，应该也位于www.examplebank.com之下。而如果是CSRF攻击传来的请求，Referer字段会是包含恶意网址的地址，不会位于www.examplebank.com之下，这时候服务器就能识别出恶意的访问。

6. 把token放在cookie里面，如果网站有xss漏洞，攻击者依然可以获取到cookie的信息，更好的方式是把token保存在redis中(如果网站有多台服务器则token不能保存在session中)，在渲染表单页面时，把token设置到表单的hidden字段中，对于ajax请求，发起ajax请求前，需要在执行一次请求，获取token，然后把token拼接到url中


## 报文劫持
运营商可能会劫持响应报文，并在报文中加入以下非用户期待的信息(例如广告)，还有可能添加任意的首部信息例如Location:http://bad.com, 诱导用户到非法网站。

使用HTTPS可以防止报文劫持，因为HTTPS的报文都是加密过的，无法篡改报文。

开启X-Frame-Options: SAMEORIGIN 可以减少点击劫持（Clickjacking）而引入的一个响应头


## 管道和持久链接
管道技术允许客户端并行向服务器发起请求。持久链接保证tcp链接的复用，不需要频繁开启新链接。
