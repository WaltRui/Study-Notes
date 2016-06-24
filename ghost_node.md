#Ghost 博客系统源码学习笔记
##项目介绍
Ghost 是一套基于 Node.js 构建的开源博客平台。
##项目结构分析
### 1. 项目入口为index.js
``` javascript
var express,
    ghost,
    parentApp,
    errors;

require('./core/server/overrides');//时区设置

// Make sure dependencies are installed and file system permissions are correct.
//启动检测，包括nodeVersion，nodeEnv（主要是配置文件），packages（检查依赖包是否安装）,
contentPath(项目文件路径权限检测)，sqlite（sqlite数据库检查），mail（邮件功能检测），builtFilesExist（/core/built/assets/中是否已经生成了相关文件）
require('./core/server/utils/startup-check').check();

// Proceed with startup
express = require('express');
ghost = require('./core');
errors = require('./core/server/errors');

// Create our parent express app instance.
parentApp = express();

// Call Ghost to get an instance of GhostServer
ghost().then(function (ghostServer) {
    // Mount our Ghost instance on our desired subdirectory path if it exists.
    parentApp.use(ghostServer.config.paths.subdir, ghostServer.rootApp);

    // Let Ghost handle starting our server instance.
    ghostServer.start(parentApp);
}).catch(function (err) {
    errors.logErrorAndExit(err, err.context, err.help);
});

```
此js文件依次进行了：时区的设置，项目运行环境、配置和功能检测，创建express应用开发框架，然后对Ghost猪程序进行初始化（配置加载）以及对Errors处理库的初始化。
其中，时区设置，环境检测等很常规。代码很容易理解。项目的核心部分为core文件夹下Ghost的初始化和启动部分。
###2. ./core/server/index.js文件
从启动文件首先进入的是./core/index.js 文件，不过此文件相对简单，只是对server的简单封装，代码如下，不在详解。
~~~ JavaScript
// ## Server Loader
// Passes options through the boot process to get a server instance back
var server = require('./server');

// Set the default environment to be `development`
process.env.NODE_ENV = process.env.NODE_ENV || 'development';

function makeGhost(options) {
    options = options || {};

    return server(options);
}

module.exports = makeGhost;

~~~
而./core/server/index.js中，主要方法就只有init（）。顾名思义即为对Ghost系统的初始化。
* 首先是进行国际化配置加载。
~~~ JavaScript
    i18n.init();//主要是语言加载（translations文件夹下的json文件），不过目前好像只有英文的配置信息
~~~
然后就是加载config，进行一系列的初始化了
~~~JavaScript
// Load our config.js file from the local file system.
    return config.load(options.config).then(function () {
        return config.checkDeprecated();
    }).then(function () {
        // Initialise the models
        models.init();
    }).then(function () {
        // Initialize migrations
        return migrations.init();
    }).then(function () {
        // Populate any missing default settings
        return models.Settings.populateDefaults();
    }).then(function () {
        // Initialize the settings cache
        return api.init();
    }).then(function () {
        // Initialize the permissions actions and objects
        // NOTE: Must be done before initDbHashAndFirstRun calls
        return permissions.init();
    }).then(function () {
        return Promise.join(
            // Check for or initialise a dbHash.
            initDbHashAndFirstRun(),
            // Initialize apps
            apps.init(),
            // Initialize sitemaps
            sitemap.init(),
            // Initialize xmrpc ping
            xmlrpc.listen(),
            // Initialize slack ping
            slack.listen()
        );
    }).then(function () {
        // Get reference to an express app instance.
        var parentApp = express();

        // ## Middleware and Routing
        middleware(parentApp);

        // Log all theme errors and warnings
        validateThemes(config.paths.themePath)
            .catch(function (result) {
                // TODO: change `result` to something better
                result.errors.forEach(function (err) {
                    errors.logError(err.message, err.context, err.help);
                });

                result.warnings.forEach(function (warn) {
                    errors.logWarn(warn.message, warn.context, warn.help);
                });
            });

        return new GhostServer(parentApp);
    }).then(function (_ghostServer) {
        ghostServer = _ghostServer;

        // scheduling can trigger api requests, that's why we initialize the module after the ghost server creation
        // scheduling module can create x schedulers with different adapters
        return scheduling.init(_.extend(config.scheduling, {apiUrl: config.url + config.urlFor('api')}));
    }).then(function () {
        return ghostServer;
    });

~~~
这里面每个then方法下的模块初始化都很类似。几乎都用到了Promise规范的相关内容，初学nodejs(JavaScript也初学)，Promise相对较为陌生，
花了很久才搞懂，这期间，也被call和this的使用折腾了很久。如果对这些不是很熟悉的话，请移步相关知识点和扩展章节。
##相关知识点
####1. Process对象
Process对象是Node的一个全局对象，提供当前Node进程的信息。

相关链接：

http://javascript.ruanyifeng.com/nodejs/process.html
####2.部分全局变量
* module.filename：开发期间，该行代码所在的文件。
* __filename：始终等于 module.filename。
* __dirname：开发期间，该行代码所在的目录。
* process.cwd()：运行node的工作目录，可以使用  cd /d 修改工作目录。
* require.main.filename：用node命令启动的module的filename, 如 node xxx，这里的filename就是这个xxx。

####3. Promise
Promise是CommonJS的规范之一，拥有resolve、reject、done、fail、then等方法，能够帮助我们控制代码的流程，避免函数的多层嵌套。

如 Q 、 bluebird 和 Deferred 等均为Promise标准的模块。

参考：

http://www.zhangxinxu.com/wordpress/2014/02/es6-javascript-promise-%E6%84%9F%E6%80%A7%E8%AE%A4%E7%9F%A5   （Promise通俗理解，不涉及技术）

https://segmentfault.com/a/1190000000684654
##扩展内容
####1. Call & Apply
Call 方法可以用来代替另一个对象调用一个方法。Apply，应用某一对象的一个方法，用另一个对象替换当前对象。

参考：http://www.cnblogs.com/huyong/p/4139875.html
####2. this的使用
在Javascript中,This关键字永远都指向函数(方法)的所有者。

参考：http://www.laruence.com/2009/09/08/1076.html
####3. JavaScript作用域
JS权威指南中有一句很精辟的描述:
> “JavaScript中的函数运行在它们被定义的作用域里,而不是它们被执行的作用域里.”

参考：http://www.laruence.com/2009/05/28/863.html

