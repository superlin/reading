Ajax
===
##数据传输
###请求数据
有五种常用技术用于向服务器请求数据：

*	XMLHttpRequest (XHR)
*	动态脚本标签插入
*	iframes
*	Comet
*	多部分的XHR

多部分XHR（MXHR）允许你只用一个HTTP 请求就可以从服务器端获取多个资源。它通过将资源（可以是CSS 文件，HTML 片段，JavaScript 代码，或base64 编码的图片）打包成一个由特定分隔符界定的大字符串，从服务器端发送到客户端。JavaScript 代码处理此长字符串，根据它的媒体类型和其他“信息头”解析出每个资源。

```javascript
var req = new XMLHttpRequest();
req.open('GET', 'rollup_images.php', true);
req.onreadystatechange = function() {
	if (req.readyState == 4) {
		splitImages(req.responseText);
	}
};
req.send(null);
```
这是一个非常简单的请求。向rollup_images.php 要求数据，一旦你收到返回结果，就将它交给函数splitImages 处理。rollup_images.php读取三个图片，并将它们转换成base64 字符串。它们之间用一个简单的字符，UNICODE的1，连接起来，然后返回给客户端，此数据由splitImage 函数处理：

```javascript
function splitImages(imageString) {
	var imageData = imageString.split("\u0001");
	var imageElement;
	for (var i = 0, len = imageData.length; i < len; i++) {
		imageElement = document.createElement('img');
		imageElement.src = 'data:image/jpeg;base64,' + imageData[i];
		document.getElementById('container').appendChild(imageElement);
	}
}
```
如果MXHR 请求很多张图片，那么响应报文越来越大，有必要在每个资源收到时立刻处理，而不是等待整个响应报文接收完成。这可以通过监听readyState 3 实现：

```javascript
var req = new XMLHttpRequest();
var getLatestPacketInterval, lastLength = 0;
req.open('GET', 'rollup_images.php', true);
req.onreadystatechange = readyStateHandler;
req.send(null);

function readyStateHandler{
	if (req.readyState === 3 && getLatestPacketInterval === null) {
		// 开始轮询
		getLatestPacketInterval = window.setInterval(function() {
			getLatestPacket();
		}, 15);
	}
	if (req.readyState === 4) {
		// 停止轮询
		clearInterval(getLatestPacketInterval);
		// 获取最新的数据
		getLatestPacket();
	}
}

function getLatestPacket() {
	var length = req.responseText.length;
	var packet = req.responseText.substring(lastLength, length);
	processPacket(packet);
	lastLength = length;
}
```
当readyState 3 第一次发出时，启动了一个定时器。每隔15 毫秒检查一次响应报文中的新数据。数据片段被收集起来直到发现一个分隔符，然后一切都作为一个完整的资源处理。

使用此技术有一些缺点，其中最大的缺点是以此方法获得的资源不能被浏览器缓存。

另一个缺点是：老版本的IE不支持readyState 3 或data: URL。IE 8 两个都支持，但在IE 6 和7 中必须设法变通。

使用MXHR的情况：

*	网页包含许多其他地方不会用到的资源（所以不需要缓存），尤其是图片
*	网站为每个页面使用了独一无二的打包的JavaScript 或CSS 文件以减少HTTP 请求，因为它们对每个页面来说是独一的，所以不需要从缓存中读取，除非重新载入特定页面

###发送数据
当数据只需发送给服务器时，有两种广泛应用的技术：**XHR** 和**灯标**。

灯标与动态脚本标签插入非常类似。JavaScript 用于创建一个新的Image 对象，将src 设置为服务器上一个脚本文件的URL。此URL 包含我们打算通过GET 格式传回的键值对数据。注意并没有创建img 元素或者将它们插入到DOM 中。

```javascript
var url = '/status_tracker.php';
var params = [
	'step=2',
	'time=1248027314'
];
(new Image()).src = url + '?' + params.join('&');
```

灯标是向服务器回送数据最快和最有效的方法。服务器根本不需要发回任何响应正文，所以你不必担心客户端下载数据。唯一的缺点是接收到的响应类型是受限的。如果你需要向客户端返回大量数据，那么使用XHR。如果你只关心将数据发送到服务器端（可能需要极少的回复），那么使用图像灯标。

##数据格式

*	XML
*	XPath
*	JSON
*	JSONP
*	HTML
*	自定义格式（如：使用分号或逗号分隔字段）

##Ajax性能向导
有两种主要方法避免发出一个不必要的请求：

*	在服务器端，设置HTTP 头（如``Expires: Mon, 28 Jul 2014 23:30:00 GMT``），确保返回报文将被缓存在浏览器中。
*	在客户端，于本地缓存已获取的数据，不要多次请求同一个数据。

##总结

*	减少请求数量，可通过JavaScript 和CSS 文件打包，或者使用MXHR。
*	缩短页面的加载时间，在页面其它内容加载之后，使用Ajax 获取少量重要文件。
*	确保代码错误不要直接显示给用户，并在服务器端处理错误。
*	学会何时使用一个健壮的Ajax 库，何时编写自己的底层Ajax 代码。