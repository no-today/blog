---
title: 写让人理解的代码
date: 2019-05-10

categories:
    - Developer
---

代码的写法应该使理解代码的人所需要的时间最小化

<!--more-->

# 把信息装到名字里

## 有表现力的词

| 单词 | 更多选择 | 
|-------|:--------:| 
| send | deliver、dispatch、announce、distribute、route | 
| find | search、extract、locate、recover | 
| start | launch、create、begin、open | 
| make | create、set up、build、generate、compose、add、new |

## 带单位的命名

| 参数或变量 | 带单位的命名 | 
|-------|:--------:| 
| start(int delay) | delay -> delaySecs | 
| createCache(int size) | size -> sizeMB | 
| throttleDownload(float limit) | limit -> maxKB | 
| setHeight(float height) | height -> heightCM | 

## 给名字附加额外信息

| 场景 | 变量名 | 更好的名字 | 
|-------|:--------:|:-------:| 
| 一个纯文本的密码，需要加密后才可以使用 | password | plaintextPassword | 
| 一条用户评论，需要转义后显示 | comment | unescapedComment | 
| 已转化为UTF-8的HTML文本 | html | htmlUtf8 |
| 以"URL"方式编码的输入数据 | data | dataURLEncode |

**[谷歌代码规范](https://github.com/google/styleguide) | [中文](http://zh-google-styleguide.readthedocs.io/en/latest/)** 

- [Java 代码规范](https://google.github.io/styleguide/javaguide.html)
- [C++ 代码规范](https://google.github.io/styleguide/cppguide.html)
- [Python 代码规范](https://github.com/google/styleguide/blob/gh-pages/pyguide.md)
- [JavaScript 代码规范](https://google.github.io/styleguide/jsguide.html)

## 总结

1. 使用专业的词
2. 避免使用空泛的词
3. 给变量名带上附加信息
4. 为作用域更大的变量起一个长的名字
5. 有目的的使用大小写和下划线

# 让人不会误解的名字

不会误解的名字是最好的名字——阅读代码的人应该理解你的本意，并且不会有其他的理解。

- 表示上限和下限：max min
- 表示包含的范围：first last
- 表示包含、排除某个范围：begin end

命名一个 bool 类型的值，应该使用 is 、 has 这样的词来明确它所表达的含义。
避免使用反义的词，比如 disable。

要小心用户对特定词的期望。例如用户会认为 get 或 size 是一个轻量的方法。

# 写代码也需要审美？

1. 如果多个代码块做同样的事，尝试让它们有同样的剪影
2. 把代码块按照 “列” 对齐可以让代码更容易阅读
3. 如果一段代码中用到了 A、B、C，那么在使用它们的下方就应该保持顺序一致
4. 使用空行将大段的代码分为逻辑上的“段落”

# 什么样的注释是好的

| 标记 | 含义 | 
|-------|:--------:| 
| TODO | 等待处理的事情 | 
| FIXME | 已知无法运行的代码 | 
| HACK | 对一个问题不得不采用的简单粗暴的方案 | 
| XXX | 危险，这里有严重的问题 |

## 该写什么样的注释

注释的目的是帮助读代码的人了解作者在写代码时的思想。

## 什么地方不需要注释：

- 能从代码本身中快速推断的事实
- 用来装饰垃圾代码（比如拗口的方法名），实际上应该把名称修改好

## 记录你的想法

- 为什么代码写成这样而不是另一个样子的内在理由（“指导性批注”）
- 代码中的不足，使用像 TODO 或者 XXX 这样的标记
- 常量背后的意义，为什么是这个值？

## 在读者的立场思考

- 预料到代码中哪些部分会让读者说：“哎嘿？什么鬼” 给它们加上注释
- 为小白意料之外的行为加注释
- 在文件、类级别上使用“全局观”注释来解释所有的部分是如何一起工作的
- 用注释总结代码块，让读者不会迷茫在细节里

# 什么样的注释是好的

## 写出言简意赅的注释

- 当像 “这里” 和 “it” 这样的代词可能指代多个事物时，避免使用它们
- 尽量精确的描述方法行为
- 在注释中用精心挑选的输入/输出例子进行说明
- 声明代码的高层次意图，而非明显的细节
- 用含义丰富的词来使注释更加简洁

# 简化流程让代码易读

| 编程结构 | 高层次程序流程是人如何变得不清晰的 | 
|-------|:--------:| 
| 线程 | 不清楚什么时候会执行代码 | 
| 信号量/中断处理程序 | 有些代码随时都有可能执行 | 
| 异常 | 可能会从多个函数向上冒泡的执行 | 
| 匿名函数 | 很难知道到底会执行什么代码，因为在编译器还未确定 |

# 总结

1. 写一个比较时，把改变的值放在左边，把稳定的值放在右边
2. 可以重新排列 if else 代码块，优先处理正确的、简单的逻辑。
有时这些准则会冲突，当不冲突时，遵循这些经验法则。
3. 像三目运算符、do while循环经常会导致代码可读性变差。最好不要使用它们，
因为总是有更整洁的方式。
4. 嵌套的代码块需要花一些时间去理解。每层新的嵌套都会给读者“思维栈” push 
一条数据。应该让它们变得“线性”，来避免深层嵌套。
5. 提早返回可以让代码更整洁。

# 拆分又臭又长的表达式

引入“解释变量”代替较长的子表达式，有三个好处：

1. 它把巨大的表达式拆分成一个小段
2. 通过简单的名字来描述一个子表达式，让代码文档化
3. 它帮助读者识别代码中重要的概念

**用德摩根定理来操作逻辑表达式**

# 变量和可读性

1. 减少变量，通过立刻处理结果来消除中间变量。
2. 减少变量作用域，作用域越小越好。
3. 变量只写一次最好，只设置一次的变量会让代码变得更容易理解。

[biezhi/write-readable-code](https://github.com/biezhi/write-readable-code)