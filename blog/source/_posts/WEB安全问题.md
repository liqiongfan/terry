---
title: WEB安全问题
date: 2020-08-02 14:05:16
tags:
- WEB
- SQL注入
- XSS
- CSRF
- XEE
- LFI
categories:
- tech
description: WEB安全问题
---

### 什么是WEB安全？
WEB安全是指网络编码安全问题，防止各种注入、代码漏洞问题，维护WEB网络安全

### PHP WEB安全的种类
- SQL注入

SQL注入应该算是非常常见的一种安全编码问题，因为现在一般下而言，很多开发都是直接拼接SQL语句，如下就是一个常见的例子：

网站通过PHP获取用户信息：
```php php
<?php

$username = $_GET['username'];
$query = "SELECT * FROM users WHERE username = '$username'";
```

攻击者控制通过 GET 和 POST 发送的查询（或者例如 UA 的一些其他查询）。一般情况下，你希望查询户名为「 terry 」的用户产生的 SQL 语句如下：

```sql mysql
SELECT * FROM users WHERE username = 'peter'
```

如果客户端上送特殊的参数，如： 击者发送了特定的用户名参数，例如：`' OR '1'='1` ，那么拼接的SQL如下：

```sql mysql
SELECT * FROM users WHERE username = 'terry' OR '1' = '1'
```

所以这里建议使用mysql的预处理功能，能够防范SQL注入风险

- XSS 跨站脚本攻击

```html
<body>
<?php
$searchQuery = $_GET['q'];
/* some search magic here */
?>
<h1>You searched for: <?php echo $searchQuery; ?></h1>
<p>We found: Absolutely nothing because this is a demo</p>
</body>
```

因为我们把用户的内容直接打印出来，不经过任何过滤，非法用户可以拼接 URL：

```html
search.php?q=%3Cscript%3Ealert(1)%3B%3C%2Fscript%3E
```

PHP 渲染出来的内容如下，可以看到 Javascript 代码会被直接执行：
```html
<body>
<h1>You searched for: <script>alert(1);</script></h1>
<p>We found: Absolutely nothing because this is a demo</p>
</body>
```

JavaScript 可以完成如下工作：
+ 偷走你用户浏览器里的 Cookie；
+ 通过浏览器的记住密码功能获取到你的站点登录账号和密码；
+ 盗取用户的机密信息；
+ 你的用户在站点上能做到的事情，有了 JS 权限执行权限就都能做，也就是说 A 用户可以模拟成为任何用户；
+ 在你的网页中嵌入恶意代码

如果防止此类问题，PHP可以使用函数 `htmlentities` 处理：
```php php
<?php
$searchQuery = htmlentities($searchQuery, ENT_QUOTES);
```

另外必须使用 Cookie 时，如果无需 JS 读取的话，请必须设置为 "HTTP ONLY"。这个设置可以令 JavaScript 无法读取 PHP 端种的 Cookie。

- XSRF/CSRF

CSRF 是跨站请求伪造的缩写，它是攻击者通过一些技术手段欺骗用户去访问曾经认证过的网站并运行一些操作。

```php php
<?php
//delete-account.php

$confirm = $_GET['confirm'];

if($confirm === 'yes') {
  //goodbye
}
```

攻击者可以在他的站点上构建一个触发这个 URL 的表单（同样适用于 POST 的表单），或者将 URL 加载为图片诱惑用户点击：
```html html
<img src="https://example.com/delete-account.php?confirm=yes" />
```

- LFI 本地文件包含攻击

LFI （本地文件包含） 是一个用户未经验证从磁盘读取文件的漏洞。
我经常遇到编程不规范的路由代码示例，它们不验证过滤用户的输入。我们用以下文件为例，将它要渲染的模板文件用 GET 请求加载。
```php php
<body>
<?php
  $page = $_GET['page'];
  if(!$page) {
    $page = 'main.php';
  }
  include($page);
?>
</body>
```

由于 Include 可以加载任何文件，不仅仅是PHP，攻击者可以将系统上的任何文件作为包含目标传递。
```html html
index.php?page=../../etc/passwd
```
要防御此类攻击，你必须仔细考虑允许用户输入的类型，并删除可能有害的字符，如输入字符中的“.” “/” “\”。
如果你真的想使用像这样的路由系统（我不建议以任何方式），你可以自动附加 PHP 扩展，删除任何非 [a-zA-Z0-9-_] 的字符，并指定从专用的模板文件夹中加载，以免被包含任何非模板文件。
我在不同的开发文档中，多次看到造成此类漏洞的 PHP 代码。从一开始就要有清晰的设计思路，允许所需要包含的文件类型，并删除掉多余的内容。你还可以构造要读取文件的绝对路径，并验证文件是否存在来作为保护，而不是任何位置都给予读取。

- 不充分的密码哈希
现在单纯使用md5、sha1算法是不安全的，如果需要加密客户密码，需要使用新版本的函数 `password_hash` 和 `password_verify`

新版的 PHP 中也自带了安全的密码哈希函数 password_hash ，此函数已经包含了加盐处理。对应的密码验证函数为 password_verify 用来检测密码是否正确。password_verify 还可有效防止 时序攻击.

使用例子：
```php php
<?php

//user signup
$password = $_POST['password'];
$hashedPassword = password_hash($password, PASSWORD_DEFAULT);

//login
$password = $_POST['password'];
$hash = '1234'; //load this value from your db

if(password_verify($password, $hash)) {
  echo 'Password is valid!';
} else {
  echo 'Invalid password.';
}
```

- 中间人攻击

MITM （中间人） 攻击不是针对服务器直接攻击，而是针对用户进行，攻击者作为中间人欺骗服务器他是用户
欺骗用户他是服务器，从而来拦截用户与网站的流量，并从中注入恶意内容或者读取私密信息，通常发生在公共 WiFi 网络中
也有可能发生在其他流量通过的地方，例如ISP运营商。对此的唯一防御是使用 HTTPS，使用 HTTPS 可以将你的连接加密
并且无法读取或者篡改流量。你可以从 Let's Encrypt 获取免费的 SSL 证书，或从其他供应商处购买
这里不详细介绍如何正确配置 WEB 服务器，因为这与应用程序安全性无关，且在很大程度上取决于你的设置。
你还可以采取一些措施使 HTTPS 更安全，在 WEB 服务器配置加上 Strict-Transport-Security 标示头
此头部信息告诉浏览器，你的网站始终通过 HTTPS 访问，如果未通过 HTTPS 将返回错误报告提示浏览器不应显示该页面。
然而，这里有个明显的问题，如果浏览器之前从未访问过你的网站，则无法知道你使用此标示头，这时候就需要用到 Hstspreload。

- 命令注入

这可能是服务器遇到的最严重的攻击，命令注入的目标是欺骗服务器执行任意 Shell 命令

你如果使用 shell_exec 或是 exec 函数。让我们做一个小例子，允许用户简单的从服务器 Ping 不同的主机。

```php php
<?php

$targetIp = $_GET['ip'];
$output = shell_exec("ping -c 5 $targetIp");
```

输出将包括对目标主机 Ping 5次。除非采用 sh 命令执行 Shell 脚本，否则攻击者可以执行想要的任何操作。
```php php
ping.php?ip=8.8.8.8;ls -l /etc
```

Shell 将执行 Ping 和由攻击者拼接的第二个命令，这显然是非常危险的。
`escapeshellarg` 转义用户的输入并将其封装成单引号。

```php php
<?php

$targetIp = escapeshellarg($_GET['ip']);
$output = shell_exec("ping -c 5 $targetIp");
```

- XXE

XXE （XML 外部实体） 是一种应用程序使用配置不正确的 XML 解析器解析外部 XML 时，导致的本地文件包含攻击，甚至可以远程代码执行。

XML 有一个鲜为人知的特性，它允许文档作者将远程和本地文件作为实体包含在其 XML 文件中。

```xml xml
<?xml version="1.0" encoding="ISO-8859-1"?>
 <!DOCTYPE foo [
   <!ELEMENT foo ANY >
   <!ENTITY passwd SYSTEM "file:///etc/passwd" >]>
   <foo>&passwd;</foo>
```

如果你使用 libxml 可以调用 libxml_disable_entity_loader 来保护自己免受此类攻击。使用前请仔细检查 XML 库的默认配置，以确保配置成功。

