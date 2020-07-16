---
layout: post
title: vue常见问题
date: 2020-07-15 16:00:26
tags: vue
categories: vue
---

# vue常见问题

## 前言

这个系列的博客将记录我在vue中遇到的一些问题和解决思路

## vue变量复制

参考链接：https://blog.csdn.net/weixin_42693164/article/details/102546335

我在使用vue变量的传递时出现了下面的问题

```
//我们有两个对象 a,b  都有一个x的属性
//初始化
this.a = {x:1}
this.b = {x:2} 
console.log(this.a,this.b) //{x: 1} {x: 2}
this.a = this.b
console.log(this.a,this.b) //{x: 2} {x: 2}
this.b.x =10
console.log(this.a,this.b) //{x: 10} {x: 10}
this.a.x =11
console.log(this.a,this.b) //{x: 11} {x: 11}
```

可以发现 当我们把b赋值给a时，后续对a或者b的修改 都会影响到另外一个变量

但是如果是基本类型 就不会有这种现象，如字符串类型或者整形

出现这种情况的原因是js的浅复制和深复制

首先复习一下，在js中有两种数据类型

```
（1） 基本数据类型：number、string、boolean、null、undefined、symbol（ES6）
（2） 引用数据类型：object、function（函数实际也是对象）
```

其次js有两种内存模式

```
（1）栈内存：空间小，有默认大小
（2）堆内存：空间大，可以自适应大小
所以从他们的特点很容易看出栈内存一般用于存储基本数据类型，而堆内存一般用于存储引用数据类型
```

我们知道一般的js都是在栈中由上至下执行的（有一些资料显示，js并没有从严格意义上去区分栈和堆，在一些场景下也是有所区分的，例如：浅复制和深复制），而堆内存的数据一般是在栈内存存了一个地址指向对应的堆内存，js执行的时候便通过这个地址来找到对应的堆内存和数据

所以这个问题便知道了发生的原因，我们在进行引用数据类型的复制时，直接将引用数据类型赋给了另外一个引用数据类型，那么实际在栈中，只是简单的复制了一下栈中存放的地址，并不是这个对象实际的值，同一个地址指向的当然就是同一个值。所以改了一个，另外一个也会发生变化。这个也被称为浅复制。

如何解决：深复制

```
最简单的深复制
var a = JSON.parse(JSON.stringify(b))
```

这个实际就是利用JSON.stringify（obj）将对象的内容转换成字符串，那么在栈内存中就会给他一个空间存储，之后这个字符串想去外面的世界看看有多精彩，通过JSON.parse还原回原来的对象，带着家一起出走到了堆内存中，在栈内存中留下了联系方式（地址），从而实现了深复制。

注意：JSON.parse(JSON.stringify(obj))不能复制函数类型，obj也是要可以枚举才行，在IE7以下浏览器会报错

```
递归方法实现深复制

//递归的方法:

function deepClone(ob) {
	//根据不同对象类型赋值
	let cloneObj= Array.isArray(obj)?[]:{}
	//传入值不能为空且为对象类型
	if(obj & typeof obj === "object" ){
	//对象的遍历方法
		for(key in obJ){
			//bj. hasOwnProperty (key)是验证对象自身属性中是否有指定的属性，会忽略掉原型链上继承的属性
			//在遍历对象时要包略维承属性，因为for... in循环只能遍历到可枚举属性(一般默认enumerable=true )
			//如果检测一个对象任意属性可以用obj. prototype. hasOwmProperty (key)
			1f (obj.hasOmnProperty (key)){
				//判断子元素是否为对象
				if(obj,[key]&&typeof obj[key] === "object"){
					//执行递归
					cloneObj = deepClone(obj[key])
				}else{
					//不是就直按赋值
					clone0bj = obyfkey]
				}
			}
		}
	}
	//返回结果
	return cloneObj
}

```

特别说明：对于js基本数据类型的赋值谈不上是深复制,因为每每声明一个变量时，栈内存中就会给其一个固定空间，如下面的a和b，实际他们两个都在各自的空间，空间里面都放着实际值，互不干扰。

番外：Null的数据类型实际是object类型，为何会在基本数据类型里面呢？？？
查看资料很多都说是一个将错就错的bug
实际想一下，很多关于提高性能的书里面都有提到一个“对象不用时就obj=null“，这是因为浏览器有一个垃圾回收机制，当检测到这原有的堆内存没有被占用了就会被销毁，null就相当于一个对象的空地址,值就是固定的，占用空间是固定的，这可能就是将错就错的原因，没有用的全局对象不手动销毁，浏览器在不能检测这个变量何时不再使用，就不会销毁，会造成内存泄漏。



## vue alert问题

```
//有时候我们在vue里面直接写
alert("hello")
//是不起效果的
//我们可以更改写法
widow.alert("hello")
//这样是有效果的
//原因是vue this的指向问题
```

