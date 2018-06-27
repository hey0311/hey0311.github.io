---
layout: post
title: "JavaScript运行机制"
date: 2018-06-25 
description: "JavaScript,运行机制"
tag: JavaScript
---   

JavaScript是一门单线程语言，因此所有任务都要排队来进行。JavaScript中的任务可分为两类：同步任务和异步任务。异步任务要在同步任务执行完之后才执行。

### 事件循环（Event Loop）

JavaScript中有一个事件循环（Event Loop）的概念，同步任务在执行栈中执行,异步请求发起后,只要有了结果,就在任务队列中放置一个事件,系统在执行完执行栈中的任务后,将读取任务队列,将其放到执行栈开始执行.然后继续执行完之后读取任务队列。

### setTimeout定时器

setTimeout为一个定时器函数，它表示在指定时间后挂载一个任务到异步队列中。因此，并不意味着你设定一个1秒的定时器，1秒后定时器中的任务就一定会被执行，因为在异步队列中的定时器任务必须在同步任务执行完之后才开始执行。如果同步任务的运行时间大于1秒，则定时器中的任务并不能在执行的1秒后执行。

	for (var i=0;i<4;i++){
		setTimeout(function(){
			console.log(i);
		},1000);
	}

以上代码将在1秒后输出4个4.由于4次的for循环几乎一瞬间完成的，因此setTimeout的作用是1秒后挂载4个console.log(i)任务到异步队列。等同步任务执行完后（也是几乎一瞬间），然后等待1秒，开始执行异步任务1，console.log(i)被放置到全局环境中开始执行，而全局环境中的i值为4，因此输出4.同理，4个异步任务几乎是一瞬间输出了4个4.

### JavaScript运行机制的应用

**1.分割任务，避免浏览器未响应**

《高性能JavaScript》一书中介绍了使用定时器来分割长时间运行的任务。大多数浏览器让一个UI单线程共同执行JavaScript和更新用户界面，同一时间只能执行其中一种操作，这意味着JavaScript代码正在执行时用户界面无法相应输入，反之亦然。

看下面一段代码

	var div=document.getElementById('demo');
	for(var i=0x100000;i<0xFFFFFF;i++){
		div.style.backgroundColor='#'+i.toString(16);
	}

以上代码中整个循环是一个JavaScript任务，在该JavaScript任务完成之前，浏览器的更新界面任务会被暂停，等循环全完成之后才会更新界面。因此运行以上代码表现的结果是浏览器页面未响应一段时间，在未响应的时间内div元素也不会依次变化颜色，只会显示最后的白色。

可以使用定时器将每次循环内的任务定义为一个异步任务，避免浏览器卡死，且实时更新用户界面，如下：

	var div=document.getElementById('demo')
	var timer,i=0x100000;
	function func(){
		timer=setTimeout(func,0);
		div.style.background='#'+i.toString(16);
		if(i++==0xFFFFFF) clearInterval(timer);
	}
	timer=setTimeout(func,0);

如果是循环处理数组，《高性能JavaScript》中提供了一种比较通用的任务分割函数：

	function processArray(items,process,callback){
		var todo=items.concat();
		setTiemout(function(){
			process(todo.shift());
			if(todo.shift()){
				setTimeout(arguments.callee,25);
			}else{
				callback(items);
			}
		},25)
	}

我在工作中多次运用到定时器来优化产品性能，例如，在表格中渲染迷你图，表格的每一行都包含若干图形，如果在表格行的循环中动态加载迷你图，将导致浏览器未响应，因此采用定时器分割任务。又例如，threejs中的3D图渲染优化。当3D模型含有大量子模型，threejs会循环处理所有子模型，这将导致浏览器卡死，也可采用定时器来分割处理任务。

**2.防抖动函数（debounce）**

防抖动函数可以防止某个函数在短时间内被密集调用，代码如下：


	function debounce(fn,delay){
		var timer=null;
		return function(){
			var _this=this;
			var args=arguments;
			clearTimeout(timer);
			timer=setTimeout(function(){
				fn.apply(_this,args);
			},delay)
		}
	}

debounce方法返回一个新版的该函数，这个新版函数调用后，只有在指定时间内没有新的调用，才会执行，否则就重新计时。

lodash插件中就包含了防抖动函数，引入了该插件的可以使用_.debounce()方法。

**3.清除计时器**

setTimeout和setInterval都是返回整数且连续的,使用clearTimeout和clearInterval函数取消定时器,取消所有setTimeout:

	var gid=setInterval(clearAllTimeout,0);
	function clearAllTimeout(){
		var id=setTimeout(function(){},0);
		while(id>0){
			if(id !== gid){
				clearTimeout(id);
			}
			id--;
		}
	}

参考资料：

[1] Nicbolas C Z.高性能JavaScript[M].电子工业出版社，2015.8

