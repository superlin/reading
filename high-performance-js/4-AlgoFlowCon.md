算法和流程控制
===

##循环
普通循环：
```javascript
for (var i=0; i < items.length; i++){
	process(items[i]);
}
```
每次都要查询items.length，改进版：
```javascript
for (var i=0, len=items.length; i < len; i++){
	process(items[i]);
}
```
递减版本：
```javascript
for (var i=items.length-1; i >= 0; i--){
	process(items[i]);
}
```
虽然是简单的改变，但是性能要提高50%-60%

##条件
大多数情况下``switch``要比``if-else``快一点，只有在条件数量很多时才会快很多，``if-else``最好用于只有两个条件或者条件很少。

``if-else``常常需要优化，最简单的优化方式是**最常见的条件放在最前面**，如下例所示，小于5的值出现最多，先判断：
```javascript
if (value < 5) {
	//do something
} else if (value > 5 && value < 10) {
	//do something
} else {
	//do something
}
```
第二种优化方式是**将一系列的if-else组织为嵌套的if-else语句**
```javascript
if (value == 0){
	return result0;
} else if (value == 1){
	return result1;
} else if (value == 2){
	return result2;
} else {
	return result3;
}
```
最差需要比较3次，可以按如下优化，最多需要比较2次：
```javascript
if (value < 2){
	if (value == 0){
		return result0;
	} else {
		return result1;
	}
} else {
	if (value == 2){
		return result2;
	} else {
		return result3;
	}
}
```

>译者注：有点二分的味道

有时候``switch``和``if-else``都不是最好的方式，当有大量的不同的值要用来比较，之前的都比查找表慢。如下例所示：
```javascript
switch(value){
	case 0:
		return result0;
	case 1:
		return result1;
	case 2:
		return result2;
	case 3:
		return result3;
	case 4:
		return result4;
	case 5:
		return result5;
	case 6:
		return result6;
	case 7:
		return result7;
	case 8:
		return result8;
	case 9:
		return result9;
	default:
		return result10;
}
```
使用查找表优化：
```javascript
//定义结果数组
var results = [result0, result1, result2, result3, result4, result5,result6,result7, result8, result9, result10];

//返回正确的结果
return results[value];
```
按以上方式操作变成了查询一个数组元素或者对象成员。查找表一般用于值和键之间有某种逻辑对应关系。``switch``适用于每个键有不同的操作或者一系列操作。

##递归
各个浏览器的调用堆栈大小一般是固定的，除了IE，IE的调用堆栈的大小和系统内存大小有关。调用堆栈溢出后一般浏览器都会抛出异常错误，chrome是唯一的不会报错的浏览器。

递归有两种形式，第一种是直接递归：
```javascript
function recurse(){
	recurse();
}
recurse();
```
另外一种是使用两个函数的精妙形式：
```javascript
function first(){
	second();
}
function second(){
	first();
}
first();
```

任何使用递归实现的算法都可以使用循环迭代实现。

为了减少递归计算次数，可以将之前计算的值保存起来，从而不用重复计算，这就是**记忆**：
```javascript
function memoize(fundamental, cache){
	cache = cache || {};
	var shell = function(arg){
		if (!cache.hasOwnProperty(arg)){
			cache[arg] = fundamental(arg);
		}
		return cache[arg];
	};
	return shell;
}
```
##总结
*	``for/while/do-while``的性能差不多
*	避免使用for-in循环，除非你要迭代一些未知的属性
*	提高循环性能的方式是减少每次循环的工作和减少迭代的次数
*	通常``switch``要比``if-else``快一些，但不总是好的解决方案
*	查找表是比``switch``或``if-else``更好更快的选择
*	调用堆栈的大小会限制递归的次数；栈溢出错误会阻止其他代码的执行
*	如果遇到栈溢出可以使用循环来处理或者使用记忆的方式减少重复的工作