---
layout: post
title: "webpack 编译优化"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - Webpack
---

## 如果是多页项目，修改入口 `entry`，区分开发和生产环境

用我正在开发的项目举例，修改之前 `entry` 没有区分开发和生产环境，入口对象里包含 pages 目录下所有页面的入口 js

_webpack.base.conf.js_

    entry: utils.getEntry()

_utils.js_

    exports.getEntry = function () {
        var entry = {}
        Object.keys(config.pages).forEach(function (name) {
            entry[name] = config.pages[name].entry
        })
        return entry
    }

`config.pages` 是前面封装好的一个对象，存储着所有页面的信息，在这里交给 `Object.keys()` 方法去枚举它的属性并返回一个数组，再将该数组交给 `forEach` 循环遍历，循环里的 `name` 就是 `config.pages` 对象的 `key`。 `config.pages` 如下：

    {
        home: {
            name: 'home',
            entry: 'F:/path.../src/pages/home/main.js',
            template: 'F:/path.../src/pages/home/index.html'
        },
        'product-details': {
            name: 'product-details',
            entry: 'F:/path.../src/pages/product-details/main.js',
            template: 'F:/path.../src/pages/product-details/index.html'
        }
        ...
    }

修改之后：

_utils.js_

    exports.getEntry = function () {
        var entry = {}
        Object.keys(config.pages).forEach(function (name) {
            if (process.env.NODE_ENV === 'production') {
                entry[name] = config.pages[name].entry
            } else {
                var target_pages = !!JSON.parse(process.env.npm_config_argv).original[3] ? JSON.parse(process.env.npm_config_argv).original[3] : 'home'
                var target_pages_arr = target_pages.split('!')
                target_pages_arr.forEach(function (target_name) {
                    if (name === target_name) entry[name] = config.pages[name].entry
                })
            }
        })
        return entry
    }

首先通过 `process.env.NODE_ENV` 判断当前环境，如果是开发环境，就将所有页面的入口 js 添加进 `entry` （这里视你的情况而定，我开发的这个项目需要将所有的页面打包上线，所以 `entry` 里添加了所有页面的入口 js），如果是生产环境，则从执行的命令（`npm run dev -- pageName` | `npm run dev == pageNameA!pageNameB!pageNameC`）中获取当前正在开发的页面，再将这些页面的入口 js 添加进 `entry`，这样可以大大减少开发时每次编译所消耗的时间。

## 使用 HappyPack

webpack 构建时需要解析和处理大量的文件，但运行在 Node.js 上的 webpack 是单线程模型的，这就导致了 webpack 构建的时间开销比较大，项目越大问题越严重。HappyPack 可以帮我们分解任务并管理线程，它将任务分解给多个子进程并发执行，子进程处理完之后再将结果发送给主进程，我们只需要按照它的[用法和配置](https://github.com/amireh/happypack)正确接入即可。

_webpack.base.conf.js_

    const HappyPack = require('happypack')
    const happyThreadPool = HappyPack.ThreadPool({ size: 5 })

    module.exports = {
        module: {
            rules: [{
                test: /\.js$/,
                use: ['happypack/loader?id=babel'],
                include: [resolve('src')]
            }]
        },
        plugins: [
            new HappyPack({
                id: 'babel',
                loaders: ['babel-loader?cacheDirectory'],
                threadPool: happyThreadPool
            })
        ]
    }

- 对于 Loader，将文件的处理交给 `happypack/loader`，`happypack/loader`后面的 querystring -- `?id=babel` 会告诉它该选择插件中的哪个 HappyPack 实例来处理这些文件。

- 在插件中 `new` 了一个 HappyPack 的实例，作用是告诉 `happypack/loader` 怎么处理 js 后缀的文件， `id` 的值和 Loader 中的 `?id=babel` 对应， `loaders` 属性的值即原 Loader 配置，注意 `loaders` 属性需是数组类型。

- HappyPack 实例中的 `threadPool` 参数表示共享进程池，多个 HappyPack 实例都使用同一个共享进程池中的子进程去处理任务，可以防止资源占用过多，该例中构造出来的共享进程池 （`const happyThreadPool = HappyPack.ThreadPool({ size: 5 })`） 包含5个子进程。

配置完之后安装新依赖：

``` bash
npm i -D happypack
```

安装完之后重新进行构建即可。

构建时控制台会输出类似于下面的日志：

![happypack](/img/article/happypack.png)

这是 happypack 输出的，如果想关闭它，在实例化 HappyPack 插件时增加 `verbose` 参数并将它设置为 `false` 即可。

## 使用 DllPlugin 和 DllReferencePlugin

DllPlugin 和 DllReferencePlugin 的作用原理类似于 windows 系统中的动态链接库，即 .dll 后缀的文件。动态链接库中包含着供其它模块调用的函数和数据。

在 Web 项目中，可以将源代码中依赖的所有基础模块提取出来，统一打包到一个单独的动态链接库里，当需要导入的模块存在于这个提前生成的动态链接库里时，该模块就不能被再次打包，而是直接从动态链接库里面获取。

如果动态链接库里包含较多复用率比较高的模块,便会大大减少项目构建所需要的时间，因为这些模块被编译一次之后就不会再被重复编译，在以后的构建过程中用到这些模块的话就会直接从动态链接库中获取。

动态链接库中包含的大多是一些常用的第三方模块，如 vue、vuex 等，只要不升级这些模块的版本，就不用重新编译动态链接库。

_配置方法_

### 步骤1 创建配置文件 webpack.dll.config.js

首先新建一个配置文件 webpack.dll.config.js，该文件的位置视你的项目结构而定，最好和其它 webpack 配置文件放在同一目录下，该文件内容如下：

    const path = require("path")
    const webpack = require("webpack")

    module.exports = {
        entry: {
            vendor: [
                'iscroll',
                'mint-ui',
                'vue/dist/vue.common.js',
                'vue-iscroll-view',
                'vue-lazyload',
                'vue-scroller',
                'whatwg-fetch'
            ]
        },
        output: {
            path: path.join(__dirname, '../static/js'),
            filename: '[name].dll.js',
            library: '[name]_library'
        },
        plugins: [
            new webpack.DllPlugin({
                path: path.join(__dirname, '..', '[name]-manifest.json'),
                name: '[name]_library',
                context: __dirname
            })
        ]
    };

`entry.vendor` 是要放进动态链接库的模块的数组。

`output.path` 表示输出文件放置的目录。

`output.filename` 表示输出的动态链接库文件名称，`[name]` 代表当前动态链接库的名称，拿到的是 `entry` 配置项的 key，这里就是 `vendor`。也可以生成多个动态链接库，将模块按类型进行区分，比如可以将项目依赖的第三方模块 `vue`、`vuex` 等放入一个库中，将依赖的 polyfill 放入另外一个库中，这时 `entry` 的配置大致如下：

    entry: {
        vue: [
            'mint-ui',
            'vue/dist/vue.common.js',
            ...
        ],
        polyfill: [
            'whatwg-fetch',
            ...
        ]
    }

`output.library` 用于存放动态链接库的全局变量名称，比如对于 `vendor` 这个库来说名称就是 `vendor_library`。

实例化 `webpack.DllPlugin` 时传递的参数 `path` 表示描述动态链接库的 *-manifest.json 文件输出的路径。

参数 `name` 表示动态链接库的全局变量名称，需和 `output.library` 的值保持一致。该字段的值也是输出的 *-manifest.json 中 `name` 字段的值。

参数 `context` 是 manifest 文件中请求的上下文，选填，默认是当前 webpack 文件的上下文。

### 步骤2 生成动态链接库

配置好 webpack.dll.config.js 之后执行命令 `webpack --config build/webpack.dll.config.js` 就可以构建动态链接库文件了，我们将这句命令加到 package.json 的 scripts 里：

    "scripts": {
        "build:dll": "webpack --config build/webpack.dll.config.js"
    }

执行 `npm run build:dll` 即可。

执行完之后就会生成 vendor.dll.js 和  vendor-manifest.json，vendor.dll.js 在 static 目录下的 js 文件夹里，它就是“动态链接库文件”，vendor-manifest.json 在项目的根目录下。

### 步骤3 引用动态链接库

配置 DllReferencePlugin 插件，使用 vendor-manifest.json 来引用动态链接库。

_webpack.base.conf.js_

    plugins: [
        ...
        new webpack.DllReferencePlugin({
            context: __dirname,
            manifest: require('../vendor-manifest.json')
        })
    ]

`context` 需与 Dllplugin 里 `context` 参数所指向的上下文保持一致。
`manifest` 参数用来引入之前生成的 vendor-manifest.json。

### 步骤4 页面模板文件引入动态链接库

在页面的模板文件里手动引入动态链接库文件。

    <script src="./static/js/vendor.dll.js"></script>

到这一步所有配置就已完成，根据动态链接库中包含的模块数量和复用率不同，构建所减少的时间也会有所不同。

_PS1_

如果动态链接库所包含的模块有所变化，就需要重新构建生成一份动态链接库文件。

_PS2_

生成的 js 文件是未经压缩的，可以在 webpack.dll.config.js 中添加插件将生成的动态链接库文件进行压缩：

    plugins: [
        ...
        new webpack.optimize.UglifyJsPlugin({
            compress: {
            warnings: false
            }
        })
    ]


## babel-loader 使用 cacheDirectory

`'babel-loader?cacheDirectory'`

用于缓存 babel 的编译结果，加快重新编译的速度。

## 缩小文件的查找范围

    module: {
        rules: [{
            test: /\.js$/,
            loader: 'babel-loader?cacheDirectory',
            include: path.resolve(__dirname, '../src')
        }]
    }

include 也可以是数组，路径视具体项目结构而定。用于命中指定目录下的文件，加快 webpack 的搜索速度。