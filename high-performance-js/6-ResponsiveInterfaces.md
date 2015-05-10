响应接口
===
大多数浏览器有一个单独的处理进程，它由两个任务所共享：JavaScript 任务和用户界面更新任务。每个时刻只有其中的一个操作得以执行，也就是说当JavaScript代码运行时用户界面不能对输入产生反应，反之亦然。

##浏览器UI线程
JavaScript 和UI 更新共享的进程通常被称作浏览器UI 线程（虽然对所有浏览器来说“线程”一词不一定准确）。此UI 线程围绕着一个简单的队列系统工作，任务被保存到队列中直至进程空闲。一旦空闲，队列中的下一个任务将被检索和运行。这些任务不是运行JavaScript 代码，就是执行UI 更新，包括重绘和回流（在第三章讨论过）。
```html
<html>
<head>
	<title>Browser UI Thread Example</title>
</head>
<body>
	<button onclick="handleClick()">Click Me</button>
	<script type="text/javascript">
	function handleClick(){
		var div = document.createElement("div");
		div.innerHTML = "Clicked!";
		document.body.appendChild(div);
	}
	</script>
</body>
</html>
```
当例子中的按钮被点击时，它触发UI 线程创建两个任务并添加到队列中。第一个任务是按钮的UI 更新，它需要改变外观以指示出它被按下了，第二个任务是JavaScript 运行任务，包含handleClick()的代码，被运行的唯一代码就是这个方法和所有被它调用的方法。假设UI 线程空闲，第一个任务被检查并运行以更新按钮外观，然后JavaScript 任务被检查和运行。在运行过程中，handleClick()创建了一个新的<div>元素，并追加在<body>元素上，其效果是引发另一次UI 改变。也就是说在JavaScript 运行过程中，一个新的UI更新任务被添加在队列中，当JavaScript 运行完之后，UI 还会再更新一次，如下图所示。

![UI thread tasks](http://superlin.github.io/reading/high-performance-js/9-UIThread.png)

###浏览器限制
浏览器在JavaScript 运行时间上采取了限制。这是一个有必要的限制，确保恶意代码编写者不能通过无尽的密集操作锁定用户浏览器或计算机。此类限制有两个：**调用栈尺寸限制**（第四章讨论过）和**长时间脚本限制**。

有两种方法测量脚本的运行时间。第一个方法是统计自脚本开始运行以来执行过多少语句。此方法意味着脚本在不同的机器上可能会运行不同的时间长度，可用内存和CPU 速度可以影响一条独立语句运行所花费的时间。第二种方法是统计脚本运行的总时间。在特定时间内可运行的脚本数量也因用户机器性能差异而不同，但脚本总是停在固定的时间上。

*	Internet Explorer 4中，设置默认限制为5 百万条语句
*	Firefox 默认限制为10 秒钟
*	Safari 默认限制为5 秒钟
*	Chrome 没有独立的长运行脚本限制，替代以依赖它的通用崩溃检测系统来处理此类实例
*	Opera 没有长运行脚本限制，将继续运行JavaScript 代码直至完成，由于Opera 的结构，当运行结束时它并不会导致系统不稳定

###多久算是太久
>JavaScript运行了整整几秒钟很可能是做错了什么……  -Brendan Eich

**一个单一的JavaScript 操作应当使用的总时间（最大）是100 毫秒**。

Nielsen 指出如果该接口在100 毫秒内响应用户输入，用户认为自己是“直接操作用户界面中的对象。”超过100 毫秒意味着用户认为自己与接口断开了。由于UI 在JavaScript 运行时无法更新，如果运行时间长于100 毫秒，用户就不能感受到对接口的控制。

##用定时器让出时间片
尽管你尽了最大努力，还是有一些JavaScript 任务因为复杂性原因不能在100 毫秒或更少时间内完成。这种情况下，理想方法是让出对UI 线程的控制，使UI 更新可以进行。让出控制意味着停止JavaScript 运行，给UI 线程机会进行更新，然后再继续运行JavaScript。这就是定时器的用途。

###定时器基础
``setTimeout()``和``setInterval()``第二个参数指出什么时候应当将任务添加到UI 队列之中，并不是说那时代码将被执行。

###定时器精度
在Windows 系统上定时器分辨率为15 毫秒，也就是说一个值为15 的定时器延时将根据最后一次系统时间刷新而转换为0 或者15。设置定时器延时小于15 将在Internet Explorer 中导致浏览器锁定，所以最小值建议为25 毫秒（实际时间是15 或30）以确保至少15 毫秒延迟。

###在数组处理中使用定时器
数组循环可能需要过长的时间，一般可以使用定时器来优化循环。是否可用定时器取代循环有两个决定性因素：**过程并不是必须同步处理**、**数据不是必须有序处理**。满足这两个条件的数据操作可以用定时器来改写，封装如下：

```javascript
function processArray(items, process, callback){
	// 复制原始数组
	var todo = items.concat();

	setTimeout(function(){
		process(todo.shift());
		if (todo.length > 0){
			setTimeout(arguments.callee, 25);
		} else {
			callback(items);
		}
	}, 25);
}
```

如果一个函数运行时间太长，那么查看它是否可以分解成一系列能够短时间完成的较小的函数。可将一行代码简单地看作一个原子任务，多行代码组合在一起构成一个独立任务。某些函数可基于函数调用进行拆分。将拆分后的每个函数都放入数组中，然后调用定时器进行处理，封装如下：

```javascript
function multistep(steps, args, callback){
	// 复制原始数组	
	var tasks = steps.concat();

	setTimeout(function(){
		// 执行下一个任务
		var task = tasks.shift();
		task.apply(null, args || []);

		// 判断是否还有任务
		if (tasks.length > 0){
			setTimeout(arguments.callee, 25);
		} else {
			callback();
		}
	}, 25);
}
```

有时每次只执行一个任务效率不高。考虑这样一种情况：处理一个拥有1'000 个项的数组，每处理一个项需要1 毫秒。如果每个定时器中处理一个项，在两次处理之间间隔25 毫秒，那么处理此数组的总时间是(25 + 1) × 1'000 = 26'000 毫秒，也就是26 秒。如果每批处理50 个，每批之间间隔25 毫秒会怎么样呢？整个处理过程变成(1'000 / 50) × 25 + 1'000 = 1'500 毫秒，也就是1.5 秒，而且用户也不会察觉界面阻塞，因为最长的脚本运行只持续了50 毫秒。通常批量处理比每次处理一个更快。

JavaScript 可连续运行的最大时间是100 毫秒，建议是将这个数字削减一半，不要让任何JavaScript 代码持续运行超过50 毫秒，只是为了确保代码永远不会影响用户体验。

在第一种方法基础上进行改进，封装如下：

```javascript
function timedProcessArray(items, process, callback){
	//复制原始数组
	var todo = items.concat();
	
	setTimeout(function(){
		var start = +new Date();
		do {
			process(todo.shift());
		} while (todo.length > 0 && (+new Date() - start < 50));
		
		if (todo.length > 0){
			setTimeout(arguments.callee, 25);
		} else {
			callback(items);
		}
	}, 25);
}
```

##Web Worker
Web Worker是HTML5的一部分，而且已经被Firefox 3.5，Chrome 3，和Safari 4 原生实现。Web Worker的代码运行不仅不会影响浏览器UI，而且也不会影响其它工人线程中运行的代码。

###Web Worker运行环境
Web Worker不绑定UI 线程，这也意味着它们将不能访问许多浏览器资源。Web Worker修改DOM 将导致用户界面出错，但每个网页工人线程都有自己的全局运行环境，只有JavaScript 特性
的一个子集可用。工人线程的运行环境由下列部分组成：

*	浏览器对象，只包含四个属性：appName, appVersion, userAgent, 和platform
*	location 对象（和window 里的一样，只是所有属性都是只读的）
*	self 对象，指向全局Web Worker对象
*	importScripts()方法，使Web Worker可以加载外部JavaScript 文件（异步加载）
*	所有ECMAScript 对象，诸如Object，Array，Data，等等
*	XMLHttpRequest
*	setTimeout()和setInterval()方法
*	close()方法可立即停止Web Worker

###Web Worker交互
Web Worker和网页代码通过事件接口进行交互。网页代码可通过postMessage()方法向Web Worker传递数据，它接收单个参数，即传递给Web Worker的数据。

```javascript
var worker = new Worker("code.js");
worker.onmessage = function(event){
	alert(event.data);
};
worker.postMessage("Nicholas");
```
只有某些类型的数据可以使用postMessage()传递。你可以传递原始值（string，number，boolean，null和undefined），也可以传递Object 和Array 的实例，其它类型就不允许了。有效数据被序列化，传入或传出Web Worker，然后反序列化。

###加载外部文件
当Web Worker通过importScripts()方法加载外部JavaScript 文件，它接收一个或多个URL 参数，指出要加载的JavaScript 文件网址。Web Worker以阻塞方式调用importScripts()，直到所有文件加载完成并执行之后，脚本才继续运行。由于Web Worker在UI 线程之外运行，这种阻塞不会影响UI 响应。

```javascript
importScripts("file1.js", "file2.js");
self.onmessage = function(event){
	self.postMessage("Hello, " + event.data + "!");
};
```

###实际用途
例如，解析一个很大的JSON 字符串，假设数据足够大，至少需要500 毫秒才能完成解析任务。很显然时间太长了以至于不能允许JavaScript 在客户端上运行它，因为它会干扰用户体验。此任务难以分解成用于定时器的小段任务，所以工人线程成为理想的解决方案。

```javascript
var worker = new Worker("jsonparser.js");
// 当数据可用，事件处理函数被调用
worker.onmessage = function(event){
	// 获取返回的JSON数据
	var jsonData = event.data;
	// 使用JSON数据
	evaluateData(jsonData);
};
// 出入需要解析的json文本
worker.postMessage(jsonText);
```
Web Worker负责JSON解析：
```javascript
// jsonparser.js

// 当JSON数据可用，事件处理函数被调用
self.onmessage = function(event){
	// JSON数据通过event.data传入
	var jsonText = event.data;
	// 解析
	var jsonData = JSON.parse(jsonText);
	// 返回结果
	self.postMessage(jsonData);
};
```
即使JSON.parse()可能需要500 毫秒或更多时间，也没有必要添加更多代码来分解处理过程。此处理过程发生在一个独立的线程中，所以你可以让它一直运行完解析过程而不会干扰用户体验。

解析一个大字符串只是许多受益于网页工人线程的任务之一。其它可能受益的任务如下：

*	编/解码一个大字符串
*	复杂数学运算（包括图像或视频处理）
*	给一个大数组排序

任何超过100 毫秒的处理，都应当考虑工人线程方案是不是比基于定时器的方案更合适。当然，还要基于浏览器是否支持工人线程。

##总结

*	JavaScript 运行时间不应该超过100 毫秒。过长的运行时间导致UI 更新出现可察觉的延迟，从而对整体用户体验产生负面影响。
*	JavaScript 运行期间，浏览器响应用户交互的行为存在差异。无论如何，JavaScript 长时间运行将导致用户体验混乱和脱节。
*	定时器可用于安排代码推迟执行，它使得你可以将长运行脚本分解成一系列较小的任务。
*	Web Worker是新式浏览器才支持的特性，它允许你在UI 线程之外运行JavaScript 代码而避免锁定UI。