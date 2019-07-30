---
title: 'Cookie, localStorage, sessionStorage和 session'
date: 2019-07-30 21:59:20
tags:
- cookie
- storateAPI
- session
- 网络
- 网络安全
categories:
- 网络
---

我今天要是不学习一下的话会焦虑致死的，所以即使上完班回来收拾完都十点了，我还要开电脑o(╥﹏╥)o

今天复习一下之前搞过的Cookie、StorageAPI 和 Session

由于HTTP协议是无状态的，所以每次用户的请求到达服务器的时候，HTTP服务器并不知道这个用户是谁，是否登录过。现在服务器知道我们是否已经登录过，是因为服务器在用户登录的时候设置了浏览器的Cookie，而Session则是借助Cookie实现的一种更高层次的服务器和浏览器之间的对话。

所以，简单来说，cookie 和 session 的作用，就是为了在无状态的HTTP协议上维护会话的状态。

## cookie

### 1. 字段

服务器在请求的response header中，向客户端设置 cookie：

`Set-Cookie: <cookie-name>=<cookie-value>;(可选参数1);(可选参数2)`

参数字段：

`Expires`：cookie的最长有效时间，若不设置则Cookie生命周期和会话期相同

`Max-Age`：cookie生成后失效的秒数

`domain`：指定cookie可以送达的主机名，若设置的是一级域名，那么二级域名也可以获取对应的域名

`path`：指定特定domain下特定path的请求才可以携带这个cookie

`Secure`：如果设置了这个字段，就代表必须使用 SSL 或者 HTTPS 协议的时候，cookie才能被发回服务器

`Http-only`：如果设置了这个字段，就代表浏览器不能用 javascript 获取 cookie 对象（document.dookie)

### 2. 访问方式

cookie的访问方式有两种

- 通过js访问，document.cookie

  返回当前页面的可用cookie，但是不安全，容易被黑客窃取cookie信息，可以通过生成cookie时设置 HttpOnly 属性为 true， 限制cookie只能在 http 发送请求时携带

- 浏览器发送请求时自动携带

  设置cookie时会设置 domain 域名，当用户向这个域名发送请求时，浏览器会自动去自己存储的 cookie 列表中，查找到当前域名下的所有cookie，然后加到 http请求 中发送给服务器，服务器收到后查找最匹配的cookie并进行响应操作

  > Cookie 列表：包含所有cookie，包含 httponly 的，而且同一个域名下可能有多个cookie
  >
  > 如果是fetch请求的话，需要设置 withCredentials: true ，浏览器才会在http请求中添加cookie

一般情况下，cookie是由服务端生成的，服务端通过在发送给客户端的响应报文头中设置 set-cookie 字段来设置cookie，与此同时，客户端也可以自己通过设置 document.cookie 来 **添加** 一个新的cookie（不会覆盖已有的cookie，除非设置 cookie 的name、value、domain、path 都和一个已经存在的cookie重复）

### 3. 实现机制

1. 浏览器向某个URL发起HTTP请求（可以是任何请求，比如GET一个页面、POST一个登录表单等）

2. 对应的服务器收到该HTTP请求，并计算应当返回给浏览器的HTTP响应。

   > HTTP响应包括请求头和请求体两部分，可以参见：[读 HTTP 协议](https://harttle.land/2014/10/01/http.html)。

3. 在响应头加入`Set-Cookie`字段，它的值是要设置的Cookie。

   在[RFC2109 6.3 Implementation Limits](https://www.ietf.org/rfc/rfc2109.txt)中提到： UserAgent（浏览器就是一种用户代理）至少应支持300项Cookie， 每项至少应支持到4096字节，每个域名至少支持20项Cookie。

4. 浏览器收到来自服务器的HTTP响应。

5. 浏览器在响应头中发现`Set-Cookie`字段，就会将该字段的值保存在内存或者硬盘中。

   `Set-Cookie`字段的值可以是很多项Cookie，每一项都可以指定过期时间`Expires`。 默认的过期时间是用户关闭浏览器时。

6. 浏览器下次给该服务器发送HTTP请求时， 会将服务器设置的Cookie附加在HTTP请求的头字段`Cookie`中。

   浏览器可以存储多个域名下的Cookie，但只发送当前请求的域名曾经指定的Cookie， 这个域名也可以在`Set-Cookie`字段中指定）。

7. 服务器收到这个HTTP请求，发现请求头中有`Cookie`字段， 便知道之前就和这个用户打过交道了。

8. 过期的Cookie会被浏览器删除。

cookie有一个很不安全的地方，**Cookie是明文传输的**，所以恶意的中间代理可以很容易地截取到http请求中的cookie，如果我们把用户名和密码直接放在cookie里的话，中间代理可以很容易地获取到我们的这些关键信息，也可以很容易地模拟一个真实用户进行欺骗性的操作。

就算我们用一些哈希操作对cookie进行加密，只要恶意代理可以截取一次用户的正常登陆操作，就可以拿到正常的登录状态的cookie值（重放攻击，其实session也无法避免）

## Session

### session的实现机制

1. 用户提交包含用户名和密码的表单，发送HTTP请求。

2. 服务器验证用户发来的用户名密码。

3. 如果正确则把当前用户名（通常是用户对象）存储到redis中，并生成它在redis中的ID。

   这个ID称为Session ID，通过Session ID可以从Redis中取出对应的用户对象， 敏感数据（比如`authed=true`）都存储在这个用户对象中。

4. 设置Cookie为`sessionId=xxxxxx|checksum`并发送HTTP响应， 仍然为每一项Cookie都设置签名。

5. 用户收到HTTP响应后，便看不到任何敏感数据了。在此后的请求中发送该Cookie给服务器。

6. 服务器收到此后的HTTP请求后，发现Cookie中有SessionID，进行放篡改验证。

7. 如果通过了验证，根据该ID从Redis中取出对应的用户对象， 查看该对象的状态并继续执行业务逻辑。

## cookie 和 session 的区别

1. session 是在服务器端保存的数据结构，用来跟踪用户状态。cookie是在客户端保存用户信息的一种机制，用来记录 用户的一些信息（是session的一种实现方式）
2. session更安全
3. 访问增多时，session会较大地占用服务器的性能。考虑减轻服务器性能的方面，应该适当使用 Cookie
4. session 依赖 session Id， session id 是存储在 cookie中存储的，如果禁用了cookie，session也会失效，这个时候可以使用 localStorage + header 和 url 中进行传输

cookie 和 session 都不是绝对安全的，只是session 可以更有效地防止恶意用户对重要信息进行篡改之后，再对服务器发送篡改信息的攻击（但是它们对于预防重放攻击的处理办法都是相同的），因为cookie是明文，而session是服务器生成的哈希串，而且还带了checksum校验

防止重放攻击：对cookie内容进行加密，加入时间戳格式（或服务器和客户端商定的随机数递增模式），对内容加密之后再传输（保证每次加密的密文都不一样）

## cookie 和 StorageAPI

storageAPI 包括 localStorage 和 sessionStorage，都是存储在浏览器端的一种数据结构

- cookie：用于存储登录信息等
- localStorage：HTML5 新特性，为一个给定的源（origin）维持一个独立的存储区域，该存储区是长时间可用的，即使用户关闭了浏览器之后重新打开，数据仍然存在
- sessionStorage：为一个给定的源（origin）维持一个独立的存储区域，把一部分数据在当前会话中保留下来，刷新页面数据依旧存在，但是页面关闭后，sessionStorage 中的数据就会清空

localStorage 和 sessionStorage 在 window 中返回的都是一个 Storage  对象，可以使用相同的方法来操作这些对象

### 区别

| 特性           | Cookie                                                       | localStorage                                                | sessionStorage                                               |
| -------------- | ------------------------------------------------------------ | ----------------------------------------------------------- | ------------------------------------------------------------ |
| 数据的生存期   | 一般由服务器生成，可设置失效时间。如果在浏览器端生成Cookie，默认是关闭浏览器后失效 | 对应origin的存储区，除非被清除，否则永久保存                | 对应origin的存储区，仅在当前会话下有效，关闭页面或浏览器后被清除 |
| 存放数据大小   | 4K左右                                                       | 5MB                                                         | 5MB                                                          |
| 与服务器端通信 | 每次都会携带在HTTP头中，如果使用cookie保存过多数据会带来性能问题 | 仅在客户端（即浏览器）中保存，不参与和服务器的通信          |                                                              |
| 易用性         | 需要程序员自己封装，源生的Cookie接口不友好                   | 源生接口可以接受，亦可再次封装来对Object和Array有更好的支持 |                                                              |

localStorage 和 sessionStorage 的异同：

相同点：

1. 存储大小均为5M左右
2. 都有同源策略的限制
3. 只在客户端中保存，不参与和服务器的通信

不同点：

1. 数据生存期不同，localStorage 永久有效，sessionStorage 只在当前会话下有效，关闭页面或浏览器之后就会被清除
2. 作用域不同：
   1. localStorage：**同一个浏览器内**，同源文档之间共享localStorage数据
   2. sessionStorage：只有**同一个tab页面**内才可以共享数据，如果是同源文档，但是处于同一浏览器的不同tab页，sessionStorage 是不能共享的；而同一个tab页中不同的 iframe 之间，是可以共享 sessionStorage 的

https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API

### 应用：

localStorage 可以用于在两个同源的浏览器tab页面之间传递消息，只要更新 localStorage，两个tab页面是共享同一个localStorage的

可以使用 `window.addEventListener('storage', function(e){})` 来监听 storage 改变的事件，并对其作出相应（具体 storage 内容 可以从 e.storageArea 中获取）

**安全性考虑**：

cookie、localStorage 和 sessionStorage 是可以通过控制台进行修改的，所以要注意代码没有 XSS 注入的风险

storage对象需要考虑其大小显示，不能无限制添加，特别是localStorage，由于是永久性存储，我们需要随手进行removeItem 或者 clear 对其进行清除，

缓解方案：使用压缩方式对数据进行压缩，同时取数据的同时也要进行解压