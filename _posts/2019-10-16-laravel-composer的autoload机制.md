---
layout: post
title:
modified:
categories: Tech
tags: [web,laravel]
comments: true
---


## 为啥需要autoload

autoload就是代替手动的`include/require`:

```php
self::$loader = $loader = new \Composer\Autoload\ClassLoader();
```

执行上面的时候，自动调用1个方法来require.

具体地,通过`spl_autoload_register`将想要调用的函数注册到1个队列里，
系统会执行队列的所有注册回调，去加载真正的类文件.

```sh
spl_autoload_register(array('ComposerAutoloaderInit063adac3dfb9af8e4cef0fc21a448d00', 'loadClassLoader')
```

上面是`loadCloassLoader`:

就是3个要素:

```sh
1 类名
2 对应的文件名
3 require的动作
```

```php
    if ('Composer\Autoload\ClassLoader' === $class) {
        require __DIR__ . '/ClassLoader.php';
    }
```

## composer实现

上面其实就是整个composer自动加载的第1个类,是个statis变量,
位于`auload_real.php`:

```php
self::$loader = $loader = new \Composer\Autoload\ClassLoader();
```

这个loader是一开始就会调用的

```php
//index.php
require __DIR__.'/../vendor/autoload.php';
//autoload.php
require_once __DIR__ . '/composer/autoload_real.php';
return ComposerAutoloaderInit063adac3dfb9af8e4cef0fc21a448d00::getLoader();

//autoload_real.php
//getLoader:
public static function getLoader()
{
        if (null !== self::$loader) {
            return self::$loader;
        }

        spl_autoload_register(array('ComposerAutoloaderInit063adac3dfb9af8e4cef0fc21a448d00', 'loadClassLoader'), true, true);
        self::$loader = $loader = new \Composer\Autoload\ClassLoader();
        spl_autoload_unregister(array('ComposerAutoloaderInit063adac3dfb9af8e4cef0fc21a448d00', 'loadClassLoader'));

        $useStaticLoader = PHP_VERSION_ID >= 50600 && !defined('HHVM_VERSION') && (!function_exists('zend_loader_file_encoded') || !zend_loader_file_encoded());
        if ($useStaticLoader) {
            require_once __DIR__ . '/autoload_static.php';

            call_user_func(\Composer\Autoload\ComposerStaticInit063adac3dfb9af8e4cef0fc21a448d00::getInitializer($loader));
        } else {
            $map = require __DIR__ . '/autoload_namespaces.php';
            foreach ($map as $namespace => $path) {
                $loader->set($namespace, $path);
            }

            $map = require __DIR__ . '/autoload_psr4.php';
            foreach ($map as $namespace => $path) {
                $loader->setPsr4($namespace, $path);
            }

            $classMap = require __DIR__ . '/autoload_classmap.php';
            if ($classMap) {
                $loader->addClassMap($classMap);
            }
        }

        //核心require在这里.
        $loader->register(true);

        if ($useStaticLoader) {
            $includeFiles = Composer\Autoload\ComposerStaticInit063adac3dfb9af8e4cef0fc21a448d00::$files;
        } else {
            $includeFiles = require __DIR__ . '/autoload_files.php';
        }
        foreach ($includeFiles as $fileIdentifier => $file) {
            composerRequire063adac3dfb9af8e4cef0fc21a448d00($fileIdentifier, $file);
        }

        return $loader;
    }
}
```

register:

```php
spl_autoload_register(array($this, 'loadClass'), true, $prepend);
```

loadClass:

```php
    public function loadClass($class)
    {
        if ($file = $this->findFile($class)) {
            includeFile($file);

            return true;
        }
    }
```

loadClass函数就是实现上面核心三点.


### composer都干了啥

可以看到`vendor/composer`多了很多自动生成的文件:

```sh
├── autoload_classmap.php
├── autoload_files.php
├── autoload_namespaces.php
├── autoload_psr4.php
├── autoload_real.php
├── autoload_static.php
├── ClassLoader.php
├── installed.json
└── LICENSE

```

整个跟composer包里的配置是相关的，把composer.json和子包里的composer.json与autoload相关的json设置都导入到了上面的几个文件(php)里:

```json
    "autoload": {
        "files": [
            "app/helpers.php"
        ],
        "classmap": [
            "database/seeds",
            "database/factories"
        ],
        "psr-4": {
            "Zhiyi\\Plus\\": "app/"
        }
    },
```

一共4种模式:
<https://my.oschina.net/sallency/blog/893518https://my.oschina.net/sallency/blog/893518>

只有有个困惑`composer dump-autoload`不知道干嘛的，现在很清楚了，就是在autoload规则发生变化后，自动生成composer下新的映射php文件
