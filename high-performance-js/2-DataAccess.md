数据访问
==========
Javascript中有四种基本的数据访问方式：``字面量``、``变量``、``数组元素``、``对象成员``，几乎所有的浏览器中：字面量和本地变量的访问速度要比数组元素和对象成员的访问速度快的多

##作用域
###作用域链与变量解析
Javascript中的函数都是对象，更具体来说，函数都是Function的实例。每个函数都有自己的属性就和其他对象一样，函数的一个重要属性就是[[scope]]。[[scope]]就是函数的作用域链，它是在函数创建时就生成的。例如如下代码：
```javascript
function add(num1, num2){
	var sum = num1 + num2;
	return sum;
}
```
上面的``add``函数的作用域链值包含全局对象。

![scope of add](http://superlin.github.io/reading/high-performance-js/1-add.png)

当函数执行时，就会创建一个“执行上下文（excution context）”的内部对象，每个运行期上下文都有自己的作用域链，用于标识符解析，当运行期上下文被创建时，而它的作用域链初始化为当前运行函数的[[Scope]]所包含的对象。

这些值按照它们出现在函数中的顺序被复制到运行期上下文的作用域链中。它们共同组成了一个新的对象，叫“活动对象(activation object)”，该对象包含了函数的所有**形式参数**、**arguments集合**、**局部变量**以及**this**，然后此对象会被推入作用域链的前端，当运行期上下文被销毁，活动对象也随之销毁。新的作用域链如下图所示：

![scope of add execution](http://superlin.github.io/reading/high-performance-js/1-run-add.png)

###改变作用域链
函数每次执行时对应的运行期上下文都是独一无二的，所以多次调用同一个函数就会导致创建多个运行期上下文，当函数执行完毕，执行上下文会被销毁。每一个运行期上下文都和一个作用域链关联。一般情况下，在运行期上下文运行的过程中，其作用域链只会被``with``语句和``catch``语句影响。

``with``语句是对象的快捷应用方式，用来避免书写重复代码。例如：
```javascript
function initUI(){
  with(document){
    var bd=body,
        links=getElementsByTagName("a"),
        i=0,
        len=links.length;
    while(i < len){
      update(links[i++]);
    }
    getElementById("btnInit").onclick=function(){
      doSomething();
    };
  }
}
```
当代码运行到``with``语句时，运行期上下文的作用域链临时被改变了。一个新的可变对象被创建，它包含了参数指定的对象的所有属性。这个对象将被推入作用域链的头部，这意味着函数的所有局部变量现在处于第二个作用域链对象中，因此访问代价更高了。如下图所示：

![with改变scope](http://superlin.github.io/reading/high-performance-js/2-with.png)

``catch``也会改变作用域链，要注意的是不要将``catch``作为解决错误的方式。为了减少``catch``对于性能的影响，应该尽量减少``catch``中的代码量，一般的做法是使用一个错误处理方法。
```javascript
try {
  doSomething();
} catch(ex) {
  handleError(ex); //委托给处理器方法
}
```

###动态作用域
``with``、``catcch``、``eval``都有动态作用域，动态作用域只是在代码执行时存在，但是无法通过静态分析他们。

###闭包
闭包用一句话形容就是：函数里再定义函数，内部函数可以访问外部函数的变量，这样就形成了闭包。闭包的形成和作用域有密切的关系，例如下面的例子：
```javascript
function assignEvents(){
	var id = "xdi9592";
	document.getElementById("save-btn").onclick = function(event){
		saveDocument(id);
	};
}
```
``assignEvents``函数中为一个DOM元素绑定了事件处理函数，这个内部函数使用了外部的``assignEvents``函数中的变量``id``，这时候就创建了一个闭包。

当``assignEvents``函数创建的时候其作用域（[[scope]]）中只有全局对象，执行``assignEvents``函数的时候会创建一个活动对象放于作用域链的顶部，正常情况下``assignEvents``函数运行结束后活动对象就会被垃圾回收。但是当运行到内部函数的定义处时，就会创建内部函数的作用域（[[scope]]），这个内部函数的作用域包含父级的活动对象和全局对象。这样就形成了闭包，当``assignEvents``函数执行结束时，由于有对象引用``assignEvents``函数的活动对象，那么这个活动对象就不会被垃圾回收，这样就形成了闭包。如下图所示：

![闭包的scope](http://superlin.github.io/reading/high-performance-js/3-closure.jpg)

闭包的缺点：

1.	要保留外部函数的AO，需要更多的内存
2. IE中会造成变量泄露
3. 访问out-of-scope变量，比访问本地变量慢

##对象成员
###原型
对象都用两种成员：实例成员、原型成员，实例成员就是自己的成员，原型成员是继承自对应原型的成员。例如：
```javascript
var book = {
	title: "High Performance JavaScript",
	publisher: "Yahoo! Press"
};
```
下图显示了book实例和其原型的关系：
![实例和原型的关系](http://superlin.github.io/reading/high-performance-js/4-proto.png)

###原型链
创建一个函数时，会自动生成对应的``prototype``对象（显式原型），创建实例时就会将实例的``__proto__``属性（隐式原型）指向显式原型对象，因此显式原型是共享的。如下例所示：
```javascript
function Book(title, publisher){
	this.title = title;
	this.publisher = publisher;
}
Book.prototype.sayTitle = function(){
	alert(this.title);
};
var book1 = new Book("High Performance JavaScript", "Yahoo! Press");
var book2 = new Book("JavaScript: The Good Parts", "Yahoo! Press");
```
原型链如下图所示：
![原型链](http://superlin.github.io/reading/high-performance-js/5-prototype-chain.png)

###嵌套成员
常常会看到``window.location.href``这种Javascript代码，这样的嵌套对象会使Javascript引擎每次都扫描所有的成员来解析成员，所以嵌套成员越深，访问速度越慢。执行``location.href``总是比``window.location.href``更快。

###缓存对象成员
对于那些不止访问一次的成员可以使用本地变量存储起来，例子如下：
```javascript
function toggle(element){
	if (YAHOO.util.Dom.hasClass(element, "selected")){
		YAHOO.util.Dom.removeClass(element, "selected");
		return false;
	} else {
		YAHOO.util.Dom.addClass(element, "selected");
		return true;
	}
}
```
可以使用下面的方式改造来提高性能：
```javascript
function toggle(element){
	var Dom = YAHOO.util.Dom;
	if (Dom.hasClass(element, "selected")){
		Dom.removeClass(element, "selected");
		return false;
	} else {
		Dom.addClass(element, "selected");
		return true;
	}
}
```

##总结

*	字面量和本地变量的访问速度要比数组元素和对象成员的访问速度快的多
*	本地变量的访问速度要比out-of-scope变量的访问速度快，因为他们在作用域链的第一个VO中。越处于作用域链的下端访问时间越长，全局变量总是在作用域链的最下端所以他们的访问速度最慢
*	避免使用with和catch，他们会改变作用域链
*	嵌套的对象成员会影响性能，应将其最小化
*	沿着原型链查找方法或属性时，查找路径越长访问越慢
*	通常来说，你可以将经常访问的数组元、对象成员以及out-of-scope变量保存为本地变量来提高访问速度
