---
layout: post
title: swift programming language guide
description: swift语言编程指导
category: blog
tag: ios,swift
---

## 基本语法示例

下面是基本的语法demo：

	import Cocoa

	// 值永远不会隐式转换成其它类型，如果要转换值类型，需要显示转换
	let label = "The width is";
	let width = 94;
	let widthLabel = label + String(width);
	
	
	// 把值转化成字符串：利用括号加反斜杠
	let apples = 3;
	let origanes = 5;
	let appleSumm = "I have \(apples) apples.";
	let origanesSumm = "I have \(origanes) origanes.";
	
	
	// 创建数组
	var shoppingList = ["catfish","water","tulips","blue paint"];
	shoppingList[1] = "bottle of water";
	
	// 创建字典
	var occupations = [
	    "Malcolm":"Captain",
	    "Kaylee":"Mechanic",
	];
	occupations["Jayne"] = "Public Relations";
	
	
	// 创建空数组和空字典
	let emptyArray = [String]();
	let emptyDictionary = [String:Float]();
	
	
	/* 
	 控制流
	 */
	
	let individualScores = [75,43,103,87,12];
	var teamScore = 0;
	for score in individualScores{
	    if score > 50 {
	        teamScore += 3;
	    }else{
	        teamScore += 1;
	    }
	}
	print(teamScore);
	
	var optionalString:String? = "Hello";
	print(optionalString == nil);
	
	var optionalName:String? = "John Appleseed";
	var greeting = "Hello!";
	if let name = optionalName{
	    greeting = "Hello,\(name)";
	}
	
	// 整数范围
	let minValue = UInt8.min;
	let maxValue = UInt8.max;
	
	// 类型转化
	let twoThousand: UInt16 = 2_000;
	let one: UInt8 = 1;
	let twoThousandAndOne = twoThousand + UInt16(one);
	
	// 类型别名
	typealias AudioSample = UInt16;
	var maxAmplitudeFound = AudioSample.min;
	print(maxAmplitudeFound);

### 分支语句

#### if语句

Swift中的if使用基本上和OC一致，具有以下特点：

* swift中的if可以省略()
* swift 中哪怕if后面只有一条语句，也不能省略{}
* 在 C和OC中，有一个概念非0即真；在swift中，条件只能是bool值，取值只有两个 true/false

demo：
	
	let num = 10;
	if num == 10{
	    print("OK");
	}

	if true{
	    print("OK");
	}

#### switch语句

Swift中的switch语句的特点：

* switch后面的()可以省略
* OC中的switch如果没有break会穿透，但Swift中不会
*  OC中如果要在case中定义变量，必须加上{}确定作用域，而swift中不用
*  OC中default的位置可以随便写，只有所有case都不满足才会执行default而swift中的default只能写在最后
*  OC中的default可以省略,Swift中“大部分”情况下不能省略

demo:

	var num = 10;
	switch num{
	    case 1:
	        print("1");
	        break;
	    case 5:
	        print("5");
	        break;
	    case 10:
	        print("10");
	        break;
	    default:
	        print("other");
	        break;
	}

### 循环

#### for循环

下面罗列的是swift特色循环：

1) `0..<10` 代表一个区间范围，从0开始到9，包含头不包含尾

	for i in 0..<10
	{
	    print(i);
	}

2）`_`代表忽略，如果不关心某个参数，就可以使用`_` ,在Swift开发中 _使用频率非常高

	for _ in 0..<10
	{
	    print("lng");
	}
	
3）0...10 代表一个区间范围 从0开始到10 包含头又包含尾

	for i in 0...10
	{
	    print(i);
	}

注意:区间之间是不可以有空格的

#### while循环

Swift中国的while循环和OC中差不多，而且在开发中很少使用while

	var a = 0;
	while a < 10
	{
	    print(a);
	    a += 1;
	}

#### do...while循环

Swift升级到2.0之后, do while循环发生了很大的变化,do while循环没有do了，因为do被用作捕获异常了

	var b = 0;
	repeat{
	    print(b);
	    b += 1;
	}while b < 10;
	

### 分支

在OC中 if else 可以使用三目运算符来简写的。

	let num = 10;
	let res = (num == 5) ? 5 : 10;
	print(res);
	
### 可选类型

可选类型的意思就是可以有，也可以没有  Optional ?

一个方法或数据类型后面有 `?` ,就代表返回的类型是一个可选类型。

! 表示告诉编译器，可选类型中一定有值，强制解析。如果可选类型中没有值而又进行了强制解析，那么程序会崩溃

	let url = NSURL(string:"http://www.baidu.com");
	print(url);
	print(url!);
	
### 数组

数组的定义：

	var arr0 = [1,2,3];
	var arr1:Array = [1,2,3];
	var arr2:Array<Int> = [1,2,3];
	var arr3:[Int] = [1,2,3];

空数组：

	var arr6 = [Int]();
	var arr7 = Array<Int>();

数组元素的类型：如果想明确表示数组中存放的是不同类型的数据，可以用Any关键字，表示数组中可以存放不同类型的数据。

	var arr10:Array<Any> = [1,"qi",1.75];
	
获取数组的长度：
	
	var arr12 = [1,2,3];
	print(arr12.count);

判断数组是否为空：

	print(arr12.isEmpty);
	
检索：

	print(arr12[0]);

追加:

	arr12.append(4);
	print(arr12);
	arr12 += [5]; // 可以自己搞追加
	print(arr12);

插入:

	arr12.insert(6, at: 0);
	print(arr12);

删除:

	arr12.remove(at: 0);
	print(arr12);
	
	arr12.removeLast();
	print(arr12);
	
	arr12.removeAll();
	print(arr12);
	
	var arr13 = [1,2,3];
	arr13.removeAll(keepingCapacity: true);//是否保持容量, 如果为true, 即便删除了容量依然存在
	print(arr13.capacity);
	//注意: 如果数组是一个不可变数组不能更新/插入和删除
	
遍历：

	var arr1 = [1,2,3];
	
	// 方法一
	for number in arr1
	{
	    print(number);
	}
	// 方法二：
	for i in 0..<arr1.count
	{
	    print(arr1[i]);
	}

	// 取出某个区间的值
	for number in arr1[0..<3]
	{
	    print(number);
	}
	
### 字典

字典中的key一定要是可以hash的(String, Int, Float, Double, Bool), value没有要求

创建字典：

	var dict = ["name":"qi","age":"26"];
	print(dict);
	
	var dict1:Dictionary<String,Any> = ["name":"qi","age":26];
	print(dict1);
	
	var dict2:[String:Any] = ["name":"lnj", "age":30]
	print(dict2);
	
	var dict3:[String:Any] = Dictionary(dictionaryLiteral: ("name", "lnj"), ("age", 30))
	print(dict3)

修改字典中某一个key的值：

	dict3["name"] = "qiqi";
	print(dict3);
	
	dict3.updateValue("qi", forKey: "name");
	print(dict3);

遍历字典：

	for (key , value) in dict3
	{
	    print("key = \(key), value = \(value)")
	}
	
	print("-------------");
	for key in dict3.keys
	{
	    print(key);
	}
	
	print("-------------");
	for value in dict3.values
	{
	    print(value);
	}