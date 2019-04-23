Gulp#

gulp是一个前端自动化构建工具，通过代码优于配置的策略，Gulp 让简单的任务简单，复杂的任务可管理。

全局安装

npm install --global gulp
作为项目的开发依赖(devDependencies)安装

npm install --save-dev gulp
在项目根目录下创建一个名为 glupfile.js 的文件

var gulp = require('gulp');

gulp.task('default', function() {
  // 将你的默认的任务代码放在这
});
运行 gulp

gulp
