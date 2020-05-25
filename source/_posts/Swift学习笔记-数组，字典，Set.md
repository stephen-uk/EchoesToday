---
title: Swift学习笔记--数组，字典，Set
date: 2020-04-29 00:17:16
tags: Swift学习笔记
---

# 集合类型 -- 数组，字典，Set

“在所有的编程语言中，元素的集合都是最重要的数据类型”

摘录来自: Chris Eidhof. “Swift 进阶。” Apple Books. 

## 0x1 数组

数组是一个容器，它以有序的方式存储相同类型的元素，并且允许随机访问每个元素，根据索引我们可以通过时间复杂度为O(1)的情况下获取到指定元素，这一特性也成为了hashmap的实现原理。

### 数组可变性

```swift
// 斐波那契数列
let fibs = [0, 1, 1, 2, 3, 5]
//当前数组不可变，不能进行插入，删除等操作

// 斐波那契数列
var fibs = [0, 1, 1, 2, 3, 5]
//当前数组为可变数组，可以对数组进行插入删除等操作
```

需要注意let，使用 let 定义一个指向类实例的引用类型时，它保证的是这个引用永远不会发生变化，也就是说，不能再给这个引用赋一个新的值，但是这个引用所指向的对象却是可以改变的，所以我们可以确定的是在这个地方 数组并不是引用类型的对象。

所以会有👇的操作

```swift
var x = [1,2,3]
var y = x
x.append(4)
//x = [1,2,3,4]
//y = [1,2,3]
```

> 在 Swift 中，所有的基本类型：整数（Integer）、浮点数（floating-point）、布尔值（Boolean）、字符串（string)、数组（array）和字典（dictionary），都是值类型，并且在底层都是以结构体的形式所实现。类是引用类型。

<!-- more -->

### 数组的遍历及索引

Swift 数组提供了所有常规操作方法，像是 isEmpty 和 count。数组也允许通过下标直接访问指定索引上的元素，像是 fibs[3]。不过要确保索引值没有超出范围。

遍历数组的一些方法

- `for x in array`
- `for x in array.dropFirst()`  //除了第一个元素以外的数组其余部分
- `for x in array.dropLast(5)` //除了最后 5 个元素以外的数组 
- `for (num, element) in collection.enumerated() ` 列举数组中的元素和对应的下标
- `if let idx = array.index { someMatchingLogic($0) } `  寻找一个指定元素的位置 !~
- `array.map { someTransformation($0) }`  //对数组中的所有元素进行变形
- `array.filter { someCriteria($0) }`  //筛选出符合某个特定标准的元素



### 数组变形

map 对数组中的每一个元素进行转换操作并返回新的数组。举例👇

```swift
//原始做法
var squared: [Int] = []
for fib in fibs {
	squared.append(fib * fib)
}
squared // [0, 1, 1, 4, 9, 25]”

//Map的做法
let squares = fibs.map { fib in fib * fib }
squares // [0, 1, 1, 4, 9, 25]
```

> map 方法是方法将行为参数化，是一种函数式编程，简单的说就是一个高阶函数，一个函数的参数为另一个函数，那么这个函数就是一个高阶函数。

数组中类似的方法有

- map 和 flatMap — 对元素进行变换。

- filter — 只包含特定的元素。

- allSatisfy — 针对一个条件测试所有元素。

- reduce — 将元素聚合成一个值。

- forEach — 访问每个元素。

- sort(by:), sorted(by:), lexicographicallyPrecedes(_:by:), 和 partition(by:) — 重排元素。

- firstIndex(where:), lastIndex(where:), first(where:), last(where:), 和 contains(where:) — 一个元素是否存在？

- min(by:) 和 max(by:) — 找到所有元素中的最小或最大值。

- elementsEqual(_:by:) 和 starts(with:by:) — 将元素与另一个数组进行比较。

- split(whereSeparator:) — 把所有元素分成多个数组。

- prefix(while:) — 从头取元素直到条件不成立。

- drop(while:) — 当条件为真时，丢弃元素；一旦不为真，返回其余的元素 (和 prefix 类似，不过返回相反的集合)。

- removeAll(where:) — 删除所有符合条件的元素


>map 与 flatMap的区别：
>
>flatMap 跟 map都是 对数组的每一个元素进行变换不一样的地方主要是两点。
>
>- 降维操作，如果数组是一个二维数组，那么flatMap会将二维数组展开，并返回一个一维数组.
>- 过滤nil，如果数组中有nil或者是操作完成后有nil元素，则会过滤掉
>- 可以将多个数组合并成一个。



## 0x2 字典

字典包含键以及它们所对应的值，其中每个键是唯一的。通过键来获取值所花费的平均时间是常数量级的，作为对比，在数组中搜寻一个特定元素所花的时间将与数组尺寸成正比。和数组有所不同，字典是无序的，使用 for 循环来枚举字典中的键值对时，顺序是不确定的。

> 与通常使用的以键为参数的下标方法形成对比的是，作为实现 Collection 协议的一部分，字典也有一个接受索引的下标方法。就像数组一样，当用一个无效的索引调用这个方法时，程序会崩溃。
>
> 需要注意的是在swift中字典的定义是使用中括号，而非oc中的花括号，而且这种定义的方式需要设置默认值，不然推断不出来key的类型 ` var hues = ["Heliotrope": 296, "Coral": 16, "Aquamarine": 156]`

### 字典可变性

与数组一样，字典也是值类型的数据结构，所以let 定义的字典是不可变的，var 声明的可变字典，可以通过 map[key] = value 的方式添加一条数据，可以通过map[key] = nil的方式移除数据，

### 字典比较实用的方法

合并字典操作 `merge(_:uniquingKeysWith:)`, 如果有相同的key 可以通过返回的参数来确定使用哪个字典的key。

```swift
var map = Dictionary<String, Any>()
map["Hello1"] = "World1"
map["Hello2"] = "World2"
map["Hello3"] = "World3"


var map2 = Dictionary<String, Any>()
map2["Hello4"] = "World4"
map2["Hello1"] = "World5"
map2["Hello6"] = "World6"

map.merge(map2) { (a, b) -> Any in
    return a
}
print(map)
```

result :

`["Hello4": "World4", "Hello1": "World1", "Hello3": "World3", "Hello2": "World2", "Hello6": "World6"]`

### Hashable要求

字典通过对Key的hashValue来为每个键在其底层作为存储的数组上指定一个位置。也就是Dictionary要求它的key类型遵守Hashable协议的，标准库中的所有基本数据类型都是遵守Hashable协议的包括**字符串，整数，浮点数以及布尔值**。



## 0x3 Set

集合是一组无序的元素，每个元素只会出现一次。和Dictionary一样，Set也是通过哈希表实现的，并拥有类似的性能特征和要求。集合中的元素必须满足Hashable协议。

集合在Swift中存在的类型有Set，OptionSet，IndexSet，CharacterSet，其中IndexSet和CharacterSet比较特殊。

> IndexSet 表示了一个由正整数组成的集合,他的内部是通过一组范围列表实现，IndexSet里其实只存储了选择的首位和末位两个整数值。所以它的存储空间相对Set<Int>的方式要节省节省。
>
> CharacterSet 与IndexSet类似，不过是用于字符类型。

## 0x4 Range

范围代表的是两个值的区间，可以通过 `..<` 创建一个不包含上边界的半开范围，或者使用 ... 创建一个包含上下边界的闭合范围

```swift
// 0 到 9, 不包含 10
let singleDigitNumbers = 0..<10
Array(singleDigitNumbers) // [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
// 包含 "z"
let lowercaseLetters = Character("a")...Character("z")
//从0 开始到无穷大，
let fromZero = 0...
let upToZ = ..<Character("z")
```



