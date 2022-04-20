# PHP的编程小妙招

## 弱类型，但是强约束

考虑如下一种情况：进行数据库查询时我们总需要一个数组存储查询语句和语句中绑定的变量。该数组形式如下所示：

```php
[
    'string' => 'SELECT * FROM students WHERE grade = ? and score > ?',
    'params' => ["2022级", 60],
]
```

但是，在编程时，很不幸地，你把 `$query['string']` 写成了 `$query['srting']`，也就是有一处笔误（typo）。然后整个程序崩溃了。

在慌慌张张地修改代码之前，不妨坐下来冷静思考一下，出现这种情况，究竟是我们的问题，还是PHP的问题？在C语言和Golang中，我们访问一个结构体中的变量，如果不存在，编译器就会直接报错，这是很显然的。在Golang中，访问Map索引而没有实现判断其存在性则编译器也会报错。

所以，出现问题都是因为PHP是一种过分自由的弱类型语言：PHP中的数组甚至可以具有树形结构。为了避免刚刚描述的typo等隐含的错误，最好的做法就是减少数组的使用，为了尽可能地利用好PHP的Linter，有必要使用一个类取代上述数组。而且，有了类和其中属性的辅助，就算对你的项目一无所知的人来阅读你的代码，也不会被从不在任何地方声明数组的索引搞得晕头转向了。

```php
/**
 * @property string $string
 * @property array $params
 */
class DBQuery {
    public $string;

    public $params;

    public function __construct($string = "", $params = [])
    {
        $this->string = $string;
        $this->params = $params;
    }
}
```

## 尽可能地隐式类型转换

强制类型转换应当避免。当我们需要转换类型的时候，可以选用相应的魔术方法或者实现相应接口以实现隐式类型转换。

### 万物皆可字符串

任何类都可以使用 `__toString()` 魔术方法定义如何转换成字符串。

### 万物皆可序列化

任何类都可以实现 `JsonSerializable` 接口并实现 `jsonSerialize()` 方法定义，当该类作为 `json_encode()` 函数的参数时如何转换。

## 一些设计模式的实现

### 流式API

在JavaScript中，你或许写过或者见过类似的编程实现：

```javascript
"Hello World".toLowerCase()
    .replace('world', 'computer')
    .split(' ')
```

这就是所谓的流式API，或者说瀑布式调用。观察发现，上文的 `toLowerCase()`，`replace()`，`split()` 等方法都是`string` 类型的函数，且返回值也是 `string` 类型。因此，在PHP中，定义一些返回值是 `$this` 的函数是实现流式API的一种办法。

```php
class Response
{
    private $statusCode;

    private $headers = [];

    // ...

    /**
     * 添加一个返回头。
     *
     * @param string $key
     * @param string $value
     *
     * @return lib\Response
     */
    public function addHeader($key, $value)
    {
        // 先全小写再首字母大写
        $key = ucfirst(strtolower($key));
        // 把value强制转换为字符串
        $this->headers[$key] = (string) $value;
        // 返回自身指针从而实现流式API
        return $this;
    }

    /**
     * 设置 HTTP 状态码。
     *
     * @param int $statusCode
     *
     * @return lib\Response
     */
    public function setStatusCode($statusCode)
    {
        $this->statusCode = $statusCode;
        return $this;
    }

    // ...
}

// 调用
$response = new Response;
$response
    ->addHeader('location', 'https://www.bilibili.com/video/BV14y4y1U7RK/')
    ->setStatusCode(302);
```

这种实现的好处是各个方法是可以交换顺序的，比如上文所展示的 `addHeader()` 和 `setStatusCode()`。

但是也有弊端：当我们需要规定调用流的顺序的时候，我们就不希望用户任意交换方法顺序，以免发生错误（由状态机决定）。此时我们就可以控制方法返回的内容来控制API调用顺序，也就是说，返回值不都是 `$this` 指针，有时返回新类。

```php
class DB
{
    // 指定数据表
    public static function table($table)
    {
        return new QueryBuilder($table);
    }
}

class QueryBuilder
{
    // 记录一个where子句
    public function where($column, $operator, $value)
    {
        // ...
        return $this;
    }

    // 返回所有查询结果
    public function get()
    {
        $res = new QueryResult;
        // 记录查询结果到 $res
        return $res;
    }
}

// 调用
DB::table('students')
    ->where('grade', '=', '2022级')
    ->where('score', '>', 60)
    ->get();
```

在这个例子中，两个 `where` 是可以交换次序的，但是必须遵循 `table`->`where`->`get` 的调用次序。

### 单例模式

> 一个类<!-- 吻 -->就够了，第二个就多余了。

有一些特殊的类，我们希望它们在软件的生命周期中，有且只有一个实例，以减少频繁实例化带来的开销，比如存储全局配置的 `Config`。这种设计模式就叫单例模式。下面给出单例模式的一种实现：`Singleton` 类。继承了 `Singleton` 的子类，如果仅使用 `getInstance()` 获取实例，就可以保证在整个软件的生命周期中，该子类有且仅有一个实例。

```php
/**
 * 单例模式的一种PHP实现。
 *
 * 继承此类者可以通过self::getInstance()获取自身的一个单例。
 */
class Singleton
{
    /**
     * @var self 存储自身的唯一的实例（单例）
     */
    protected static $instance;

    /**
     * 禁用clone魔术方法。
     *
     * @return void
     */
    protected function __clone()
    {
        // Disable cloning
    }

    /**
     * 获取自身单例。
     *
     * 注意new关键字后面是static而不是self，否则实例化的不是子类而永远是Singleton。
     *
     * @return static
     */
    public static function getInstance()
    {
        /* 若self::$instance尚未存储自身的一个单例，则实例化并存储。 */
        if (!self::$instance instanceof static)
            self::$instance = new static(...func_get_args());

        /* 同时返回该实例 */
        return self::$instance;
    }
}
```
