---
layout: post
title: 前端工具链（二）：Task Runner - Grunt
tags:
    - frontend
    - toolchain
    - nodejs
    - grunt
comments: true
---

Grunt 是一个 JavaScript task runner, 用于工作流程的自动化，如 linting, 单元测试，编译，minification 等。 Grunt 本身更像是个框架，许多的任务都靠它庞大的插件库来完成。

## 安装

Grunt 依赖于 node.js 和 npm. 它分为 grunt-cli 和 grunt 两个 package. 其中 grunt-cli 只是一个 CLI, 它通过 node.js 的 require() 来加载本地目录的 grunt. 因此，Grunt 的安装分为两个部分。

1. 全局安装 grunt-cli.

    ```
    npm install -g grunt-cli
    ```

    选择全局安装，才能方便的在 shell 中调用 grunt 命令

2. 在项目中安装 grunt.

    ```
    npm install --save-dev grunt

    ```

## 配置文件 Gruntfile.js

Gruntfile.js 是 Grunt 的配置文件，存放在项目的根目录下，采用 JavaScript 语法。Grunt 也可以用 CoffeeScript 写配置文件，文件名为 Gruntfile.coffee.

下面是一个 Gruntfile 的例子。

```javascript
module.exports = function(grunt) {

  // Project configuration.
  grunt.initConfig({
    pkg: grunt.file.readJSON('package.json'),
    concat: {
      options: {
        seperator: ';'
      },
      dist: {
        src: ['src/**/*.js'],
        dest: ['dist/assets/main.js']
      }
    }
  });

  grunt.loadNpmTasks('grunt-contrib-concat');

  grunt.registerTask('default', [
    'concat'
  ]);

};
```

可以看到， Grunt 使用了 node.js 风格的导出函数。在函数中，首先调用 `initConfig` 方法来初始化配置对象 `grunt`. 

在 initConfig 中，首先传入 `package.json` 中的项目配置，这个在后面可以供 Grunt 引用。

接下来则是 task concat 的配置。每个 task 的配置都可以包含一个 `options` 对象，用于向传递配置参数。在 `options` 之外，还可以包含多个可自行命名的对象，这些就是被配置的 targets. 在这个例子中，concat 配置了一个叫 `dist` 的 target. Target 一般要配置源文件 `src` 和输出文件 `dest`. 当然，也有许多 tasks 是不需要输出文件，或者有默认的输出文件的。这种 tasks 可以不指定 `dest`. 用于指定输入输出文件的格式还有好几种，请参考 [Configuring Tasks][grunt-configuring-tasks].

在传递完配置后，使用 `loadNpmTasks` 方法从插件中加载 tasks. 当然，这个插件也要用 npm 安装好。

最后，用 `registerTask` 方法注册了一个新 task. 一个 task 可以由多个子 task 构成，形成构建流水线。

## 使用

Grunt 可以使用以下三种方式调用：

1. 指定 task 和 target, 如

    ```
    grunt concat:dist
    ```

2. 指定 task, 执行该 task 中的所有 target, 如

    ```
    grunt concat
    ```

3. 不带参数，自动执行 task `default`

    ```
    grunt
    ```

[grunt-configuring-tasks]: http://gruntjs.com/configuring-tasks "Configuring tasks - Grunt: The JavaScript Task Runner"
