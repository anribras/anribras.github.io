---
layout: post
title:
modified:
categories: Tech
tags: [web, docker]
comments: true
---

## laradock

git pull 后记得把最新的 env-sample copy 到`.env`.

`.env`就是核心，各种配置都在里面，可能有的需要 build，有的直接 docker restart 就生效.

lnmp 环境:

```sh
docker-compose up -d nginx mariadb php-fpm phpmyadmin
```

build 肯定很慢...,而且容易各种问题...

失败了多来几次..

其实都是下载出了问题...不行挂个全局代理
<https://www.looaon.com/index.php/%E5%BA%94%E7%94%A8%E9%85%8D%E7%BD%AE/832.html>

```sh
export http_proxy="http://127.0.0.1:8123/"
```

代理不行再挂回来...我服了我自己...

```sh
systemctl start polipo.service
systemctl stop polipo.service
```

### workspace

貌似很有用，不用每个工程都安装一堆依赖了.(但是需要 laradock...一回事儿么..)

例如项目分布如下:

```sh
+project
+laradock
```

例如把当前工作区(../) 挂载到 workspace 容器的 /var/www/

添加个 alias,爽歪歪:

```sh
alias lbash='docker exec -ti --user=laradock laradock_workspace_1 bash'
alias lnginx='docker exec -ti laradock_nginx_1 bash'
alias lmysql='docker exec -ti laradock_mariadb_1 bash'
alias lphp='docker exec -ti laradock_php-fpm_1 bash'
```

#### 其他配置

- php-fpm 有报错:

```sh
PHP Warning:  PHP Startup: Unable to load dynamic library 'tideways.so' (tried: /usr/local/lib/php/extensions/no-debug-non-zts-20170718/tideways.so (/usr/local/lib/php/extensions/no-debug-non-zts-20170718/tideways.so: cannot open shared object file: No such file or directory), /usr/local/lib/php/extensions/no-debug-non-zts-20170718/tideways.so.so (/usr/local/lib/php/extensions/no-debug-non-zts-20170718/tideways.so.so: cannot open shared object file: No such file or directory)) in Unknown on line 0
```

某个插件没装好?<https://github.com/laradock/laradock/issues/2169>

- laravel 权限问题

<https://laracasts.com/discuss/channels/general-discussion/laravel-framework-file-permission-security>

- phpmyadmin 登录

因为自己配置了 mariadb..所以登录的 docker host 是 mariadb.

laravel 工程里的配置也改为相应的 mariadb.

如果 local mysql-cli 直接用`mysql -h 127.0.0.1 -u xx -p xx`即可登录

增加 database 通过 phpmyadmin 体验更佳...命令行从来就没注意过.

- php artisan

这个不是在 workspace 里运行...是在 docker php-fpm 里..记住了.

workspace 里的 php 有啥用？估计配合 composer 来安装,也可以做 phpstorm 的 interpreter.

- 容器内 api 函数

其实也就是想彻底的前后端分离。controller 想调用 api 来拉数据，而不是重新写一遍 model...

超级坑爹的点. 应该是仅在 docker 环境下有.

php-fpm 容器里 curl 访问`localhost/api/v2/xxx`，是不行的..

必须用`http://laradock_nginx_1/api/v2/xxx`的方式.

连`http://172.22.0.3/api/v2/xxx`都不行...也是醉了.

具体来说.

具体是这函数:

```php
function api($method = 'POST', $url = '', $params = [])
{
    $client = new Client([
//        'base_uri' =>  config('app.url'),
        'base_uri' => config('app.docker_nginx_host'),
        'defaults' => [
            'exceptions' => false
        ]
    ]);
    $headers = ['Accept' => 'application/json', 'Authorization' => 'Bearer '.Session::get('token')];
    if ($method == 'GET') {
        $response = $client->request($method, $url, [
            'query' => $params,
            'headers' => $headers,
            'http_errors' => false,
        ]);
    } else {
        $response = $client->request($method, $url, [
            'form_params' => $params,
            'headers' => $headers,
            'http_errors' => false,
        ]);
    }
    return json_decode($response->getBody(), true);
}
```

- xdebug

一直没真正搞清楚 xdebug 的真正原理.

大概有 2 种模式,一种非代理(直连)，一种代理模式.

第 1 种模式:

```sh
phpstorm及配置的php interpreter端为server, listen 9000;
   注意这个配置的interpreter，可以是本机直接安装的，docker,甚至远程的...
   最直观就是本机直接安装的

web server端的php-fpm为client,通过其xdebug模块连接到server,进行debug交互.
```

第 2 种模式:

- 切换端口

好吧..调试环境 xx.xx 很 ok,如果是`xx.xx:8000`,需要哪些改动?

```sh
1. laradocker nginx 对外映射端口改为8000;
2. laravel 工程 app.url改为`xx.xx:8000`;
3. js axios的baseurl改为`xx:xx:8000`;
4. xdebug的端口同样改为`localhost:8000`;
```

至此，可以告别之前的 dnmp 了, 艰难的开始 laradock 之旅.

- 时区纠正
  .env:

```sh
WORKSPACE_TIMEZONE=PRC
```

我用的 mariadb,但是 laradock 里又没有添加 TZ 设置:
docker-compose.yml:

```sh
mariadb:
  environment:
        # add
        - TZ=${WORKSPACE_TIMEZONE}
      n
```

然后重启 mariadb 容器服务即可.

## 理想的前端调试环境

折腾了一个周六，终于可以小小的总结一把了.

因为分本地和云端. .js/css 等开发的文件在本地.

- app.url APP_URL 等变量都写 localhost.

- laravel-mix 的 browserify proxy 代理本地

```js
mix.browserSync({
    proxy: 'http://localhost:8000',
    files: [
        xxx,
    ],
    reloadOnRestart: true,
    watchOptions: {
        usePolling: true,
    },
```

- mariadb 在 laravel 工程的.env(or env.yml),配置为云端.

- js 里的 axios 请求仍然采用云端,(restful 最新实现在云端)

- 本地 js/css 基于上面弄好的 laradock. laravel-mix 配置 browserify, 改后即见.

复杂是复杂了点，但再也不用疯狂传云端，然后 F5 的土鳖 debug 法了...

而是考虑了项目实际同时又采用了比较现代的前端构建与开发的方式.
