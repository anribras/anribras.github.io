---
layout: post
title:
modified:
categories: Tech
tags: [thinksns]
comments: true
---
<!-- TOC -->

- [基础件](#基础件)
    - [服务容器:](#服务容器)
    - [服务提供者](#服务提供者)
    - [Facade](#facade)
- [MVC](#mvc)
    - [Controller:](#controller)
    - [View:](#view)
- [DB](#db)
- [Laravel mix](#laravel-mix)

<!-- /TOC -->

# 基础件

## 服务容器:

<https://laravelacademy.org/post/769.html>

<https://laravelacademy.org/post/9534.html>


容器需要解析真正的对象,
```php
//way 1
$fooBar = $this->app->make('HelpSpot\API');

//way 2
$api = resolve('HelpSpot\API');


//way3
$api = $this->app->makeWith('HelpSpot\API', ['id' => 1]);

//way4 自动注入，即在controller的自动构造函数里，添加需要解析对象的参数,并保存它
class UserController extends Controller{
    protected $users;
    public function __construct(UserRepository $users)
    {
        $this->users = $users;
    }

}
```


## 服务提供者

放在`app/Providers`.

可以在`AppServiceProvider`添加启动和绑定,也能自己定义provider,通过命令更快.
```
```

注册抽象类，具体实现一般由`config/app.php`下的provider配置来提供.

每个运行的实例都从 bootstrap/app.php 脚本获取 Laravel 应用实例，

采用这种配置的方式，其实就是使用了`抽象工厂`模式.

artisan生成:
```sh
php artisan make:provider RiakServiceProvider
```

## Facade
```sh
 Laravel facades serve as "static proxies" to underlying classes in the service container, providing the benefit of a terse, expressive syntax while maintaining more testability and flexibility than traditional static methods.
```

可以理解为操作复杂对象的简化方法,就是外观模式里的外观类的方法.




# MVC

## Controller:
`app/Http/Controller下`定义的类，其方法可以在Router里调用,作为controller逻辑

## View:
支持blade模板，php和css.

给试图传数据:
```sh
return view('home')->with('tasks', Task::all());
or 
return view('home', ['tasks' => Task:all()]);
```
Task什么鬼?

View共享变量给他人:
```sh
view()->share('siteName', 'Laravel学院');
```

View Composer:
当视图被渲染时的回调函数或类方法.
1 创建`app/Http/View/Composers`
2 提供ComposerProvider　来绑定到容器，才能使用.
```sh
php artisan make:provider ComposerServiceProvider
2 在boot里添加视图和composer类的绑定
3 在composer类里实现compose方法为回调.
```

# DB

创建数据库迁移文件:
```sh
//创建
php artisan make:migration create_users_table --create=users
//更新
php artisan make:migration create_users_table --table=users
//修改某表
php artisan make:migration alter_users_add_nickname --table=users
```

数据表本身的创建，修改，在up里做就行.

执行变更:
```sh
php artisan migrate 
//可回滚 回滚到指定版本
php artisan migrate:rollback
```




`resource/views`

Model:

# Laravel mix

`resource/js/app.js` 引入其他js模块

`resource/sass/app.scss`　引入其他css模块

webpack都配置好了.




