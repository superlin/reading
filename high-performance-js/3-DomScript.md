DOM操作
=====
DOM操作是昂贵的，一般是富web应用的性能瓶颈。这一章主要讨论三部分内容：

*	访问修改DOM元素
*	修改DOM样式，重绘和回流
*	DOM事件代理

##浏览器中的DOM
DOM和Javascript的实现一般是独立的。

*	IE：Javascript-JScript（jscript.dll），DOM-Trident（mshtml.dll）
*	Safari：Javascript-JavascriptCore，DOM-WebKit
*	Chrome：Javascript-V8，DOM-WebKit
*	FireFox：Javascript-SpiderMonky（最新的是TraceMonkey），DOM-Gecko

由于两者是独立的，互相访问一般是需要代价的。用一个形象的比喻来形容：**DOM是一座小岛，JS是另一座小岛，两个小岛之前有一座连接的大桥，但是桥上有一收费站，每次从JS小岛到DOM小岛都要经过这个收费站，因此通过小岛的次数越少代价越小，最好的方式是尽量留在JS小岛上**。

##DOM操作与修改
简单访问DOM是有代价的-“过路费”，修改元素的代价更大，因为修改常常会引起浏览器重新计算页面的结构。

最坏的情况就是在循环中操作DOM，如下所示：
```javascript
function innerHTMLLoop() {
	for (var count = 0; count < 15000; count++) {
		document.getElementById('here').innerHTML += 'a';
	}
}
```javascript
每次循环迭代都要操作两次DOM：获取``innerHTML``的值、修改``innerHTML``的值。更有效的方式是使用本地变量保存更新内容，在循环结束后一次性设置：
```
function innerHTMLLoop2() {
	var content = '';
	for (var count = 0; count < 15000; count++) {
		content += 'a';
	}
	document.getElementById('here').innerHTML += content;
}
```
这种修改在旧的浏览器中性能可以提升100-200多倍，在新的浏览器中也能提升20-50倍。

总之：**少和DOM接触，尽量和Javascript在一起吧**。

###innerHTML vs DOM方法
一般情况下使用``innerHTML``比使用DOM方法（如``createElement``等）好，在新的浏览器中有时不是很明显，在基于WebKit的浏览器中正好相反。

###克隆节点
更新页面内容时可以通过两种方式：克隆已存在的DOM元素或者创建新的元素。换句话说：使用``cloneNode``或``createElement``方法。

在大多数浏览器中克隆的性能要好一些，但是优势也并不是很明显，性能提升在2%-10%之间。

###HTML集合
返回值是HTML集合的方法：

*	document.getElementByName()
*	document.getElementByClassName()
*	document.getElementByTagName()

返回值是HTML集合的属性：

*	document.images
*	document.links
*	document.forms
*	document.forms[0].elements

HTML集合是动态的，即文档变化时集合中的元素也会随之变化。

HTML实际是一系列的查询语句，每次更新数据时都会更新这些查询语句，所以效率非常低。

如下代码会出现无限循环，因为每次插入后集合也会被更新，``length``属性的值也会随之增长：

```javascript
// 无限循环
var alldivs = document.getElementsByTagName('div');
for (var i = 0; i < alldivs.length; i++) {
	document.body.appendChild(document.createElement('div'))
}
```

访问集合的``length``属性比访问常规的数组的``length``属性要慢，因为每次都要重新执行查询语句。

###遍历DOM
遍历DOM可以使用``childNodes``或``firstChild``+``nextSibling``，这两种方式在多种浏览器中执行时间差不多。除了IE中，IE6中``nextSibling``大约快6倍，IE7中快105倍，所以推荐使用``nextSibling``。

``childNodes``、``firstChild``、``nextSibling``不区分注释、文本节点和元素节点，
只获得元素节点，可以使用以下属性：

*	children
*	firstElementChild
*	lastElementChild
*	nextElementSibling
*	previousElementSibling

``querySelectorAll``返回NodeList-不是动态的。

##重绘与回流
隐藏的DOM在渲染树上没有对应的节点。

改变盒模型、增减内容-元素的几何特性改变，会引起**回流**，如果不是几何特性的变化，例如只是背景颜色的变化，这些变化会引起**重绘**。

###如何触发回流

*	可见的DOM元素被删除或添加
*	元素位置改变
*	元素盒模型变化（内外边距、边框、高度和宽度等）
*	内容变化，例如文本变化、图片替换为一个不同的大小
*	页面渲染初始化
*	窗口大小改变

###缓存与刷新渲染树的改变
获取下列属性或使用下列方法时，会立即刷新渲染树的改变：

*	offsetTop，offsetLeft，offsetWidth，offsetHeight
*	scrollTop，scrollLeft，scrollWidth，scrollHeight
*	clientTop，clientLeft，clientWidth，clientHeight
*	getComputedStyle()，currentStyle（IE）

###减少重绘与回流
合并多次DOM或样式改变，一次性处理这些改变。

例如：
```javascript
var el = document.getElementById('mydiv');
el.style.borderLeft = '1px';
el.style.borderRight = '2px';
el.style.padding = '5px';
```
三次改变盒模型，会引起三次回流。

更有效的方式是使用``cssText``，一次性应用改变。

```javascript
var el = document.getElementById('mydiv');
el.style.cssText = 'border-left: 1px; border-right: 2px; padding: 5px;';
```
或者给元素添加class，并为class设置样式：
```javascript
var el = document.getElementById('mydiv');
el.className = 'active';
```

批处理DOM变化有以下三种方式：

*	隐藏元素，应用变化，显示元素
*	使用``documentFragment``创建脱离DOM的子树，然后将子树添加到DOM中
*	复制原始元素到脱离DOM的节点中，修改元素，再用其替换原始元素

推荐使用第二种方法，因为第二种方法引起的回流和DOM操作少。

###动画元素脱离流处理

*	要应用动画的元素使用绝对定位，将其从布局流中取出
*	应用动画，动画执行时，会引起部分页面的重绘
*	动画结束，恢复原来的定位，这样只会下推文档一次

##事件代理
事件触发分为捕获、目标、冒泡阶段，IE不支持捕获，但是冒泡已经可以很好得处理事件代理了。直接看代码：

```javascript
document.getElementById('menu').onclick = function(e) {
	e = e || window.event;
	var target = e.target || e.srcElement;
	var pageid, hrefparts;
	// 只对连接有效
	// exit the function on non-link clicks
	if (target.nodeName !== 'A') {
		return;
	}
	// 从连接中获取页面ID
	hrefparts = target.href.split('/');
	pageid = hrefparts[hrefparts.length - 1];
	pageid = pageid.replace('.html', '');
	// 更新页面
	ajaxRequest('xhr.php?page=' + id, updatePageContents);
	// 阻止默认动作和停止冒泡
	if (typeof e.preventDefault === 'function') {
		e.preventDefault();
		e.stopPropagation();
	} else {
		e.returnValue = false;
		e.cancelBubble = true;
	}
};
```

使用事件代理的一般步骤：

*	通过事件对象获取目标元素（target）
*	停止向上冒泡（可选）
*	阻止默认行为（可选）

##总结

*	减少DOM访问，尽量使用Javascript来操作
*	使用本地变量存储要多次访问的DOM元素
*	要小心处理HTML集合，因为他们是动态的，迭代的时候缓存集合的length，复制集合元素到数组中再处理
*	使用更快的API，例如``querySelectorAll()``和``firstElementChild``
*	注意重绘和回流，批量改变样式和操作DOM，脱离文档操作DOM，减少获取布局信息的次数
*	动画使用绝对定位，使用拖拽代理
*	使用事件代理减少事件监听处理