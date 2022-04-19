# PHP基础语法

首先我们来复习一下PHP中的基础语法。

## 变量

PHP有四种基本类型：bool, integer, float(double), string；两种复合类型：array, object；以及NULL。

除此之外还有资源类型，表示对外部资源的引用，比如打开的文件的描述符、连接数据库的管道等，在此不做讨论。

### 类型判断

基本类型都可以使用`is_*()`函数判断。

```php
is_bool(False);
// TRUE, 顺便一提 true 和 false 不区分大小写

is_object(0);
// false

$a = 1.;
is_float($a);
// true
is_double($a);
// true
// 没错float和double是一样的

is_int(NULL);
// false
is_null(NULL);
// NULL 类型唯一可能的值就是 NULL
```

### 数组（array）类型

PHP中的数组是PHP编程中最常用的数据类型，形式十分宽松。
它可以是类似C语言的仅包含若干同种类型数据的数组，也可以是JavaScript里的键值对对象，甚至可以是一个二叉树。

宽松的形式为开发带来了极大的遍历，但也埋下了重大的隐患。例如，在取变量时不会检查索引（index）是否存在。在强类型编译语言比如golang中，在编译阶段编译器就会强制你检查索引的存在性，但PHP不会。

你可能会说，只要我肉眼保证索引存在不就好了？但如果你在索引中写了一个typo，可能就要debug十分钟了。如何尽量避免这个问题，我会在PHP的编程小妙招里展开说明。

### 队列和栈

和JavaScript一样，PHP提供了过于丰富的关于数组操作的函数。其中有两套可以用来实现队列和栈。

```php
// 队列

$list = [/* items */];

// 出队头
while ($item = array_shift($list)) {
    // 处理 $item

    // 入队尾
    array_push($list, $anotherItem);
}
```

```php
$stack = [/* items */];

// 出栈
while ($item = array_pop($stack)) {
    // 处理 $item

    // 入栈
    array_push($stack, $anotherItem);
}
```

## 分支与条件

### If 语句

```php
if ($condition) {
    // condition satisfied
} else {
    // or not
}
```

### Switch 语句

实际上，几乎任何编程语言中都应该尽量避免使用Switch语句，因为可读性很差且后期维护困难。一般情况下，我们可以认为Switch语句和if-else语句是等价的。但这不是说后者比前者更好，实际上两者的可读性和不可维护性都差不多。一个两三百行的分支语句总是能让后继的开发者望而却步。

一个适合使用Switch语句的情况是，C语言或者shell脚本中获取指令参数的时候，通常结合while循环一起使用。

```shell
while getopts "ab:" arg; do
    case $arg in
    a) flag_a=1 ;;
    b) flag_b=$OPTARG ;;
    esac
done
```

## 循环

### For循环

传统的for循环，例如C语言中的`for (int i = 0; i < 10; i++)`，应该尽量舍弃。在现代的，高级的编程语言里，如下所示的遍历循环更加常见。

```php
foreach ($iterable as $item) {
    // 处理 $item
}

foreach ($iterable as $key => $value) {
    // 处理 $key 和 $value
}
```

## 函数

PHP 7开始，PHP中添加了许多类似JavaScript，或者说其他现代语言的特性，比如匿名函数和闭包。在PHP中，匿名函数和闭包没有区别。

```php
function a () {
    echo "something";
}

// 普通的函数调用
a();

$a = function () {
    echo "something";
}

// 闭包的函数调用
$a();

// 使用了闭包的二叉树中序遍历函数
function LDR($node, $callback) {
    if (is_null($node))
        return;
    LDR($node->left, $callback);
    $callback($node);
    LDR($node->right, $callback);
}

// 在闭包中使用use引入外部变量
$prompt = "Number is: ";
LDR($root, function ($node) use ($prompt) {
    echo $prompt . $node->number;
});
```
