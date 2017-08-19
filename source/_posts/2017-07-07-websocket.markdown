---
layout: post
title: "WebSocket介绍与实例分析"
date: 2017-07-07 00:32:47 +0800
comments: true
categories: Network
---
{% img /images/2017-07-07/1.png %}

WEB访问一般都是基于Request/Response模式的HTTP协议，请求由浏览器发起，服务器处理请求后发送回响应。HTTP协议是短连接单向通信模型，服务器不能主动发送内容给客户端。为了解决这个问题，浏览器可以周期性地主动发送请求给服务器，服务器若没有相应数据则返回空响应，否则返回相应数据并关闭连接。这种方案被称为轮询(POLLING)。轮询需要不停地建立连接、并闭连接，不断浪费网络资源。为了减少不必要的网络消耗，LONG POLLING方案被提出。浏览器发送请求给服务器后，若服务器没有相应数据时并不返回空响应及断开连接，而是将请求保持住，当有了相应数据后或者超时发生再将数据返回给客户端并断开连接。当连接断开后，客户端会再次发起请求。这种方案依然会消耗多个TCP连接，HTTP Streaming技术进一步优化了TCP连接的使用效率。当服务器返回响应后并不关闭TCP连接，后续的请求和响应继续复用该连接。

本质上这些方案都是半双工单向通信，于是业内为基于浏览器的应用程序开发提出了全双工的实时双向通信方式: WebSocket。WebSocket是基于TCP的一种数据传输协议，它被设计成可以支持原来直接运行于TCP之上的任何协议。其他协议可以将WebSocket视为传输层，比如XMPP，STOMP，AMQP等。这给基于浏览器的应用程序开发带了具大灵活性，可以不再单纯依赖原有的HTTP协议。

<!--more-->

WebSocket涉及两大块内容:

* WebSocket协议: 在TCP协议之上构建的面向消息的双向通讯协议，由[RFC6455](https://tools.ietf.org/html/rfc6455)描述。
* WebSocket API: 为HTML5规范的一部分，由W3C标准化，由各浏览器实现, 可以参考[文档](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)

WebSocket协议分为两部分：握手和数据传输。握手部分基于HTTP Upgrade请求，当服务器端与客户端完成握手后，升级连接为以WebSocket数据传输协议进行双向数据通信。

WebSocket协议URL也与HTTP类似，明文协议`scheme`使用`ws:`, 而基于SSL/TLS的WebSocket协议的`scheme`为`wss:`。握手请求中的`Sec-WebSocket-Key` Header会携带一个`BASE64`编码的字符串，服务器端收到该字符串，在后边拼接固定字符串`258EAFA5-E914-47DA-95CA-C5AB0DC85B11`拼接，然后计算拼接后字符串的SHA1值，再将SHA1值以BASE64编码后在响应的header `Sec-WebSocket-Accept`中返回。

握手算法以PHP代码展示如下:
```php
<?php
$accept_key = base64_encode(sha1($key . "258EAFA5-E914-47DA-95CA-C5AB0DC85B11", true));
?>
```

握手完成的状态码为`101`。

比如，握手请求示例如下:
```plain
GET /ws HTTP/1.1
Host: 192.168.33.12
Connection: Upgrade
Pragma: no-cache
Cache-Control: no-cache
Upgrade: websocket
Origin: http://192.168.33.12
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: z4XuiRiQBy5pdZhjrpnm6w==
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
```
服务端接收到请求后，将根据握手算法计算出的值随响应包发送回客户端:
```plain
HTTP/1.1 101 Switching Protocols
Server: openresty/1.11.2.3
Date: Wed, 05 Jul 2017 16:04:30 GMT
Connection: upgrade
Upgrade: websocket
Sec-WebSocket-Accept: avPZsvkfFcAwyIakc0rghccqDoI=
```

握手完成后, 开始以数据帧形式通信，一条消息可以拆分为多个数据帧进行发送。本文不再详述WebSocket数据帧格式，具体参考[RFC 6455](https://tools.ietf.org/html/rfc6455)。

下面我们来看WebSocket API。

客户端使用WebSocket之前需要首先创建WebSocket对象来初始化WebSocket连接:
```javascript
var socket = new WebSocket(“ws://192.168.33.12/ws”);
```
WebSocket API设计为事件驱动形式，WebSocket对象有4种事件:

* open: 握手完成后触发，默认回调函数为onopen
* message: 收到服务端发送的消息后触发, 默认回调函数为onmessage
* close: 连被关闭后触发, 默认回调函数为onclose
* error: 通信过程中发生错误时触发, 默认回调函数为onerror

若事件发生时需要调用多个回调，可以使用addEventListener添加多个回调函数，如:
```javascript
socket.addEventListener(“open”, function(event) {
    // handle open event
});
```
WebSocket连接建立后，需要调用`send()`方法发送数据。WebSocket支持发送文本和二进制数据。二进制数据可以为`ArrayBuffer`和`Blob`两种，具体使用哪个，需要通过`binaryType`属性来指定:
```javascript
socket.binaryType = ”arraybuffer”; 
```
或:
```javascript
socket.binaryType = ”blob”;
```

接收JSON文本的示例如下:
```javascript
socket.onmessage = function(event) {
    if (typeOf event.data === String) {
        // create a JSON object
        var jsonObject = JSON.parse(event.data);
        var username = jsonObject.name;
        var message = jsonObject.message;
        console.log(“Received data string”);
    }
}
```
而接收ArrayBuffer的示例如下:
```javascript
socket.binaryType = ”arraybuffer”;
socket.onmessage = function(event) {
    if (event.data instanceof ArrayBuffer) { 
        var buffer = event.data;
        console.log(“Received arraybuffer”);
    }
}
```
当数据通信完成后，需要调用`close()`关闭WebSocket连接，连接完成后`close`事件被触发。`close`事件回调示例如下:
```javascript
socket.onclose = function(event) {
    var code = event.code;
    var reason = event.reason;
    var wasClean = event.wasClean;
    // handle close event 
};
```

在处理应用程序逻辑时可能需要根据WebSocket连接状态进行不同操作。连接状态可以从WebSocket对象的readyState属性获取。在WebSocket连接生命周期内，共有4种状态:

* connecting: 正在连接中
* open: 连接完成，只有该状态下才可以发送和接收数据。
* closeing: 正在关闭
* closed: 已关闭

处理示例如下:
```javascript
switch (socket.readyState) {
    case WebSocket.CONNECTING:
        // do something
        break;
    case WebSocket.OPEN:
        // do something
        break;
    case WebSocket.CLOSING:
        // do something
        break;
    case WebSocket.CLOSED:
        // do something
        break;
    default:
        // this never happens
        break;
}
```

下面我们基于OpenResty的WebSocket实现和Redis来构造一个推送示例。

在nginx.conf中，添加WebSocket的配置块, 指定由`conf`目录下的`websocket.lua`来处理请求:
```plain
location = /ws {
    lua_socket_log_errors off;
    lua_check_client_abort on;
    content_by_lua_file conf/websocket.lua;
}
```

在NGINX的`root`目录下创建ws.html, 内容如下:
```html
<html>
<head>
<script>
var ws = null;
function connect() {
    if (ws !== null) return log('already connected');
    ws = new WebSocket('ws://192.168.33.12/ws?channel=foo')
    ws.onopen = function() {
        log('connected');
    };

    ws.onerror = function(error) {
        log(error.message);
    };

    ws.onmessage = function(e) {
        log('recv: ' + e.data);
    };

    return false;
}

function log(text) {
    var li = document.createElement('li');
    li.appendChild(document.createTextNode(text));
    document.getElementById('log').appendChild(li);
    return false;
}
</script>
</head>

<body>
    <button type = "button" onclick = "return connect();">
        Connect
    </button>
    <ol id="log"></ol>
</body>
</html>
```
在这个页面里我们添加了一个按钮，当点击按钮时创建连接WebSocket, 之后收到消息便显示到页面上。


在NGINX的`conf`目录下添加`websocket.lua`文件，内容如下:
```lua
local server = require "resty.websocket.server" 
local cjson = require "cjson" 

if not ngx.var.arg_channel then
    ngx.log(ngx.ERR, "no channel supplied");
    return ngx.exit(444)
end

local wb, err = server:new{
    timeout = 5000,
    max_payload_len = 65535
}

if not wb then
    ngx.log(ngx.ERR, "failed to new websocket: ", err)
    return ngx.exit(444)
end

local redis = require "resty.redis" 
local red = redis:new()

red:set_timeout(1000)

local ok, err = red:connect("127.0.0.1", 6379)
if not ok then
    return ngx.exit(444)
end

local res, err = red:subscribe(ngx.var.arg_channel)
if not res then
    return ngx.exit(444)
end

red:set_timeout(10000)
while true do
    local res, err = red:read_reply()
    if not res then
        if err ~= "timeout" then
            return ngx.exit(444)
        end
    else
        local bytes, err = wb:send_text(cjson.encode(res))
        ngx.log(ngx.ERR, "send text: ", cjson.encode(res))
        if not bytes then
            ngx.log(ngx.ERR, "failed to send text: ", err)
            return ngx.exit(444)
        end
    end
end

wb:send_close()
```
首先从WebSocket的握手请求中获取channel参数，接着定阅本地Redis中以该参数命名的channel。当收到channel的消息后，实时发送给客户端。


我们从浏览器访问`ws.html`页面, 单击页面上的`connect`按钮, 结果如图:

{% img /images/2017-07-07/2.png %}

页面结果显示WebSocket连接已经建立。接下来，我们在服务器上通过`redis-cli`推送消息:
```plain
[root@localhost vagrant]# redis-cli
127.0.0.1:6379> publish foo hello
(integer) 2
127.0.0.1:6379> publish foo world
(integer) 2
127.0.0.1:6379>
```
页面结果如下图，服务器推送的消息已经展示在页面上。

{% img /images/2017-07-07/3.png %}

上述示例代码只是简单演示效果，并没有考虑逻辑是否合理。整体来看，WebSocket还是比较底层的协议，只提供了消息通道，很多通用逻辑需要应用程序自己来实现，如身份验证，客户端管理等等。
