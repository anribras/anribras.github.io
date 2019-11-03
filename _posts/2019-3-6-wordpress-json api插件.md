---
layout: post
title:
modified:
categories: Tech

tags: [wordpress]
---

<!-- TOC -->

- [插件初始化](#插件初始化)
- [JSON_API 对象](#JSON_API-对象)
  - [query](#query)
  - [如何扩展](#如何扩展)
  - [具体在哪里执行](#具体在哪里执行)

<!-- /TOC -->

json-api 系列的插件，简单好用，我喜欢.

调用都很简单:

```sh
http://xxx/api/get_info
```

怎么实现的?

## 插件初始化

核心是让 rewrite 生效.

```php
// Add initialization and activation hooks
add_action('init', 'json_api_init');
register_activation_hook("$dir/json-api.php", 'json_api_activation');
register_deactivation_hook("$dir/json-api.php", 'json_api_deactivation');

function json_api_activation() {
  // Add the rewrite rule on activation
  global $wp_rewrite;
  add_filter('rewrite_rules_array', 'json_api_rewrites');
  $wp_rewrite->flush_rules();
}

function json_api_deactivation() {
  // Remove the rewrite rule on deactivation
  global $wp_rewrite;
  $wp_rewrite->flush_rules();
}
```

路由重写的方式:

```sh
add_filter('rewrite_rules_array', 'json_api_rewrites');

function json_api_rewrites($wp_rules) {
  $base = get_option('json_api_base', 'api');
  if (empty($base)) {
    return $wp_rules;
  }
  $json_api_rules = array(
    "$base\$" => 'index.php?json=info',
    "$base/(.+)\$" => 'index.php?json=$matches[1]'
  );
  return array_merge($json_api_rules, $wp_rules);
}
```

实际的实现，都是在原 wp 增加１个 json=的参数的基础上，比原生 rest 简单了很多倍.

## JSON_API 对象

核心的 3 坨,也是 3 个对象:

```php
$this->query = new JSON_API_Query();
$this->introspector = new JSON_API_Introspector();
$this->response = new JSON_API_Response();
```

### query

即 class JSON_API_Query.
query 是把 url 里参数转成了对象，方便使用.
核心动作是 get:

```php
  function get($key)
  {
    if (is_array($key)) {
      $result = array();
      foreach ($key as $k) {
        $result[$k] = $this->get($k);
      }
      return $result;
    }
    $query_var = (isset($_REQUEST[$key])) ? $_REQUEST[$key] : null;
    $wp_query_var = $this->wp_query_var($key);
    if ($wp_query_var) {
      return $wp_query_var;
    } else if ($query_var) {
      return $this->strip_magic_quotes($query_var);
    } else if (isset($this->defaults[$key])) {
      return $this->defaults[$key];
    } else {
      return null;
    }
  }
```

各种找 key，核心是 wp_query_var,又是调了 wp 的`get_query_var`,也就是 wp 的查询入口，也可以说是查询类 restapi 的核心了，可查哪些东西?
<https://codex.wordpress.org/Class_Reference/WP_Query#Parameters>

### 如何扩展

plugin 里处处都用的对象，自然是重点了.

json-api 把功能按 controller 分,可以很方便的扩展自己需要的 controller,比如核心的 core,posts,respond 等，后期扩展的 user 等，若自己要扩展也是按这个套路.

具体怎么做的?非常简单粗暴...

controllers 文件夹下，如果定义了`class JSON_API_XXX_Controller`的类，那么就识别为１个 controller.

```php

  function check_directory_for_controllers($dir, &$controllers) {
    $dh = opendir($dir);
    while ($file = readdir($dh)) {
      if (preg_match('/(.+)\.php$/i', $file, $matches)) {
        $src = file_get_contents("$dir/$file");
        if (preg_match("/class\s+JSON_API_{$matches[1]}_Controller/i", $src)) {
          $controllers[] = $matches[1];
        }
      }
    }
  }
```

### 具体在哪里执行

给`template_redirect`注册了个钩子:

```php
add_action('template_redirect', array(&$this, 'template_redirect'));

function template_redirect() {
    // Check to see if there's an appropriate API controller + method
    $controller = strtolower($this->query->get_controller());
    $available_controllers = $this->get_controllers();
    $enabled_controllers = explode(',', get_option('json_api_controllers', 'core'));
    $active_controllers = array_intersect($available_controllers, $enabled_controllers);

    if ($controller) {

      if (empty($this->query->dev)) {
        error_reporting(0);
      }

      if (!in_array($controller, $active_controllers)) {
        $this->error("Unknown controller '$controller'.");
      }

      $controller_path = $this->controller_path($controller);
      if (file_exists($controller_path)) {
        require_once $controller_path;
      }
      $controller_class = $this->controller_class($controller);

      if (!class_exists($controller_class)) {
        $this->error("Unknown controller '$controller_class'.");
      }

      $this->controller = new $controller_class();
      $method = $this->query->get_method($controller);

      if ($method) {

        $this->response->setup();

        // Run action hooks for method
        //如果没有hook，就是创建钩子.
        do_action("json_api", $controller, $method);
        do_action("json_api-{$controller}-$method");

        // Error out if nothing is found
        if ($method == '404') {
          $this->error('Not found');
        }

        // Run the method
        $result = $this->controller->$method();
        // Handle the result
        $this->response->respond($result);
        // Done!
        exit;
      }
    }
}
```
