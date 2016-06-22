#Ghost 博客系统源码学习笔记
##项目介绍
Ghost 是一套基于 Node.js 构建的开源博客平台。
##新增知识点
1. process对象是Node的一个全局对象，提供当前Node进程的信息
相关链接：http://javascript.ruanyifeng.com/nodejs/process.html
2. module.filename：开发期间，该行代码所在的文件。
__filename：始终等于 module.filename。
__dirname：开发期间，该行代码所在的目录。
process.cwd()：运行node的工作目录，可以使用  cd /d 修改工作目录。
require.main.filename：用node命令启动的module的filename, 如 node xxx，这里的filename就是这个xxx。











##项目结构分析
项目入口为index.js
``` javascript
var express,
    ghost,
    parentApp,
    errors;

require('./core/server/overrides');//时区设置

// Make sure dependencies are installed and file system permissions are correct.
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