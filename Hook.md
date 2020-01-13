## Hook

> [行为和钩子]: https://www.kancloud.cn/manual/thinkphp5_1/354129

##### 一、先导入后监听执行

实例化

```php
public function __construct(App $app)
{
    $this->app = $app;
}
```

import() 批量导入插件

```php
/**
 * 批量导入插件
 * @access public
 * @param  array     $tags 插件信息
 * @param  bool      $recursive 是否递归合并
 * @return void
 */
public function import(array $tags, $recursive = true)
{
    if ($recursive) {
        foreach ($tags as $tag => $behavior) {
            $this->add($tag, $behavior);
        }
    } else {
        // 右边的数组元素附加到左边的数组后面 两个数组中都有的键名，则只用左边数组中的，右边的被忽略
        $this->tags = $tags + $this->tags;
    }
}
```

add() 动态添加行为扩展到某个标签

```php
/**
 * 动态添加行为扩展到某个标签
 * @access public
 * @param  string    $tag 标签名称
 * @param  mixed     $behavior 行为名称
 * @param  bool      $first 是否放到开头执行
 * @return void
 */
public function add($tag, $behavior, $first = false)
{
    isset($this->tags[$tag]) || $this->tags[$tag] = [];

    if (is_array($behavior) && !is_callable($behavior)) {
        if (!array_key_exists('_overlay', $behavior)) {
            // 合并
            $this->tags[$tag] = array_merge($this->tags[$tag], $behavior);
        } else {
            // 覆盖
            unset($behavior['_overlay']);
            $this->tags[$tag] = $behavior;
        }
    } elseif ($first) {
        array_unshift($this->tags[$tag], $behavior);
    } else {
        $this->tags[$tag][] = $behavior;
    }
}
```

listen() 监听标签的行为

```php
/**
 * 监听标签的行为 
 * @access public
 * @param  string $tag    标签名称
 * @param  mixed  $params 传入参数
 * @param  bool   $once   只获取一个有效返回值
 * @return mixed
 */
public function listen($tag, $params = null, $once = false)
{
    $results = [];
    $tags    = $this->get($tag);
	
    foreach ($tags as $key => $name) {
        $results[$key] = $this->execTag($name, $tag, $params);
		// 如果返回false则当前标签位的后续行为将不会执行
        if (false === $results[$key] || (!is_null($results[$key]) && $once)) {
            break;
        }
    }

    return $once ? end($results) : $results;
}
```

get() 获取插件信息

```php
/**
 * 获取插件信息
 * @access public
 * @param  string $tag 插件位置 留空获取全部
 * @return array
 */
public function get($tag = '')
{
    if (empty($tag)) {
        //获取全部的插件信息
        return $this->tags;
    }

    return array_key_exists($tag, $this->tags) ? $this->tags[$tag] : [];
}
```

execTag() 执行某个标签的行为

默认调用对应标签位的方法，如app_init标签位对应appInit()方法。若没有对应的方法则调用run()方法。

```php
/**
 * 执行某个标签的行为 
 * @access protected
 * @param  mixed     $class  要执行的行为
 * @param  string    $tag    方法名（标签名）
 * @param  mixed     $params 参数
 * @return mixed
 */
protected function execTag($class, $tag = '', $params = null)
{
    // type 0 将Java风格转换为C的风格(大写字母前加_并全转为小写) 
    // 1 将C风格转换为Java的风格(去除_并将紧跟着的后面的字母转大写，首字母可大写或小写)
    // 比如 type=1 app_init 转化为 appInit
    $method = Loader::parseName($tag, 1, false);

    if ($class instanceof \Closure) {
        $call  = $class;
        $class = 'Closure';
    } elseif (is_array($class) || strpos($class, '::')) {
        $call = $class;
    } else {
        $obj = Container::get($class);

        if (!is_callable([$obj, $method])) {
            $method = self::$portal;
        }

        $call  = [$class, $method];
        $class = $class . '->' . $method;
    }

    $result = $this->app->invoke($call, [$params]);

    return $result;
}
```

##### 二、不绑定直接执行

默认执行的是run方法

```php
/**
 * 执行行为
 * @access public
 * @param  mixed     $class  行为
 * @param  mixed     $params 参数
 * @return mixed
 */
public function exec($class, $params = null)
{
    if ($class instanceof \Closure || is_array($class)) {
        // 传入数组则可选择的调用方法
        $method = $class;
    } else {
        // 默认调用run方法
        if (isset($this->bind[$class])) {
            $class = $this->bind[$class];
        }
        $method = [$class, self::$portal];
    }

    return $this->app->invoke($method, [$params]);
}
```

##### 三、指定入口方法名称

```php
/**
 * 指定入口方法名称 默认是run
 * @access public
 * @param  string  $name     方法名
 * @return $this
 */
public function portal($name)
{
    self::$portal = $name;
    return $this;
}
```

