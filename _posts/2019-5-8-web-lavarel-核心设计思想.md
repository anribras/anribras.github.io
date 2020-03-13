---
layout: post
title:
modified:
categories: Tech
tags: [lavarel]
comments: true
---

<!-- TOC -->

- [服务容器](#服务容器)
  - [控制反转(IOC)与依赖注入(DI)](#控制反转IOC与依赖注入DI)
- [服务提供者](#服务提供者)
- [Facade](#Facade)
- [Contract](#Contract)

<!-- /TOC -->

## 服务容器

<https://laravelacademy.org/post/769.html>

<https://laravelacademy.org/post/9534.html>

### 控制反转(IOC)与依赖注入(DI)

讲的都是1个概念.

**Ioc:** 主对象依赖的某些对象原来在对象内部产生.现在把这个动作放到外面,比如Ioc容器,,IoC容器控制了对象生命周期,需要对象时，直接找容器，容器实例负责查找及创建依赖对象(也可以直接绑定已有对象的实例).

**DI** 由容器动态的将某个依赖关系注入到组件之中。依赖注入的目的并非为软件系统带来更多功能，而是为了提升组件重用的频率，并为系统搭建一个灵活、可扩展的平台, 这个平台就是基于IOC容器.

注入的原理是`反射`,根据类名创建实例,或者是`Clouse`,即闭包函数创建实例.

所以容器需要知道它负责的对象如何创建,这就是bind(绑定创建实例的闭包函数)和instance(绑定已有实例)的作用.

如何保存绑定大量对象?,就用array数组搞定:

```sh
#bind绑定后的binding数组:
[
    'A\1': 返回new的闭包函数 or 具体实现类 or 类名
    'A\2': 返回new的闭包函数 or 具体实现类 or 类名
]
#如果是`singleton`方法，实例存在instances数组,供下次使用.
[
    'A\1': A\1实例,
    'A\2': A\2实例
]
```

有了绑定关系，剩下的就是如何解析实例了,函数`build`,`make`,`resolve`等.

```php
//container.php
$concrete = $this->getConcrete($abstract);
// We're ready to instantiate an instance of the concrete type registered for
// the binding. This will instantiate the types, as well as resolve any of
// its "nested" dependencies recursively until all have gotten resolved.
if ($this->isBuildable($concrete, $abstract)) {
    //build
    $object = $this->build($concrete);
} else {
    $object = $this->make($concrete);
}
```

最终的核心动作在`build`里, 如果concrete是closure,则调用产生，否则根据类名`反射`产生该实例

反射里应该是一颗递归树,因为class A的constructor参数里, 可能依赖B,B依赖C,D...

```php
    public function build($concrete)
    {
        //...
        if ($concrete instanceof Closure) {
            return $concrete($this, $this->getLastParameterOverride());
        }
        try {
                $reflector = new ReflectionClass($concrete);
            } catch (ReflectionException $e) {
                throw new BindingResolutionException("Target class [$concrete] does not exist.", 0, $e);
        }
        //...
    }

```

bind的基本原理就这样了,而singleton是单例模式,生成的单例存储会先存储到instance数组.

## 服务提供者

所有类都这样手动bind,当然麻烦,于是就有`service provider`.

先register Provider, 它会调用每个Provider实例里的register和boot，完成具体的实例bind.

```php
public function register($provider, $force = false)
    {
        if (($registered = $this->getProvider($provider)) && ! $force) {
            return $registered;
        }

        // If the given "provider" is a string, we will resolve it, passing in the
        // application instance automatically for the developer. This is simply
        // a more convenient way of specifying your service provider classes.
        if (is_string($provider)) {
            $provider = $this->resolveProvider($provider);
        }

        $provider->register();

        // If there are bindings / singletons set as properties on the provider we
        // will spin through them and register them with the application, which
        // serves as a convenience layer while registering a lot of bindings.
        if (property_exists($provider, 'bindings')) {
            foreach ($provider->bindings as $key => $value) {
                $this->bind($key, $value);
            }
        }

        if (property_exists($provider, 'singletons')) {
            foreach ($provider->singletons as $key => $value) {
                $this->singleton($key, $value);
            }
        }

        $this->markAsRegistered($provider);

        // If the application has already booted, we will call this boot method on
        // the provider class so it has an opportunity to do its boot logic and
        // will be ready for any usage by this developer's application logic.
        if ($this->isBooted()) {
            $this->bootProvider($provider);
        }

        return $provider;
    }
```

## Facade

<https://sergeyzhuk.me/2016/05/27/laravel-facades/>

```sh
 Laravel facades serve as "static proxies" to underlying classes in the service container, providing the benefit of a terse, expressive syntax while maintaining more testability and flexibility than traditional static methods.
```

Laravel 门面为 Laravel 服务的使用提供了便捷方式 ,不再需要从服务容器中类型提示和契约解析即可直接通过静态门面调用.

可以理解为操作复杂对象的简化方法,就是外观模式里的外观类的方法.

对类实例的方法的封装，使用起来更方便.

下面三种方法对某个实例方法来讲没有区别:

```php
//facade
SomeService::someMethod();
// and
app()->make('some.service')->someMethod();
// or
App::make('some.service')->someMethod();
```

核心是php的魔术方法`__callStacic`,所有调用Facade里静态方法，都会进入这个函数，进行真正的(代理)调用.

```php
    //Facde接口
    public static function __callStatic($method, $args)
    {
        $instance = static::getFacadeRoot();

        if (! $instance) {
            throw new RuntimeException('A facade root has not been set.');
        }

        return $instance->$method(...$args);
    }
    public static function getFacadeRoot()
    {
        return static::resolveFacadeInstance(static::getFacadeAccessor());
    }
    protected static function resolveFacadeInstance($name)
    {
        if (is_object($name)) {
            return $name;
        }

        if (isset(static::$resolvedInstance[$name])) {
            return static::$resolvedInstance[$name];
        }

        //核心就在这,$app就是上面的ioc容器实例
        if (static::$app) {
            return static::$resolvedInstance[$name] = static::$app[$name];
        }

        //真正的实例来源
        public static function swap($instance)
        {
            static::$resolvedInstance[static::getFacadeAccessor()] = $instance;

            if (isset(static::$app)) {
                //注册到Ioc容器
                static::$app->instance(static::getFacadeAccessor(), $instance);
            }
        }
    }
```

实现Facade接口的类，要实现`getFacadeAccessor`和`resolveFacadeInstance`,关联要真正的实例.

Date为例子,其真正的实例是`DateFactory`.

```php
class Date extends Facade
{
    const DEFAULT_FACADE = DateFactory::class;

    /**
     * Get the registered name of the component.
     *
     * @return string
     *
     * @throws \RuntimeException
     */
    protected static function getFacadeAccessor()
    {
        return 'date';
    }

    /**
     * Resolve the facade root instance from the container.
     *
     * @param  string  $name
     * @return mixed
     */
    protected static function resolveFacadeInstance($name)
    {
        if (! isset(static::$resolvedInstance[$name]) && ! isset(static::$app, static::$app[$name])) {
            $class = static::DEFAULT_FACADE;
            //swap执行真正的instance绑定到ioc容器
            static::swap(new $class);
        }

        return parent::resolveFacadeInstance($name);
    }
}
```

想用Date时，可以:

```php
//直接用门面静态方法
 \Illuminate\Support\Facades\Date::createXXX()
```



但Laravel 的门面作为服务容器中底层类的「静态代理」，相比于传统静态方法，在维护时能够提供更加易于测试、更加灵活、简明优雅的语法。

如果还是嫌弃用之前`use xxxx\Facades\xx`太麻烦，config.app里配置个alias:

```php
?php
return [
    //...
    'aliases' => [
        'App'     => Illuminate\Support\Facades\App::class,
        'Artisan' => Illuminate\Support\Facades\Artisan::class,
        'Auth'    => Illuminate\Support\Facades\Auth::class,
        'Blade'   => Illuminate\Support\Facades\Blade::class,
        'Bus'     => Illuminate\Support\Facades\Bus::class,
        'Cache'   => Illuminate\Support\Facades\Cache::class,
        'Config'  => Illuminate\Support\Facades\Config::class,
    ],

    // ...
];
```

直接用就好了:

```php
Cache::get(...);
```


## Contract

契约即interface. 接口规定了行为， 使用接口比使用具体类更松耦合,而且契约可以充当框架特性的简明文档:

```php

```
