## Container

Laravel 的容器对象实现了[「PSR-11」](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-11-container.md)标准定义的相关规范。

## 基础

Laravel 主要提供三大方面的功能

- 创建对象
- 依赖注入（DI）
- 管理对象


###  创建对象

#### binding

要让容器创建对象，必须先告诉容器如何创建对象，这个过程在容器里面叫做 binding ，对应在容器的 API 叫做 `bind($abstract, $concrete = null, $shared = false)`。

参数介绍 

- `$abstract` binding 的名称，在创建对象的时候需要用到这个名称。
- `$concrete` 可选值，也可以传入一个 有返回值的 `Closure`类型,如果这个参数是 `null`，那么 `$abstract` 必须是一个完整的类名称。
- `$shared` 是否共享，如果为 `true` 容器只会创建一个该对象的实例。


例如：

```

class Animal
{

}


$container = \Illuminate\Container\Container::getInstance();

// 第一种
$container->bind(Animal::class); //在make该binding的对象的时候容器会创建Animal对象实例

// 定义创建对象闭包
$container->bind('OtherAnimal', function () {
    return 'Other Animal';
});// 在make('OtherAnimal')的时候容器会返回一个`Other Animal`字符串也就是闭包的返回值 

```



#### make 

binding 只是告诉容器如何创建对象，但是并没有真正的创建对象。如果需要创建获取 binding 的对象必须调用 `make($abstract, array $parameters = [])` 方法让容器帮我们创建对象。

参数介绍

- `$abstract`  binding 的名称
- `$parameters` 创建对象的参数。被创建对象构造函数的参数必须是明确的或者是可构建的。

例如下面的代码会报错，因为在 binding 的时候没有制定参数如果创建，在 make 的时候也没有生成，并且被创建对象的构造函数的参数既没有制定默认值，也不是 Class类型。

```
class Animal
{
    protected $name;

    public function __construct($name)
    {
        $this->name = $name;
    }
}

$container = \Illuminate\Container\Container::getInstance();

// 第一种用法：注册
$container->bind(Animal::class);// 在容器里面注册一个 binding

$container->make(Animal::class);

```

上面的问题有两种解决方式

在 binding 的时候就制定 `$name`的值

```
$container->bind(Animal::class,function(){
    return new Arimal("Tom");
});
```

另外一种方式就是在make的时候制定 

```
$container->make(Animal::class,,array('name'=>'Tom'));
```

### 依赖注入（DI)

关于 DI 的介绍，可以看[Wiki](https://zh.wikipedia.org/wiki/%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5)。

在 Laravel 里面如果被创建的类的构造函数的参数是依赖的是一个对象的话，那么它会去自动创建这个对象传入。

例如 

```

class UserProxy
{
  
}

class UserController
{
    protected $proxy;

    public function __construct(UserProxy $proxy)
    {
        $this->proxy = $proxy;
    }


}

$container = \Illuminate\Container\Container::getInstance();

$container->bind(UserController::class);
$container->make(UserController::class);// 会自动创建一个UserProxy的对象传给构造方法


```

### 管理对象

通常情况下在在一个生命周期内只会存在一个全局容器对象，我们可以通过这个全局的容器对象来创建，共享，销毁对象。

通常我们会把一个复杂的业务逻辑分拆为一个个功能单一的方法，那么我们如何做到数据共享呢？有两种方法，一种是通过定义方法的参数
进行传递，但是如果方法的层次特别深的话，这种做法会显得特别不方便，另外一种就是把数据保存在容器对象里面，需要的时候直接去容器
对象里面去取。在 Laravel 使用的非常频繁

例如

```
 protected function bindPathsInContainer()
    {
        $this->instance('path', $this->path());
        $this->instance('path.base', $this->basePath());
        $this->instance('path.lang', $this->langPath());
        $this->instance('path.config', $this->configPath());
        $this->instance('path.public', $this->publicPath());
        $this->instance('path.storage', $this->storagePath());
        $this->instance('path.database', $this->databasePath());
        $this->instance('path.resources', $this->resourcePath());
        $this->instance('path.bootstrap', $this->bootstrapPath());
    }
    
    
    //在其他业务逻辑中直接使用
    
    if (! function_exists('config_path')) {
        /**
         * Get the configuration path.
         *
         * @param  string  $path
         * @return string
         */
        function config_path($path = '')
        {
            return app()->make('path.config').($path ? DIRECTORY_SEPARATOR.$path : $path);
        }
    }

```

##  bind 与 resolve 相关细节


### bind 的细节

` public function bind($abstract, $concrete = null, $shared = false) `

三种用法

 - bind($abstract) `$abstract`必须是一个可创建的类的名称
 - bind($abstract,$class) `$abstract`是一个 `string`,`class` 是一个类名称
 - bind($abstract,$closure)  `$abstract`是一个 `string`, `$closure` 是一个闭包类型


具体的实现细节

- 删除 `$abstract`  对应的实例对象，如果存在的话
- 如果 `$concrete` 为空，那么 `$abstract` 赋值给 `$abstract`。也就是上面的第一种用法
- 保存闭包和共享标识
- rebound 的处理



### resolve 的细节

resolve 用来创建 bind 的类型。它有几个包装方法和别名方法

- `public function get($id)`
- `public function make($abstract, array $parameters = [])`
- `public function makeWith($abstract, array $parameters = [])`
- `public function offsetGet($key)` 数组的形式访问的支持

这些方法的底层方法都是 ` protected function resolve($abstract, $parameters = [])`


resolve 具体的实现细节

- 获取别名
- 标记创建对象是否明确的指定了参数或者上下文环境设置（如果是`true`则被创建的对象不能是共享对象）
- 获取对象创建的闭包对象
- 如果是闭包或者类，直接调用build方法创建，如果是嵌套方法就递归创建


`public function build($concrete)` 方法的实现细节

- 如果 `$concrete` 是闭包直接执行返回闭包的结果
- 创建 `ReflectionClass` 对象，反射的详细介绍看[Wiki](https://zh.wikipedia.org/wiki/%E5%8F%8D%E5%B0%84_(%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6))和[PHP反射](http://php.net/manual/zh/book.reflection.php)
- 判断给定的类型是否能创建对象
- 获取构造函数对象，如果没有构造函数则直接 `new` 给定的类型
- 获取构造函数的参数对象
- 获取／创建构造函数参数的值
- 创建返回对象 


## 其他

### 扩展对象



在 bind 后，可以调用 `public function extend($abstract, Closure $closure)` 设置对象的扩展规则。

```

class Animal
{

    public $extend = false;

}


$container = new \Illuminate\Container\Container();
$container->bind(Animal::class);
$container->extend(Animal::class, function (Animal $animal, \Illuminate\Container\Container $container) {
    $animal->extend = true;
});


$animal = $container->make(Animal::class);// extend 为 true



```

### hook机制

分别是

- `public function resolving($abstract, Closure $callback = null)`
- `public function afterResolving($abstract, Closure $callback = null)`

注意： hook和扩展机制设置的回掉函数最好把传入的对象返回



## 如何阅读源码

- 干净的container源码我已经准备好了，直接 clone 下来后执行 `composer install`
- 要理解源码逻辑的前提是要会用，关于怎么用，看测试用例的代码比文档更直接。所有的测试用例都在 tests 目录下
- 运行测试用例，改代码，运行测试用例，改代码...........

















