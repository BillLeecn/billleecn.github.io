---
layout: post
title: 前端工具链（一）：Runtime 与 Package Manager
tags:
    - frontend
    - toolchain
    - nodejs
    - npm
comments: true
---

## 越来越复杂的前端

Web 上的动态内容和应用越来越多，在很多地方已经取代了原有的本地应用。随着前端逻辑变得越来越复杂，代码的模块化、复用也更加重要。把大量模块化的代码手工拼接起来直接丢给浏览器运行的做法不再可行。

使用工具链来处理前端代码有以下优势。

* 自动管理模块依赖

    在后端，Java 有 maven 来管理依赖库，有 import 语句来声明依赖关系；python 也有 pip 和 import; 连最古老的 C 都有 import (但是没有统一的依赖管理). 但原生的 javascript 却什么都没有，各种库都只能手动添加到全局空间，各个模块的载入顺序就是在 HTML 里引入的顺序。在依赖关系复杂时，这种管理方式就是个噩梦。

    引入自动管理依赖关系的工具，可以大大提高开发效率。

* 提高性能

    Javascript 是不需要编译的，可以直接由浏览器执行。但是这些代码是要通过网络传输到浏览器。Javascript 里面的各种空格、换行、常常的变量名、函数名，都在浪费传输的带宽，减慢页面的加载速度。Javascript 模块要一个个加载就更是慢了。

    通过工具，可以 minify 这些 javascript, 并把所有文件连接成一个，提高页面加载的速度。

* 好用的语法糖

    CSS 在不同的浏览器之间的兼容性还不太好，可以用工具来自动添加前缀。很多浏览器还不支持 ES 6 的特性，没关系，把 ES 6 编译成旧版本的 javascript. 你说你就是觉得 javascript 不好用，没关系，你可以去用 TypeScript, CoffeeScript 等等，这些都能编译成 javascript.


## 那现在开始吧

### Javascript Runtime: Node.js

这些工具本身，也是用 javascript 编写的，因此首先要有一个可以在浏览器外运行 javascript 的环境。

[Node.js][nodejs] 是一个用 Chrome 的 V8 构建的 JavaScript runtime. 为了继承浏览器中的 javascript 单线程、事件驱动的特性，利用好 javascript 方便的函数回调，Node.js 也使用事件驱动、非阻塞的 I/O 方式。

不过这个不是这里要关注的重点，现在又不是要编写后端服务器。现在用 node.js 只是因为这些工具都依赖于这个 JavaScript runtime. 请按照[这里][nodejs-install]的说明安装 node.js.

### Package Manager of Node.js

在用上面的方法装好 Node.js 的同时，也装好了 Node.js 的 package manager, npm. 如果你用过 python, 你可能会认为这是和 pip 差不多的东西。但是，npm 其实不只是 pip, 它不只是用来从 registry 下载或发布 packages 的工具，同时也是管理当前项目的工具，可以把它理解为 pip + setuptools + 半个 make.

下面将介绍 npm 的基本使用。

#### 全局安装 Package

有些 package 带有可执行程序，这样的 package 要全局安装，shell 才能找到它的可执行程序。比如说，npm 本身就是这样的一个 package. 可以用以下命令更新 npm:

```
npm install -g npm
```

在这里，`-g` 意味着 global. 一般说来，很少会使用全局安装。

#### 建立一个新工程

npm 的依赖关系是以 package 为主体来管理的，要使用 npm 来管理依赖，首先要建立一个新 package. 在工程的根目录下，执行

```
npm init
```

这个npm 会以向导方式提示输入一些关于这个 package 的基本描述，根据这些信息生成 `package.json`. 如果这只是一个前端工程，引入 npm 只是用来管理依赖，不需要发布到 npm registry 上，那么这些信息也就随便填填就好了。

#### 安装依赖 package

在 npm 中，一个 package 的依赖会单独安装到本 package 根目录的 node_modules 目录下。

```
npm install <package>
```

#### 记录依赖关系

在安装依赖时，加入 `--save` 参数，那么 npm 会自动把安装的包加入 package.json 的 `dependencies` 中。以后使用不带参数的 `npm install` 时，这些依赖就会被自动安装。使用 `--save-dev` 则会加入 `devDependencies`. 当使用 `npm install --production` 时，`devDependencies` 中的依赖不会被安装。

由于我们现在讨论的是前端项目，npm 只是用来安装工具链的，这些 package 在最后的前端代码里面都不需要，因此全部都应该用 `--save-dev`.


### Package Manager for frontend: Bower

Bower 是一个 Web 前端的 package manager, 可以通过 npm 安装。

```
npm install -g bower
```

Bower 的基本用法和 npm 是差不多的，有 `bower init`, `bower install`; install 同样有 `--save` 和 `--save-dev` 参数，但 `-g` 参数是没有的。 Bower 下载的组件默认存放在 `bower_components`.

许多 packages 的结构都是类似的，在 HTML 中可以用以下方式导入。

```html
<script src="bower_components/<package>/<package>.js"></script>
```

## Best Practice

上面讲的是 npm 和 bower 的基本使用方法。 npm 是 node.js 的 package manager, bower 是 Web 前端的一个 package manager.

按照上面的设定，所有文件都被放在了工程的根目录下，混在一起；等最后前端项目发布的时候，又没有 make install 这样的自动安装步骤。这就要求我们把这些文件分成源代码、中间文件、生成结果三部分来管理。源代码部分要纳入版本控制，生成结果要和其它分离，方便拷贝到发布位置。因此，这里推荐使用以下的目录结构。

```
.
├── dist/
├── node_modules/
├── src/
│   ├── bower_components/
│   ├── index.html
│   ├── main.js
│   └── site.css
├── bower.json
└── package.json
```

那么问题来了，怎么让 bower 把下载的 packages 存到 `src/bower_components/` 中？

```sh
cat > .bowerrc << EOF
{
  "directory": "src/bower_components"
}
EOF
```

在 .gitignore 里把 `dist/`, `node_modules/` 和 `bower_components/` 忽略掉，就可以让  npm, bower 和 git 一起愉快地玩耍了。

[nodejs]: https://nodejs.org/en/ "Node.js"
[nodejs-install]: https://nodejs.org/en/download/package-manager/ "Installing Node.js via package manager | Node.js"
