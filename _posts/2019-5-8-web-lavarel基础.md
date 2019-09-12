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
- [服务提供者](#服务提供者)
- [Facade](#Facade)
- [Contract](#Contract)
- [MVC](#MVC)
  - [Controller](#Controller)
  - [View](#View)
  - [Model](#Model)
- [分层的思想](#分层的思想)
- [DB](#DB)

<!-- /TOC -->

## 服务容器

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

可以在`AppServiceProvider`添加启动和绑定,也能自己定义 provider,通过命令更快.

注册抽象类，具体实现一般由`config/app.php`下的 provider 配置来提供.

每个运行的实例都从 bootstrap/app.php 脚本获取 Laravel 应用实例，

采用这种配置的方式，其实就是使用了`抽象工厂`模式.

artisan 生成:

```sh
php artisan make:provider RiakServiceProvider
```

## Facade

```sh
 Laravel facades serve as "static proxies" to underlying classes in the service container, providing the benefit of a terse, expressive syntax while maintaining more testability and flexibility than traditional static methods.
```

可以理解为操作复杂对象的简化方法,就是外观模式里的外观类的方法.

## Contract

## MVC

### Controller

`app/Http/Controller下`定义的类，其方法可以在 Router 里调用,作为 controller 逻辑

### View

支持 blade 模板，php 和 css.

给试图传数据:

```sh
return view('home')->with('tasks', Task::all());
or
return view('home', ['tasks' => Task:all()]);
```

Task 什么鬼?

View 共享变量给他人:

```sh
view()->share('siteName', 'Laravel学院');
```

View Composer:
当视图被渲染时的回调函数或类方法.
1 创建`app/Http/View/Composers`
2 提供 ComposerProvider 　来绑定到容器，才能使用.

```sh
php artisan make:provider ComposerServiceProvider
2 在boot里添加视图和composer类的绑定
3 在composer类里实现compose方法为回调.
```

### Model

指定目录,生成的 model 为`app/Model/Demo.php`,否则默认为`app/Demo.php`

```sh
php artisan make:model Model/Demo
```

简单点就是下面的 DB 了.

Q: 1 个 Model 对象是如何与对应的表关联起来的?
model 里加个属性`table`

```php
protected $table = 'tags';
```

## 分层的思想

<https://laravelacademy.org/post/9711.html#toc_2>

不要束缚在 MVC 中，只要对项目进行合理的 OO 抽象.

简单点就是不要把 Controller 写复杂了，把业务逻辑分到独立的层，更清晰.

就是设计模式里的`单一职责`.

```php
class BillingController extends BaseController
{

    //biller放哪呢?
    /* App/Billler/Biller
    */

    //biller通过构造函数来注入
    public function __construct(BillerInterface $biller)
    {
        $this->biller = $biller;
    }

    public function postCharge(Request $request)
    {
        //账单的处理交给biller
        $this->biller->chargeAccount(Auth::user(), $request->input('amount'));
        return view('charge.success');
    }

}
```

## DB

创建数据库迁移文件:`database/migrations`下

```sh
//创建
php artisan make:migration create_users_table --create=users
//更新
php artisan make:migration create_users_table --table=users
//修改某表
php artisan make:migration alter_users_add_nickname --table=users
```

数据表本身的创建，修改，在 up 里做就行.

执行变更:

```sh
php artisan migrate
//可回滚 回滚到指定版本
php artisan migrate:rollback
//重置
php artisan migrate:reset
```

遇到连接问题，是配置缓存的问题,清空一下.

```sh
php artisan cache:clear
php artisan config:cache
```

数据填充，方便测试:

```sh

php artisan make:seed UsersTableSeeder
php artisan db:seed
```

`database/seeds` 下的 XXXSeeder 就是产生测试数据的类.

更猛一点，用工厂产生数据,在`database/factories`下.

```sh
php artisan make:factory Usersactory --model='\App\User'
```

E:执行 db:seed 时报错，说找不到 UsersFactory 类,命名定义了.
php 是根据类名反射创建对象的,有个配置在 composer.json 里:

```sh
    "autoload": {
        "psr-4": {
            "App\\": "app/"
        },
        "classmap": [
            "database/seeds",
            "database/factories"
        ]
    },
```

搜了下重新运行`composer dumpautoload`就好.
