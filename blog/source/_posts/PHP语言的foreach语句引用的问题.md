---
title: PHP语言的foreach语句引用的问题
date: 2020-08-02 08:39:48
tags:
- php
- foreach
- 引用
categories:
- tech
description: PHP编码问题
---

### 问题
请看如下代码：

```php
<?php

$arr = [ 'a', 'b', 'c', 'd' ];

foreach($arr as &$v) {
    echo $v . " ";
}

foreach($arr as $v) {
    echo $v . " ";
}

```

执行命令获取得到的输出:

```bash
a b c d 
a b c c
```

### 后果
如果大家在 `foreach` 循环中滥用引用但是没有进行 `用完后的销毁操作` ，或许会导致后面的数据发生错误和混乱，代码难以调试
所以建议大家采用如下的解决方案：

```php
<?php
$arr = [ 'a', 'b', 'c', 'd' ];

foreach($arr as &$v) {
    echo $v . " ";
}

unset($v);

foreach($arr as $v) {
    echo $v . " ";
}
```

或者不使用引用：

```php
<?php
$arr = [ 'a', 'b', 'c', 'd' ];

foreach($arr as $v) {
    echo $v . " ";
}

foreach($arr as $v) {
    echo $v . " ";
}
```

为了避免使用引用出问题或者后续的冗余代码，可以使用包裹函数：

```php
<?php

function loopArrayReference(array $input = null, Closure $callback = null )
{
    foreach($input as $index => &$val) {
        $callback($input, $index, $val);
    }
    unset($val);
}

```


### 解析
这里为什么会发生这种情况，第二个 `foreach` 循环的时候，为什么拿不到最后的一个值？ 原因如下：
PHP处理引用的时候，会在全局范围定义一个变量( 假设变量为 $t )后续的引用都是操作这个变量，因此我们就知道了，当第一次循环结束的时候，此时
`$t` 的值保存的是 `$arr` 的最后一个值 `'d'`， 此时 `$t` 是引用，相当于 `C语言` 里面的 `指针`
当进行到第二次循环处理的时候:

处理第一个元素的时候
```php
$t = 'a'
```

处理第二个元素的时候
```php
$t = 'b'
```

处理第三个元素的时候
```php
$t = 'c'
```

处理第四个元素的时候,此时保存的是前一个元素的值，也就是第三个元素的值
```php
$t = 'c'
```

所以得到输出:
```bash
a b c d
a b c c
```

### 后记
如果一定要使用引用，一定要记得进行使用后的销毁操作,这个也是很多只会PHP业务层代码的程序员容易忘记的事情。


