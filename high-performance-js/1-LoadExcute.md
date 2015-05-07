加载执行
==========
大多数浏览器使用一个进程来处理UI的更新和Javascript的执行，所以在某个特定的时间只有一件事情能发生。

无论Javascript代码是内联的还是使用外部文件引入的，页面的下载和渲染都需要等待脚本处理完毕，因为脚本在执行过程中可能会改变页面的内容，例如页面中部脚本执行``document.write()``。

##脚本位置
由于js加载和执行是阻塞的，因此将js文件放在页面顶部时，页面加载过程中会引起页面空白。因此**应将js文件放于页面底部**。

##脚本聚集
每次HTTP请求都需要建立连接和断开连接，多次请求需要的额外开销是很大的，因此下载一个100k的文件比下载4个25k的文件是要快的。因此要尽量减少脚本文件的数目。

可以使用combo-handled URL来下载多个脚本文件，例如：
```html
<script type="text/javascript" src="http://yui.yahooapis.com/combo?2.7.0/build/yahoo/yahoo-min.js&2.7.0/build/event/event-min.js"></script>
```

##非阻塞脚本
压缩脚本文件以及限制HTTP请求的数目只是创建响应式web应用的第一步。但是应用越来越复杂，无法将文件限制得很小，下载这样的文件会长时间的阻塞页面。

非阻塞脚本的秘诀在于在页面加载完成之后再去加载Javascript文件。例如在``window.onload``事件之后。

###推迟的脚本
使用``defer``属性的脚本会在页面加载完成之后才执行（``onload``事件之前），目前``defer``在IE4+，火狐3.5+，Chrome，Safari等都能得到很好的支持。

###动态脚本
可是使用dom操作动态添加脚本，例如：
```javascript
var script = document.createElement("script");
script.type = "text/javascript";
script.src = "file1.js";
document.getElementsByTagName("head")[0].appendChild(script);
```

火狐，opera，chrome，safari 3+在脚本加载完成后会触发``load``事件，但是IE中使用的是``readystatechange``事件。


``readyState``有5种可能值：

*	uninitialized：默认值
*	loading：已经开始下载
*	loaded：下载完成
*	interactive：数据已经下载完成但还不是完全可用
*	complete：所有数据都可用

完整的异步加载代码如下：
```javascript
function loadScript(url, callback){
	var script = document.createElement("script");
	script.type = "text/javascript";
	if (script.readyState){ //IE
		script.onreadystatechange = function(){
			if (script.readyState == "loaded" || script.readyState == "complete"){
				script.onreadystatechange = null;
				callback();
			}
		};
	} else { //Others
		script.onload = function(){
			callback();
		};
	}
	script.src = url;
	document.getElementsByTagName("head")[0].appendChild(script);
}
```

###xhr脚本注入
也可以使用``XMLHttpRequest``来请求文件，然后通过内联的方式插入脚本文件。代码如下：
```javascript
var xhr = new XMLHttpRequest();
xhr.open("get", "file1.js", true);
xhr.onreadystatechange = function(){
	if (xhr.readyState == 4){
		if (xhr.status >= 200 && xhr.status < 300 || xhr.status == 304){
			var script = document.createElement("script");
			script.type = "text/javascript";
			script.text = xhr.responseText;
			document.body.appendChild(script);
		}
	}
};
xhr.send(null);
```

###推荐的非阻塞脚本加载方式
推荐的加载方式分为两步：

1.	先将动态加载脚本的代码引入
2.	加载页面初始化的代码

示例如下所示：

```html
<script type="text/javascript" src="loader.js"></script>
<script type="text/javascript">
	loadScript("the-rest.js", function(){
		Application.init();
	});
</script>
```

这种方式的优点：

1.	不影响页面其他内容的展现
2.	第二个js文件加载完成后，DOM已经加载完成，可以进行交互了
3.	不用检查页面是否加载完成（window.onload事件）

##总结

*	将所有的``<script>``标签放置与页面底部，在``</body>``之前。这样可以保证脚本执行之前页面已经基本渲染完成
*	聚集多个脚本，下载的脚本越少，页面加载越快，无论脚本文件是内联形式还是外部链接。
*	有多种方式非阻塞的下载脚本文件：``defer``、动态创建``<script>``标签、使用xhr下载代码然后加入内联的脚本