#### 什么是CORS

CORS（Cross-Origin Resource Sharing，跨域资源共享）由一系列**HTTP 响应头**组成，**这些HTTP响应头决定浏览器是否阻止前端JS代码跨域获取资源**。

浏览器的**同源安全策略**默认会阻止网页“跨域”获取资源。但如果接口服务器**配置了CORS相关的HTTP响应头**，就可以**解除浏览器端的跨域访问限制**。

![image-20221213132524878](C:\Users\dell\AppData\Roaming\Typora\typora-user-images\image-20221213132524878.png)

#### CORS的注意事项

1 CORS主要在服务器端进行配置。客户端浏览器无需做任务额外的配置，即可请求开启了CORS的接口。

2 CORS在浏览器中有兼容性。只有支持XMLHttpRequest Level2的浏览器，才能正常访问开启了CORS的服务端接口（例如：IE10+、Chrome4+、FireFox3.5+）。

#### CORS响应头部 - Access-Control-Allow-Origin

如果指定了Access-Control-Allow-Origin字段的值为通配符*，表示允许来自任何域的请求，示例代码如下：

```js
res.setHeader('Access-Control-Allow-Origin', *)
```

#### CORS响应头部 - Access-Control-Allow-Headers

默认情况下，CORS**仅**支持客户端向服务器发送如下的9个**请求头**：

Accept、Accept-Language、Content-Language、DPR、Downlink、Save-Data、Viewport-Width、Width、Content-Type（值仅限于text/plain、multipart/form-data、application/x-www-form-urlencoded三者之一）

如果客户端向服务器**发送了额外的请求头信息**，则需要在**服务器端**，通过Access-Control-Allow-Headers对**额外的请求头进行声明**，否则这次请求会失败！

```js
// 允许客户端额外向服务器发送Content-Type请求头和X-Custom-Header请求头
// 注意：多个请求头之间使用英文的逗号进行分割
res.setHeader('Access-Control-Allow-Headers', 'Content-Type', 'X-Custom-Header')
```

#### CORS响应头部-Access-Control-Allow-Methods

默认情况下，CORS仅支持客户端发起GET、POST、HEAD请求。

如果客户端希望通过PUT、DELETE等方式请求服务器的资源，则需要在服务器端，通过Access-Control-Allow-Methods来指明实际请求所允许使用的HTTP方法。

示例代码下：

```js
// 只允许POST、GET、DELETE、HEAD请求方法
res.setHeader('Access-Control-Allow-Methods', 'GET、POST、DELETE、HEAD')
// 允许所有的HTTP请求方法
res.setHeader('Access-Control-Allow-Methods', '*')
```

#### CORS请求的分类

客户端在请求CORS接口时，根据请求方式和请求头的不同，可以将CORS的请求分为两大类，分别是：

**1 简单请求**

​	同时满足以下两大条件的请求，就属于简单请求。

- 请求方式：GET、POST、HEAD三者之一
- HTTP头部信息不超过一下几种字段：无自定义头部字段、Accept、Accept-Language、Content-Language、DPR、Downlink、Save-Data、Viewport-Width、Width、Content-Type（只有三个值text/plain、multipart/form-data、application/x-www-form-urlencoded）

**2 预检请求**

​	只要符合以下任何一个条件的请求，都需要进行预检请求：

- 请求方式为GET、POST、HEAD之外的请求Method类型
- 请求头中包含自定义头部字段
- 向服务器发送了application/json格式的数据

​	在浏览器与服务器正式通信之前，浏览器会先发送OPTION请求进行预检，以获知服务器是否允许该实际请求，所以这一次的OPTION请求称为“预检请求”。服务器成功响应预检请求后，才会发送真正的请求，并且携带真实数据。

#### 简单请求和预检请求的区别

简单请求的特点：客户端与服务器之前只会发生一次请求。

预检请求的特点：客户端与服务器之间会发生两次请求，OPTION预检请求成功之后，才会发起真正的请求。