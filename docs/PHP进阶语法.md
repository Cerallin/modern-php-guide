# PHP进阶语法

## 类

### 基于自动加载的编程范式

不论何种编程语言，稍大的项目总需要多个文件组合到一起。文件之间的依赖关系总是令人头疼。在C语言中，接口与实现分离，所有的依赖都使用头文件管理。在JavaScript中，模块间通过import和require访问互相的暴露出来的接口。PHP也通过`require`关键字加载对应文件。不过，和nodejs只能使用`require`的历史遗留问题一样，PHP的require的调用方法令人疑惑。

> PHP的require和Python的import、Perl的import类似，当我们引入一个文件或者模块的时候，不会想到这个语句有返回值。但PHP中却不是这样。

在PHP中可以使用`spl_autoload_register()`方法注册自动加载函数。例如：

```php
spl_autoload_register(function ($className) {
    $filePath = str_replace(array('\\', '_'), '/', $className) . ".php";

    if (is_readable($filePath)) {
        return require $filePath;
    }
});
```

当我在`index.php`中写下一行`use lib\Abstracts\Singleton;`，PHP会自动加载`lib/Abstracts/Singleton.php`。

在这种编程范式下，虽然无法做到接口与实现分离，但应该做到每一个文件有且仅定义了一个类，或者说，只有与文件名相同的类可以暴露给其他文件。

### 继承

PHP中的类只能至多继承一个类。下图展示的`Config`继承了`Singleton`。

可以注意到，类的属性和方法有三种访问范围：`public`、`protected`和`private`。
其中，`public`使属性和方法暴露在外，`protected`使得子类可以访问父类的方法（因为PHP的继承关系很简单所以没有其他情况），`private`的属性和方法只能被自己访问。

```php
class Singleton
{
    protected static $instance;

    public static function getInstance()
    {
        // ...
    }
}

class Config extends Singleton
{
    private $config;
}
```

### 多态

为了弥补不能多继承的缺陷，或者说一定程度上解决多继承的痛点，PHP中使用trait实现了多态。一个类可以使用多个trait，多个类也可以使用同一个trait。因此，trait的使用场景有两个：在不同类之间实现代码复用；把一个大类拆成几个traits提高可读性。

值得注意的是，trait中的方法中的self和this指向各个使用了trait的类，而继承关系中父类方法中的self和this指向自身，而不是子类。

下面给出一个使用trait的例子：

```php
trait crud {
    public function insert($sql) { $this->exec($sql); }
    public function delete($sql) { $this->exec($sql); }
    public function update($sql) { $this->exec($sql); }
    public function select($sql) { $this->exec($sql); }
}

trait table {
    public function createTable() { /* create table */ }
    public function dropTable() { /* drop table */ }
}

class DB {
    use crub, table;

    private function exec($sql) {
        // Execute sql
    }
}
```

### 接口

接口只定义方法和属性，但并不实现方法的具体内容，因此接口中的方法没有函数体。

下面给出一个例子：`Request`通过`JsonSerializable`判断`Counter`是否能序列化成JSON格式。

```php
interface JsonSerializable {
    public function jsonSerialize();
}

class Request
{
    private $content;

    public function __construct($content)
    {
        $this->content = $content;
    }

    public function dispatch() {
        if ($this->content instanceof JsonSerializable) {
            $this->content = json_encode($this->content);
        }
        // 继续其他操作
    }
}

class Counter extends Singleton implements JsonSerializable
{
    /**
     * 当作为json_encode参数时的序列化处理。
     *
     * @return array
     */
    public function jsonSerialize() {
        return [
            'site_pv' => 10,
            // ...
        ];
    }
}
```

### __call() & __callStatic()

`__call()`和`__callStatic()`是两个魔术方法，该方法在调用的动态/静态方法不存在时会自动调用。他们可以避免当调用的方法不存在时产生错误，而导致程序意外中止。不过这里将要介绍的不是使用这两个方法实现异常处理，而是利用`__call()`实现一系列相似的接口的能力。

例如，我们有一个`Log`类，专门用来记录各种日志。每一条日志有三种日志等级：`info`，`warn`，`error`。你或许想到了一种实现：

```php
class Log
{
    public static function writeLog($level, $message) {
        // ... 准备工作 ...
        // 写入日志
        file_put_contents($logFile, $level . $message . "\n", FILE_APPEND | LOCK_EX);
    }
}

// 调用
Log::writeLog('ERROR', 'Something wrong.');
```

这种实现定义一个函数就实现了记录不同等级日志的功能，缺点是函数调用时`writeLog`显得有些冗余。如果我们可以让`error`作为函数名，这样这个函数就仅有`$message`一个参数，简洁又不失可读性。像这样：

```php
class Log
{
    private static function writeLog($level, $message) {
        // ... 准备工作 ...
        // 写入日志
        file_put_contents($logFile, $level . $message . "\n", FILE_APPEND | LOCK_EX);
    }

    public function info($message) {
        return self::writeLog('INFO', $message);
    }

    public function warn($message) {
        return self::writeLog('WARN', $message);
    }

    public function error($message) {
        return self::writeLog('ERROR', $message);
    }
}

// 调用
Log::error('Something wrong.');
```

如你所见，方法调用显得好看多了。但是这种实现的缺点也很明显：我们实现了大量功能相似的方法，一点也不简介。下面介绍一种使用`__callStatic()`的实现，兼顾了上述两种实现的优点。

```php
/**
 * 记录日志。
 *
 * @method static void info(string $message)
 * @method static void warn(string $message)
 * @method static void error(string $message)
 *
 */
class Log
{
    const logLevels = ['info', 'warn', 'error'];

    private static function writeLog($level, $message) {
        // ... 准备工作 ...
        // 写入日志
        file_put_contents($logFile, $level . $message . "\n", FILE_APPEND | LOCK_EX);
    }

    /**
     * @param string $name      函数名
     * @param array $arguments  函数参数数组
     */
    public static function __callStatic($name, $arguments)
    {
        $level = strtolower($name);
        if (in_array($level, self::logLevels, true)) {
            // 三个点，意思是把 $arguments 展开作为函数参数
            self::writeLog($level, ...$arguments);
        }
    }
}

// 调用
// 因为使用了strtolower处理函数名，所以都是一样的
Log::error('Something wrong.');
Log::Error('Something wrong.');
Log::ERROR('Something wrong.');
```

## 异常处理

PHP中，错误出现的情况是由错误的语法，服务器环境导致，使解释器无法通过检查，甚至无法运行。warning、notice都是错误，只是他们的级别不同而已，并且错误不能被try-catch捕获。异常是指程序在运行中出现不符合预期的情况，属于逻辑和业务流程的错误，而不是编译或者语法上的错误。

比如说，忘记写分号导致脚本无法运行时，PHP就会报告一个错误；而异常应当是我们自己手动定义的，比如数据库连接失败的时候可以抛出一个`DBConnectException`。

PHP中使用try-catch处理异常，和C++、Java、JavaScript等其他语言一样。只不过，PHP并不会强制你处理潜在的所有异常。暨时，PHP的异常和错误统统会被记录到错误日志里。

一种常见的异常处理流程如下所示：

```php
try {
    /* ... */
    // If connect to database failed
    if (!connect_to_db())
        throw new DBConnectException;
    /* ... */
} catch (DBConnectException $e) {
    // 异常套异常
    throw new DBQueryException("Connect to database failed.", 500, $e);
}
```
