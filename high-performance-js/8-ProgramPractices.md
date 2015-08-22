编程实践
===
##避免二次解析
``eval()``，``Function()``构造器，``setTimeout()``和``setInterval()``会引起二次解析，比直接执行代码的代价高，执行时间更长，尽量避免使用。
##使用对象/数组直接量
直接量解析很快，而且代码占用减少空间。
##不要重复工作
在计算机科学领域最重要的性能优化技术之一是避免重复的工作。避免重复工作的概念实际上意味着两件事：不要做不必要的工作，不要重复做已经完成的工作。第一部分通常认为代码应当重构。第二部分——不要重复工作——通常难以确定，因为工作可能因为各种原因而在很多地方被重复。

以事件句柄的添加和删除为例，典型的跨浏览器代码如下：

```javascript
function addHandler(target, eventType, handler){
	if (target.addEventListener){ //DOM2 Events
		target.addEventListener(eventType, handler, false);
	} else { //IE
		target.attachEvent("on" + eventType, handler);
	}
}

function removeHandler(target, eventType, handler){
	if (target.removeEventListener){ //DOM2 Events
		target.removeEventListener(eventType, handler, false);
	} else { //IE
		target.detachEvent("on" + eventType, handler);
	}
}
```
乍一看，这些函数为实现它们的目的已经足够优化。隐藏的性能问题在于每次函数调用时都执行重复工作。每一次，都进行同样的检查，看看某种方法是否存在。如果你假设target 唯一的值就是DOM 对象，而且用户不可能在页面加载时魔术般地改变浏览器，那么这种判断就是重复的。如果addHandler()一上来就调用addEventListener()那么每个后续调用都要出现这句代码。在每次调用中重复同样的工作是一种浪费，有多种办法避免这一点。

###延迟加载
第一种消除函数中重复工作的方法称作延迟加载。延迟加载意味着在信息被使用之前不做任何工作。对于上面的例子，改进如下：

```javascript
function addHandler(target, eventType, handler){
	// 重写函数
	if (target.addEventListener){ //DOM2 Events
		addHandler = function(target, eventType, handler){
			target.addEventListener(eventType, handler, false);
		};
	} else { //IE
		addHandler = function(target, eventType, handler){
			target.attachEvent("on" + eventType, handler);
		};
	}
	// 调用新函数
	addHandler(target, eventType, handler);
}

function removeHandler(target, eventType, handler){
	// 重写函数
	if (target.removeEventListener){ //DOM2 Events
		removeHandler = function(target, eventType, handler){
			target.addEventListener(eventType, handler, false);
		};
	} else { //IE
		removeHandler = function(target, eventType, handler){
			target.detachEvent("on" + eventType, handler);
		};
	}
	// 调用新函数
	removeHandler(target, eventType, handler);
}
```

这两个函数依照延迟加载模式实现。这两个方法第一次被调用时，检查一次并决定使用哪种方法附加或分离事件句柄。然后，原始函数就被包含适当操作的新函数覆盖了。最后调用新函数并将原始参数传给它。以后再调用addHandler()或者removeHandler()时不会再次检测，因为检测代码已经被新函数覆盖了。

###条件预加载

除延迟加载之外的另一种方法称为条件预加载，它在脚本加载之前提前进行检查，而不等待函数调用。这样做检测仍只是一次，但在此过程中来的更早。例如：

```javascript
var addHandler = document.body.addEventListener ?
	function(target, eventType, handler){
		target.addEventListener(eventType, handler, false);
	}：
	function(target, eventType, handler){
		target.attachEvent("on" + eventType, handler);
	};
var removeHandler = document.body.removeEventListener ?
	function(target, eventType, handler){
		target.removeEventListener(eventType, handler, false);
	}:
	function(target, eventType, handler){
		target.detachEvent("on" + eventType, handler);
	};
```
条件预加载确保所有函数调用时间相同。其代价是**在脚本加载时进行检测**。预加载适用于一个函数马上就会被用到，而且在整个页面生命周期中经常使用的场合。

##使用速度快的部分
虽然JavaScript 速度慢很容易被归咎于引擎，然而引擎通常是处理过程中最快的部分，实际上速度慢的是你的代码。引擎的某些部分比其它部分快很多，因为它们允许你绕过速度慢的部分。
###位操作符
JavaScript 中的数字按照IEEE-754 标准64 位格式存储。在位运算中，数字被转换为有符号32 位格式。每种操作均直接操作在这个32 位数上实现结果。尽管需要转换，这个过程与JavaScript 中其他数学和布尔运算相比还是非常快。

例如判断数字i的奇偶性，可以使用``i%2``或``i&1``，后者比前者要快50%（取决于浏览器）。

第二种使用位操作的技术称作位掩码。位掩码在计算机科学中是一种常用的技术，可同时判断多个布尔选项，快速地将数字转换为布尔标志数组。掩码中每个选项的值都等于2 的幂。例如：2，4，8，16，将目标值与掩码相与用于判断某一位上是1还是0。

###原生方法
无论你怎样优化JavaScript 代码，它永远不会比JavaScript 引擎提供的原生方法更快。其原因十分简单：JavaScript 的原生部分——在你写代码之前它们已经存在于浏览器之中了——都是用低级语言写的，诸如C++。这意味着这些方法被编译成机器码，作为浏览器的一部分，不像你的JavaScript 代码那样有那么多限制。

常犯的一个错误是在代码中进行复杂的数学运算，而没有使用内置Math对象中那些性能更好的版本。

另一个例子是选择器API，可以像使用CSS 选择器那样查询DOM 文档。CSS 查询被JavaScript 原生实现并通过jQuery 这个JavaScript 库推广开来。jQuery 引擎被认为是最快的CSS 查询引擎，但是它仍比原生方法慢。原生的querySelector()和querySelectorAll()方法完成它们的任务时，平均只需要基于JavaScript 的CSS 查询10%的时间。

##总结

*	通过避免使用eval()和Function()构造器，以及给setTimeout()和setInterval()传递函数参数而不是字符串参数，可以避免二次执行。
*	创建新对象和数组时使用对象直接量和数组直接量。它们比非直接量形式创建和初始化更快。
*	避免重复进行相同工作。当需要检测浏览器时，使用延迟加载或条件预加载。
*	当执行数学远算时，考虑使用位操作，它直接在数字底层进行操作。
*	原生方法总是比JavaScript 写的东西要快。尽量使用原生方法。