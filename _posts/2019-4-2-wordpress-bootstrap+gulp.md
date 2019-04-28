---
layout: post
title:
modified:
categories: Tech
 
tags: [wordpress]

comments: true
---

wordpress + bootstrap + gulp.

如何玩的6? 一直崇尚这样的牛逼开发方式,一定要试试,基本上是前后端统一在弄了，真正的全栈.

整理下用的套路:
```sh
1 phpstorm ftp deploy到远程分支，直接开发;
2 wordpress wp-bootstrap-4主题
3 WPGulp来做前端管理
```
由于WPGulp是运行在本地的，不用推到服务器,在phpstorm里设置:

![Screenshot from 2019-04-02 18-29-03-ea1ec0f0-bfcc-46f2-9551-4864d369e0a1](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-04-02%2018-29-03-ea1ec0f0-bfcc-46f2-9551-4864d369e0a1.png)


##  gulp达到的效果? 

gulp是构建前端的自动化框架，再也不用通过wordpress各种钩子插件来干活儿了.

```sh
1. 自动编译 scss
2. 格式化，检查js/css文件
3. 自动压缩js/css
4. 自动合并js/css
5. 自动添加版本信息
    这就是前面说的版本机制更新缓存的重要基础.
6. 其他自动任务
```


## 我想要的效果

```sh
1. 偏重移动端,所以bootstrap如果依赖jquery,需要更新为zepto.这个不紧急.

2. 自动合并这些是肯定的;
    - 很多js, 但不是每个页面都需要用所有的js,所以合并策略需要清晰.
        - 在自定义的文件里，加上前缀, page-xxx-xx.js; post-xxx-xx.js, task根据page, post合并到对应的custrom-page, custom-post里! 666

3. 用gulp自动更新文件名/更新版本号后，如何同步到php的更新里去?
    - 我的理解，可以直接定义task，找到php文件，正则来改对应的代码,
```

对于3，目的是用gulp的task改这,特别留意那个version number,就是需要gulp来更新的地方.
```php
function wp_bootstrap_4_scripts()
{
    // Default unmodified css or js.
    wp_enqueue_style('open-iconic-bootstrap', get_template_directory_uri() . '/assets/css/open-iconic-bootstrap.css', array(), 'v4.0.0', 'all');
    wp_enqueue_style('bootstrap-4', get_template_directory_uri() . '/assets/css/bootstrap.css', array(), 'v4.0.0', 'all');
    wp_enqueue_script('bootstrap-4-js', get_template_directory_uri() . '/assets/js/bootstrap.js', array('jquery'), 'v4.0.0', true);

    // Version updated by WPGulp.
    wp_enqueue_style('wp-style', get_stylesheet_uri(), array(), '117');
    wp_enqueue_style('custom-style', get_template_directory_uri() . '/assets/css/custom.css', array(), '113');

    wp_enqueue_script('custom-js', get_template_directory_uri() . '/assets/js/custom.js', array('jquery'), '100', true);
    wp_enqueue_script('vendor-js', get_template_directory_uri() . '/assets/js/vendor.js', array('jquery'), '100', true);

    if (is_page()){
        wp_enqueue_script('product-js', get_template_directory_uri() . '/assets/js/product.js', array('jquery'), '101', true);
    }
    if (is_singular() && comments_open() && get_option('thread_comments')) {
        wp_enqueue_script('comment-reply');
        wp_enqueue_script('post-js', get_template_directory_uri() . '/assets/js/post.js', array('jquery'), '100', true);
    }
}
```


## 实现


### js

花了点时间，主要在正则修改php文件上.

gulp task,创建4个任务，目标是合并4个js文件, 开发时，把想要合并的文件拖到对应的文件夹，改动即可，

```js
const updateJSVersion = (watchingfile, name)=> {
	let result = null;
	// const crypto = require('crypto');
	// const uuid = require('uuid');
	// var md5 = crypto.createHash('md5');

	function regExpEscape(literal_string) {
		return literal_string.replace(/[-[\]{}()*+!<=:?.\/\\^$|#\s,]/g, '\\$&');
	}

    // base on fs write and read file
	fs.readFile(watchingfile,'utf8',function (err,data) {
		// 3 groups , we want to replace group 2.
        var version;
		// var reg =  /('/assets/js/custom.js', array('jquery'), ')(.*)(')/;
        // Hardest part is here!
        var reg = new RegExp(
        	'(' +
        	regExpEscape("/assets/js/")
			+ name
			+ regExpEscape(".js', array('jquery'), '")
			+ ')'
            + '(.*)'
            + '(' + regExpEscape("'") + ')'
			),
        result = reg.exec(data);
        if (result) {
            version = parseInt(result[2]) + 1;
            version = version.toString();
            console.log('updated version:' + version);
		} else {
            console.log('not find matched part');
		}

		result = data.replace(reg, '$1' + version + '$3');

        fs.writeFileSync(watchingfile, result, { 'flag': 'w' }, function(err) {
            if (err) throw err;
        });
	});
	console.log('modify functions.php done');
};

**
 * Task: `vendorJS`.
 *
 * Concatenate and uglify vendor JS scripts.
 *
 * This task does the following:
 *     1. Gets the source folder for JS vendor files
 *     2. Concatenates all the files and generates vendors.js
 *     3. Renames the JS file with suffix .min.js
 *     4. Uglifes/Minifies the JS file and generates vendors.min.js
 */
const createJSGulpTask = (taskName) => {
	let name = taskName.slice(0,-2) ;
	let srcPrefix = './assets/js/';
	let srcPath = srcPrefix + name;
	let cfg = {
	    jsSrc: srcPath + '/*.js', // name folder
		jsDest: srcPrefix, // destination folder
		jsName: name // destination js file name
	};

	fs.exists(cfg.jsSrc, function(exists) {
	    if(!exists) {
	    	var fs = require('fs-extra');
	        fs.mkdirpSync(srcPath);
		}
	});


	gulp.task( taskName , () => {
		return gulp
			.src( cfg.jsSrc, { since: gulp.lastRun( taskName ) }) // Only run on changed files.
			.pipe( plumber( errorHandler ) )
			.pipe(
				babel({
					presets: [
						[
							'@babel/preset-env', // Preset to compile your modern JS to ES5.
							{
								targets: { browsers: config.BROWSERS_LIST } // Target browser list to support.
							}
						]
					]
				})
			)
			.pipe( remember( cfg.jsSrc ) ) // Bring all files back to stream.
			.pipe( concat( cfg.jsName + '.js' ) )
			.pipe( lineec() ) // Consistent Line Endings for non UNIX systems.
			.pipe( gulp.dest( cfg.jsDest ) )
			.pipe(
				rename({
					basename: cfg.jsName,
					suffix: '.min'
				})
			)
			.pipe( uglify() )
			.pipe( lineec() ) // Consistent Line Endings for non UNIX systems.
			.pipe( gulp.dest( cfg.jsDest) )
			.pipe( notify({ message: '\n\n✅  ===> ' + taskName +  ' — completed!\n', onLast: true }) )
			.on('end',function () {
			    updateJSVersion(config.modifiedPHPFile,name);
			})

	});
};

const jsTasks = ['productJS', 'postJS','customJS','vendorJS'];

jsTasks.forEach( function (value, index, array) {
    createJSGulpTask(value);
})

/**
 * Watch Tasks.
 *
 * Watches for file changes and runs specific tasks.
 */
gulp.task(
	'default',
	gulp.parallel( 'styles', 'images', jsTasks[0], jsTasks[1],jsTasks[2],jsTasks[3],  browsersync, () => {
		gulp.watch( config.watchStyles, gulp.parallel( 'styles' ) ); // Reload on SCSS file changes.
		gulp.watch( config.imgSRC, gulp.series( 'images', reload ) ); // Reload on customJS file changes.
		jsTasks.forEach(function (value, index, array) {
			var  watchJs = './assets/js/' + value.slice(0,-2) + '/*.js';
            gulp.watch( watchJs, gulp.series( value, reload ) );
		})
		// gulp.watch( config.watchPhp, reload ); // Reload on PHP file changes.
	})
);
```
### css

css 基于同样的思路，不过style.css是最主要的文件，简单的做法是干脆全部合并到它

