---
layout: post
title:
modified:
categories: Tech
tags: [lavarel]
comments: true
---


<!-- TOC -->

- [订阅者方式重写模型事件](#订阅者方式重写模型事件)
    - [添加Event类](#添加event类)
    - [添加事件处理到User Model](#添加事件处理到user-model)
    - [消费者订阅事件](#消费者订阅事件)
    - [触发事件](#触发事件)
- [观察者方式处理](#观察者方式处理)

<!-- /TOC -->

<https://laravelacademy.org/post/9713.html>

Controller的逻辑一部分是业务的，一部分就是往model里CUID了，添加一些cuid的回调是绝对有必要的.

## 订阅者方式重写模型事件

已知模型事件有:
```sh
retrieved：获取到模型实例后触发
creating：插入到数据库前触发
created：插入到数据库后触发
updating：更新到数据库前触发
updated：更新到数据库后触发
saving：保存到数据库前触发（插入/更新之前，无论插入还是更新都会触发）
saved：保存到数据库后触发（插入/更新之后，无论插入还是更新都会触发）
deleting：从数据库删除记录前触发
deleted：从数据库删除记录后触发
restoring：恢复软删除记录前触发
restored：恢复软删除记录后触发
```

给机会让你在model的声明周期进行各种callback.

最简单的就是在Provider:boot里直接添加
```php
// app/Providers/EventServiceProvider.php

public function boot()
{
    parent::boot();

    // 监听模型获取事件
    User::retrieved(function ($user) {
        Log::info('从模型中获取用户[' . $user->id . ']:' . $user->name);
    });
}
```

但是不够优雅?来个`观察者模式`?

### 添加Event类
在`app/Events`下创建:
```sh
php artisan make:event UserCreate
```

1 添加User model进来by __construct

```php
<?php

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Queue\SerializesModels;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;

use \App\User;

class UserCreate
{
    use Dispatchable, InteractsWithSockets, SerializesModels;


    public $user;

    /**
     * Create a new event instance.
     *
     * @param User $user
     */
    public function __construct(User $user)
    {
        //1 Add user instance . from Where?
        $this->user = $user;
    }

    /**
     * Get the channels the event should broadcast on.
     *
     * @return \Illuminate\Broadcasting\Channel|array
     */
    public function broadcastOn()
    {
        return new PrivateChannel('channel-name');
    }
}

```
### 添加事件处理到User Model
```php
    //Added events for User model
    protected $dispatchesEvents = [
        'creating' => UserCreate::class,
    ];
```

这样发生的事件就会到响应的类里，可以看到类里有广播.可有自定义的BD通道.

### 消费者订阅事件

光有事件还不行，哪里事件呢?
我倒是佩服这种脑洞大的用法...你倒是1条命令就全部搞完呀...
```sh
php artisan make:listener UserEventSubscirber
```
生成`app/Listners/UserEventSubscirber`

添加Event handler:
```php
public function onUserCreate($event)
{
    Log::info('Deleted event handle');
}
public  function onUserDeleted($event)
{

    Log::info('Deleted event handle');
}
//绑定handle与eventclass
public function subscribe($events)
{
    $events->listen(
        UserCreate::class,
        UserEventSubscirber::class . '@onUserDeleting'
    );

    $events->listen(
        UserUpdate::class,
        UserEventSubscirber::class . '@onUserDeleted'
    );
}
```
还需使订阅者生效:
```php
// app/Providers/EventServiceProvider.php

protected $subscribe = [
    UserEventSubscriber::class
];
```

### 触发事件

最后是生产者了，就是在逻辑里使用该model即可.
```
$user->create();
$user->update();
```

## 观察者方式处理

上面的流程了解下`Event`,`Listener`是怎么工作的还好，但是太麻烦了..

迅猛的方法,直接创建Observer,回调再里面写，so easy.
```sh
 php artisan make:observer UserObserver --model=User
```

有一丝丝想念wordpress的`add_filter`了...那就1个又快又好.