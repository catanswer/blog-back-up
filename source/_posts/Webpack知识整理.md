---
title: Webpack知识整理
date: 2019-07-17 09:15:11
tags: Webpack
categories: 
  - Web前端
---

<!-- # Webpack知识整理 -->

 <!-- ## Webpack的定 -->
  webpack是一个基于nodejs开发的模块打包工具
  一开始， webpack只是一个js打包工具，在之后的发展中，它还可以对css、图片等文件进行打包
<!-- more -->

## Webpack安装及基本使用
#### 安装Nodejs
  在[Nodejs官网](https://nodejs.org/en/)下载安装包（建议下载左侧的稳定版本）进行安装。
  
  安装完成后，打开控制台输入一下命令，查看是否安装成功
  ```dash
    node -v
    npm -v
  ```

#### 创建使用Webpack的项目
  * 初始化项目
    -y：以默认的设置直接初始化，无需手动敲回车

    ``` bash
      npm init -y
    ```
  * 安装webpack
    全局安装（不推荐）
    ```bash
      npm i webpack webpack-cli -g
    ```
    项目中（局部）安装
    ```bash
      npm i webpack webpack-cli -D
      npm i webpack webpack-cli --save-dev

      // 查看是否成功
      // npx会在当前项目的目录中寻找webpack
      npx webpack -v
    ```

#### webpack配置文件
  * 在项目根目录下创建`webpack.config.js`（默认名称）文件, 如：
    ```javascript
      const path = require('path')

      module.exports = {
        mode: 'development',
        // 入口文件的路径
        entry: './src/index.js',
        output: {
          // 输出文件的名称
          filename: 'bundle.js',
          // 输出文件的路径
          path: path.resolve(__dirname, 'dist')
        }
        ...
      }
    ```
  * 在`package.json`文件中配置`script`, 如：
    ```json
      ...
      "scripts": {
        "build": "webpack"
      }
      ...
    ```
  这样只需要在命令行输入`npm run build`就会自动寻找到在根目录下的`webpack.config.js`配置文件并进行打包

## Config配置
#### entry、output
  ```javascript
    ...
    entry: {
      // 配置多个入口文件
      main: './src/main.js',
      sub: './src/sub.js'
    },
    output: {
      // 使用占位符表示输出文件的名字
      filename: '[name].js',
      // 如果js文件不是被直接应用的，输出的名字就会是chunkFilename的值
      chunkFilename: '[name].chunk.js',
      path: path.resolve(__dirname, 'dist'),
      // 在引入打包后的js文件前添加的公共路径
      publicPath: 'http://cdn.com.cn',
      // 将打包生成的代码，挂在到全局变量'library'上
      library: 'library',
      // 打包代码的模式。一般用在库打包时，打包后可以以不同方式进行引入使用
      libraryTarget: 'umd'
    }
    ...
  ```
#### Devtool: source-map
  [source-map](https://webpack.js.org/configuration/devtool)它是一个映射关系，它知道`dist`目录下打包后的文件实际对应`src`目录下文件的具体位置。(源代码和目标生成代码之间的映射关系)
  ```javascript
    ...
    // 会产生.map文件
    devtool: 'source-map'
    ...
  ```
  - `xxx-inline-xxx`：不会产生`.map`文件，将映射的数据直接添加进打包后的js文件
  - `xxx-cheap-xxx`: 提示错误只会精确到行，不会到列，且只涉及业务代码
  - `xxx-module-xxx`: 映射的范围包括了loaders、第三方模块等
  - `xxx-eval-xxx`：效果和`inline`一样，执行效率最快
  - **实践推荐**：
    - 开发环境：`cheap-module-eval-source-map`
    - 生产环境：`cheap-module-source-map`

#### DevServer
  [DevServer](https://webpack.js.org/configuration/dev-server)开发环境下，自动打包文件，并刷新浏览器，需单独安装使用`npm i webpack-dev-server -D`
  - `contentBase`：告诉服务器从哪里提供内容。只有在你想要提供静态文件时才需要
  - `open`：自动在浏览器中打开启动的服务地址
  - `port`：默认`8080`，更改服务端口号
  - `hot`：热更新，在`plugins`配置中还需使用`HotModuleReplacementPlugin`，实现局部更新。
    ```javascript
      plugins: [
        new webpack.HotModuleReplacementPlugin()
      ]
    ```
    - js：js文件的更改不会自动更新，需要在代码中自己监听文件的变化：
      ```javascript
        if (module.hot) {
          module.hot.accept('./xx.js', () => {
            // 监听到文件变化后执行的回调函数
          })
        }
      ```
    - css: css文件不需要自己去监听，`css-loader`帮我们实现了。
  - `hotOnly`：在项目打包失败时，不刷新页面
  - `proxy`: 代理请求，解决开发时跨域问题
  
#### Tree-Shaking
  自动剔除多余代码。只支持对`import`引入的代码进行‘瘦身’。
  - 在`development`环境下，默认不开启，如需使用，如下：
    ```javascript
      entry: { ... },
      optimization: {
        usedExports: true
      }
      output: { ... }
    ```
    但是由于配置了`usedExports`属性，所以只打包`使用过的的导出模块`，但是在使用`@babel/polyfill`对代码进行转换时，处理兼容性的代码是添加在全局或全局对象上的，并没有导出模块，但被使用了，这时该配置也会把对应的代码剔除掉，很明显不符合我们的预期。可以使用以下方法来解决这个问题：
      ```json
        // package.json
        {
          // 在以下属性中配置不需要剔除的代码
          "sideEffects": ["*.css", "@babel/polyfill"]
        }
      ```
    > 在**development**环境下打包后的文件中，还是会有未使用的代码，这是为了方便调试，以免代码行号的错误。如需看到真正的效果，只需让mode更改为**production**，此时无需再配置optimization的usedExports属性

## Loader
  webpack可以使用loader来预处理文件，这允许你打包除了js之外的任何静态资源，[官方推荐loader](https://webpack.js.org/loaders)。
  > loader执行的特点： **从上到下，从左到右的顺序执行**

  基本的使用结构如下:
  ```javascript
    entry: { ... },
    module: {
      rules: [
        {
          test: /\.png$/,
          use: 'file-loader'
        }
      ]
    },
    output: { ... }
  ```
#### 静态资源打包-图片
- [file-loader](https://webpack.js.org/loaders/file-loader)：可以用来处理图片、字体资源
  ```javascript
    module: {
      rules: [
        {
          test: /\.(png|jpe?g|gif)$/,
          use: {
            loader: 'file-loader',
            options: {
              // 使用placeholder占位符
              name: '[name]_[hash:6].[ext]',
              outputPath: 'images'
            }
          }
        }
      ]
    }
  ```
  在`options`配置项中可以设置：
    - `name`：资源名称
    - `outputPath`：输出路径
    - `publicPath`：自定义公共路径
    - `placeholder的使用`：[占位符](https://webpack.js.org/loaders/file-loader#placeholders)
  具体参考，请点击[这里](https://webpack.js.org/loaders/file-loader#options)

- [url-loader](https://webpack.js.org/loaders/url-loader)：也可以用来打包图片资源，功能和`file-loader`基本一致，只是增加了如下配置功能：
  ```javascript
    ...
    options: {
      // 图片资源大小小于该值，使用base64压缩，否则像file-loader一样单独打包
      limit: 2048
    }
    ...
  ```

#### 静态资源打包-CSS
  - [style-loader](https://webpack.js.org/loaders/style-loader)：将css添加到页面的`<style>`标签中，一般会配合`css-loader`使用。
  常用的配置如下：
    ```javascript
      ...
      module.exports = {
        module: {
          rules: [{
            test: /\.css$/,
            use: [
              {
                loader: 'css-loader',
                options: {
                  // 启用禁用CSS模块使css作用于局部文件
                  modules: true,
                  // 在 css-loader 前应用的 loader 的数量
                  // 避免嵌套引入的css在css-loader之前未得到处理
                  importLoaders: 2
                }
              }
            ]
          }]
        }
      }
      ...
    ```

  - [css-loader](https://webpack.js.org/loaders/css-loader)：处理css文件之间的引用关系，会在`import/require()`后再去解析。

  - [sass-loader](https://webpack.js.org/loaders/sass-loader)：css预处理器的loader，先将`.scss`文件处理成css然后再交给`css-loader`处理。需要注意的是，在使用的时候不仅仅需要安装`sass-loader`，还需要安装`node-sass`
    ```dash
      npm i sass-loader node-sass -D
    ```
  - [postcss-loader](https://webpack.js.org/loaders/postcss-loader)：使用[postcss](https://postcss.org/)处理css资源。常用的有自动补齐前缀（autoprefixer）等。
    ```javascript
      // post.config.js
      module.exports = {
        plugins: [
          require('autoprefixer')
        ]
      }
    ```
  
#### 使用Babel处理ES6语法
  - `babel`的具体使用方法，可以[参考官网](https://babeljs.io/setup#installation)
  - [babel-loader](https://webpack.js.org/loaders/babel-loader)：
    >`babel-loader`是`webpack`和`babel`之间通信的桥梁，并不会转换ES6的语法
    >如果需要转化ES6的语法，还需要其他模块，比如：`@babel/preset-env`等
  - `@babel/preset-env`：包含了转化ES6语法的方法。
    >但不会转换新的API，比如Iterator, Generator, Set, Maps, Proxy, Reflect, Symbol, Promise等全局对象。以及一些在全局对象上的方法(比如 Object.assign)都不会转码
  - `@babel/polyfill`: 为了解决新的API和上面全局对象或全局对象方法不能被转换的问题。
    <!-- **业务开发时，使用`@babel/preset-env`和`@babel/polyfill`** -->
    - **原理**：当运行环境中没有实现的一些方法时，通过全局对象和内置对象的prototype上添加方法来实现。
    - **使用**：在需要使用的模块中`import '@babel/polyfill'`
    - **缺点**：补充的部分会以全局变量的方式引入，会污染全局变量
    ```javascript
      module: {
        rules: [{
          test: /\.js$/,
          // 排除node_modules目录下的js文件
          exclude: /node_modules/,
          loader: 'babel-loader',
          options: {
            // 在编写'业务代码'的时候只需要设置预设配置进行转化
            presets: [
              ['@babel/preset-env', {
                // 需要兼容的浏览器列表配置
                targets: {
                  edge: '17',
                  firefox: '60',
                  chrome: '67',
                  safari: '11.1'
                },
                // 在处理低版本浏览器上兼容性时，按需加载你使用的特性，打包时减小文件大小
                useBuiltIns: 'usage'
              }]
            ]
          }
        }]
      }
    ```
    <!-- **开发类库时，使用以下两个插件** -->
  - `@babel/runtime`
    - **原理**：它是将ES6编译成ES5去执行。不管浏览器是否支持ES6，只要是ES6的语法，都会进行转换成ES5。它不会污染全局对象和内置对象的原型。比如现在需要Promise，只需要`import Promise from 'babel-runtime/core-js/promise'`引入即可使用
    - **缺点**：会产生冗余的代码；在使用时需要一个文件一个文件的引入需要的模块（`@babel-transform-runtime`就是解决这个问题的）
  - `@babel/plugin-transform-runtime`
    - **原理**：避免手动引入模块的烦恼，并抽离了公用方法
    ```javascript
      module: {
        rules: [
          {
            test: /\.js$/,
            exclude: /node_modules/,
            loader: 'babel-loader',
            options: {
              // runtime会以闭包的形式注入（引入）相应的内容，避免全局污染
              "plugins": [[
                "@babel/plugin-transform-runtime", {
                  "absoluteRuntime": false,
                  // 这里需要安装@babel/runtime-corejs2
                  "corejs": 2,
                  // 表示是否把内置的东西(Promise, Set, Map)等转换成非全局污染的
                  "helpers": true,
                  // 是否开启generator函数转换成使用regenerator runtime来避免污染全局域
                  "regenerator": true,
                  "useESModules": false 
                }
              ]]
            }
          }
        ]
      }
    ```
      >相关的`babel`配置，除了能直接写在`babel-loader`的`options`中外，还能抽离配置属性到`.babelrc`文件中，让配置更加的清晰明了

#### 其他一些功能的Loader
 - [imports-loader](https://www.webpackjs.com/loaders/imports-loader/)
  可以通过这个loader向模块中插入一些变量等功能，具体点击链接查看


## Plugins
#### html-webpack-plugin
  [html-webpack-plugin](https://webpack.js.org/plugins/html-webpack-plugin)：在打包结束后，自动生成一个html文件，并把打包生成的js自动引入到这个html文件中。
  ```javascript
    plugins: [
      new HtmlWebpackPlugin({
        filename: 'index.html',
        template: './src/index.html'
      })
    ]
  ```
#### clean-webpack-plugin
  [clean-webpack-plugin](https://github.com/johnagan/clean-webpack-plugin)：在打包之前，清理之前打包的文件目录

#### split-chunks-plugin
  [split-chunks-plugin](https://webpack.js.org/plugins/split-chunks-plugin/)配置项的介绍：
  ```javascript
    optimization: {
      splitChunks: {
        // 代码分割时，只对异步代码进行分割 可选值：all(同步+异步) initial(同步)
        chunks: 'async',
        // 生成的公共模块的最小size，单位是bytes
        minSize: 30000,
        // 对大于该值且大于minSize值的chunks进行更小的分割
        maxSize: 0,
        // 代码切割之前的最小共用数量
        minChunks: 1,
        // 按需载入的最大请求数。超过该值后，不做代码分割
        maxAsyncRequests: 5,
        // 单个入口文件的最大请求数。超过该值后，不做代码分割
        maxInitialRequests: 3,
        // 指定用于生成名称的分隔符
        automaticNameDelimiter: '~',
        // 自定义分割后代码的名称
        name: true,
        // 会继承splitChunks.*的配置 但test，priority和reuseExistingChunk只能在缓存组中配置
        // 对于同步引入的代码，还会使用cacheGroups的配置
        cacheGroups: {
          // 缓存组：vendors
          vendors: {
            // 符合该组的条件
            test: /[\\/]node_modules[\\/]/,
            // 优先级更高的缓存组才具有打包的资格
            priority: -10,
            // 分割后代码的名字 同步引入代码分割是才有效
            filename: 'vendors.js'
          },
          // 缓存组：默认 可设置为false
          default:  {
            minChunks: 2,
            priority: -20,
            // 当前chunk包含的模块已经被主bundle打包进去，将不再被打包进当前chunk
            reuseExistingChunk: true,
            filename: 'common.js'
          }
        }
      }
    }
  ```

#### mini-css-extract-plugin
  [mini-css-extract-plugin](https://webpack.js.org/plugins/mini-css-extract-plugin)分割css，把css文件剥离出作为一个单独的文件引入最后打包后的html文件。建议在`生成环境`中配置该属性。
  - **安装**：
    ```dash
      npm i mini-css-extract-plugin -D
    ```
  - **使用**：
    ```javascript
      const MiniCssExtractPlugin = require('mini-css-extract-plugin')
      ...
      module: {
        rules: [
          {
            test: /\.css$/,
            use: [
              {
                loader: MiniCssExtractPlugin.loader,
                options: {
                  // 该配置在HRM（热加载）失败时才有效
                  reloadAll: true,
                }
              },
              'css-loader',
              'postcss-loader'
            ]
          }
        ]
      },
      plugins: [
        new MiniCssExtractPlugin({
          filename: '[name].css',
          // 该属性和output中chunkFilename属性是一样的作用
          chunkFilename: '[name].chunk.css'
        })
      ]
      ...
    ```
    > **注意点**：当配置了上述属性之后，发现没有效果（分割css），可能是在`optimization`属性配置了`usedExports: true`（Tree Shaking），使用import引入的文件如果未使用不会进行打包。在package.json文件中配置："sideEffects: ["*.css"]"
  
  由于分割出来的文件中的css未被压缩处理过，所以就需要用到下面`optimize-css-assets-webpack-plugin`插件

#### optimize-css-assets-webpack-plugin
  [optimize-css-assets-webpack-plugin](https://github.com/NMFR/optimize-css-assets-webpack-plugin)：压缩css文件内容
  - **安装**：
    ```dash
      npm i terser-webpack-plugin optimize-css-assets-webpack-plugin -D
    ```
  - **使用**
    ```javascript
      const TerserJSPlugin = require('terser-webpack-plugin');
      const OptimizeCSSAssetsPlugin = require('optimize-css-assets-webpack-plugin')
      ...
      optimization: {
        minimizer: [
          new TerserJSPlugin({}),
          new OptimizeCSSAssetsPlugin({})
        ]
      }
      ...
    ```
    > 由于重写了optimization的配置项，所以原来默认的压缩js代码的配置也失效了，需重新安装配置[uglifyjs-webpack-plugin](https://webpack.js.org/plugins/uglifyjs-webpack-plugin/)

#### webpack ProvidePlugin
  [webpack.ProvidePlugin](https://www.webpackjs.com/plugins/provide-plugin/)：自动加载模块，而不必到处 import 或 require
  - 使用
    ```js
      ...
      const webpack = require('webpack')
      ...
      plugins: [
        // 这样配置后，在模块中都可以使用jquery
        new webpack.ProvidePlugin({
          $: 'jquery',
          jQuery: 'jquery'
        })
      ]
    ```


#### add-asset-html-webpack-plugin
  [add-asset-html-webpack-plugin](https://github.com/SimenB/add-asset-html-webpack-plugin)：在打包生成的html文件中添加静态资源
  ```js
    const AddAssetWebpackPlugin = require('add-asset-html-webpack');
    // webpack.config.js
    ...
    plugins: [
      new AddAssetWebpackPlugin({
        filepath: path.resolve(__dirname, '../dll/vendor.dll.js')
      })
    ]
  ```
  


## 基本使用
#### Development和Production模式区分打包
  ```dash
    // 安装webpack合并模块
    npm i webpack-merge -D
  ```
  生产环境下的配置：`webpack.prod.js`和开发环境下的配置：`webpack.dev.js`，将它俩共用的代码部分提取到`webpack.common.js`，然后在各自的代码中与之合并

#### Code Splitting-JS代码分割
  - **存在的问题**：在我们在项目中使用类似`lodash`的第三方库进行打包时，会把第三方库的代码也进行打包，影响项目的性能，这是不合理的。
  - **解决方案**
    - 多个入口文件：将第三方模块的代码拆分成多个js入口文件在项目中引入。当业务逻辑发生变化时，只要加载相应代码，第三方模块的代码不需要重新加载
    - **code spliting**：
      同步加载代码：
      ```javascript
        ...
        // 使用import导入，同步加载的代码
        optimization: {
          splitChunks: {
            chunks: 'all'
          }
        }
        ...
      ```
      异步加载代码：`split-chunks-plugin`就只对异步的代码做分割
      ```javascript
        function getComponent () {
          // 使用这种方式异步加载代码，由于是实验性的写法，需要安装@babel/plugin-syntax-dynamic-import
          return import('lodash').then(({ default: _ }) => {
            var element = document.createElement('div')
            element.innerHTML = _.join(['ysh', 'lg'], '-')
            return element
          })
        }
        getComponent().then((element) => {
          document.body.appendChild(element)
        })
      ```
  - **代码分割底层**：通过配置`splitChunks`属性来进行代码的分割，其底层是使用了`split-chunks-plugin`，对于这个插件的具体介绍，请看`split-chunks-plugin`部分
  - **Lazy Loading懒加载**
    通过异步使用`import`引入的方式来加载代码，称为懒加载。这种方式让页面加载速度更快
    举个例子：
    ```javascript
      async function getComponent () {
        const { default: _ } = await import(/*webpackChunkName:"lodash"*/'lodash');
        var element = document.createElement('div')
        element.innerHTML = _.join(['ysh', 'lg'], '-')
        return element
      }

      document.addEventListener('click', () => {
        // 点击事件发生后，才会加载 lodash
        getComponent().then((element) => {
          document.body.appendChild(element)
        })
      })
    ```
  - **Chunk是什么**
    每一个被打包的js文件都是一个chunk。

#### Code Splitting-CSS代码分割
  详细介绍请看`Plugins`章节的`mini-css-extract-plugin`

#### 打包分析
  - [官方工具](https://github.com/webpack/analyse)（需科学上网）
    - 在`package.json`文件中的`script`属性中配置一个命令，该命令的内容是`webpack --profile --json > stats.json`。
    - 执行该命令后，会在项目根目录中生成一个`stats.json`文件。该文件包含项目打包的所有信息。
    - [点击这里](https://webpack.github.io/analyse/)（科学上网），上传生成的`stats.json`文件
    - 查看打包相关信息
  - **[其他工具](https://webpack.js.org/guides/code-splitting#bundle-analysis)**
    推荐[webpack-bundle-analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer)这个工具。

#### Preloading和Prefetching
  [Preloading和Prefetching](https://webpack.js.org/guides/code-splitting#prefetchingpreloading-modules)
  ```javascript
    // 利用魔法注释方式来使用
    import(/* webpackPrefetch: true */ 'LoginModal');
  ```
  - webpackPrefetch：等待主的文件加载完成后，才去加载
  - webpackPreload：会和主的文件一起加载（不推荐）

#### Webpack与浏览器缓存
  - **问题**：在项目打包后，上线到服务器，用户第一次访问网站，浏览器会缓存网站上的资源；如果这个时候网站内容（例如某个js文件）发生更改，重新打包上传服务器，此时用户再次访问，由于浏览器本地已有缓存，则不会看到修改后的内容。
  - **解决**：项目打包时，在`output`属性的`filename`和`chunkfilename`属性中添加`[contenthash]`的占位符，这样打包后的文件就会带上哈希值，文件内容是否修改决定了该哈希值是否变化。
    ```js
      ...
      output: {
        filename: '[name].[contenthash:8].js',
        chunkfilename: '[name].[contenthash:8].chunk.js'
      }
      ...
    ```

#### 环境变量的配置
  一般区分打包环境有两种方式
  - 以文件区分
    在`build`目录下，创建`webpack.dev.js`和`webpack.prod.js`分别作为开发环境和生产环境的配置文件，提取通用的配置到`webpack.common.js`，在`package.json`中配置命令，在不同命令中根据不同环境分别指定不同的配置文件。
  - 以环境变量区分
    步骤基本和上面方式一样，不同的是在配置运行命令时，会加上环境参数，比如`build: webpack --env.production --config webpack.common.js`。开发和生产环境的配置最终都在`webpack.common.js`中根据环境变量来决定最终的打包配置。

## 实际应用
#### Library(库)的打包
  详细请参考[这里](https://www.webpackjs.com/guides/author-libraries/)
  ```js
    module.exports = {
      mode: 'production',
      entry: {
        main: './index.js'
      },
      // 外部化lodash，剥离了不需要改动的依赖模块，当用户使用该库时，应自行安装好
      // library 需要一个名为 lodash 的依赖，这个依赖在用户的环境中必须存在且可用
      externals: {
        lodash: {
          // 在不同环境下，引入的名称
          commonjs: 'lodash',
          commonjs2: 'lodash',
          amd: 'lodash',
          root: '_'
        }
      },
      output: {
        filename: 'library.js',
        // 将打包代码注入到变量的名称，具体注入到哪里，根据'libraryTarget'的值决定
        library: 'library',
        // 打包代码的模式。一般用在库打包时，打包后可以以不同方式进行引入使用
        // 默认 var：将打包后的代码赋值给变量'library'
        // this、window、global：通过对象上赋值暴露
        // commonjs2：暴露为CommonJS模块，此时'library'属性可以不配置
        // amd：暴露为AMD模块
        // umd：暴露为所有的模块定义下都可运行的方式
        libraryTarget: 'umd'
      }
    }
  ```
  ```json
    // package.json
    ...
    { 
      // 添加bundle的文件路径
      "main": "./dist/library.js",
    }
    ...
  ```
  申请[npm官网](https://www.npmjs.com/)账号，然后在项目命令行中运行`npm adduser`登录账号,然后运行`npm publish`可以发布到npm官方库上
  
#### PWA的打包配置
  > [PWA](https://www.webpackjs.com/guides/progressive-web-application/)（Progressive Web App）是一种理念，使用多种技术来增强web app的功能，可以让网站的体验变得更好，能够模拟一些原生功能，比如通知推送。在移动端利用标准化框架，让网页应用呈现和原生应用相似的体验
  - **安装**
    ```dash
      // 安装workbox-webpack-plguin
      npm i workbox-webpack-plguin
    ```
  - **使用**：在服务器出现问题是，用户也能访问到之前缓存在浏览器的内容
    ```js
      // webpack.prod.js 只需在生产环境下使用
      import WorkboxPlugin = require('workbox-webpack-plugin')
      ...
      plugins: [
        new WorkboxPlugin.GenerateSw({
          clientsClaim: true,
          skipWaiting: true
        })
      ]
      ...
    ```
    ```js
      // index.js
      // 判断浏览器是否支持serviceworker
      if ('serviceworker' in navigator) {
        window.addEventListener('load', ()=> {
          // 注册serviceworker
          navigator.serviceWorker.register('/service-work.js').then((registration) => {
            console.log('service-worker registed');
          }).catch(error => {
            console.log('service-worker register error');
          })
        })
      }
    ```


#### TypeScript的打包配置
  `TypeScript`是JavaScript的一个超集。
  - **安装**：
    ```dash
      npm i typescript ts-loader -D
    ```
  - **使用**
    ```js
      // webpack.config.js
      ...
      module: {
        rules: [
          {
            test: /\..ts$/,
            loader: 'ts-loader'
          }
        ]
      }
      ...
    ```
    除了使用`ts-loader`转换代码之外，还需要在项目根目录下创建`tsconfig.json`的文件
    ```json
      // 一些关于ts的配置
      {
        "compilerOptions": {
          // 输出的文件目录
          "outDir": "./dist",
          // 引入模块的方式
          "module": "es6",
          // 最终转换的代码的格式
          "target": "es5",
          // 是否允许在ts文件中引入js的模块
          "allowJs": true
        }
      }
    ```
    在`typescript`中使用其他第三方库时，也具有静态类型检测的功能，还需要安装对应的类型库。具体类型库，点击[这里](https://microsoft.github.io/TypeSearch/)
    ```dash
      npm i @types/lodash -D
    ```
    在安装完对应的类型库时，在使用的文件中引入时，只能是以下面这种方式：
    ```js
      import * as _ from 'lodash'
    ```
    

#### DevServer：实现请求转发、单页面路由
  ```js
    // webpack.config.js
    ...
    devServer: {
      // index设置为空时，可以代理根路径
      index: '',
      // 当使用 HTML5 History API 时，任意的 404 响应被替代为 index.html
      // historyApiFallback: true
      historyApiFallback: {
        rewrite: [{
          from: /^\/$/,
          to: '/views/landing'
        }]
      },
      proxy: {
        '/api': {
          // 转发的目标地址
          target: 'htpp://localhost:3000',
          // 对https网址请求时配置
          secure: false,
          pathRewrite: {
            // 去掉'/api'路径
            '^/api': '',
            '真实的路径': '暂时的路径'
          },
          // 可以访问到虚拟主机下的内容
          changeOrigin: true,
          // 不想代理全部的请求时
          bypass: function(req, res, proxyOptions) {
            // 当请求的是一个html时，不代理
            if (req.headers.accept.indexOf("html") !== -1) {
              console.log("Skipping proxy for browser request.");
              return "/index.html";
            }
          }
        }
      }
      
      // 多个路径时
      proxy: [{
        context: ['/auth', './api'],
        target: 'htpp://localhost:3000'
      }]
    }
    ...
  ```

#### 在Webpack中Eslint的配置
  详细请参考[这里](https://webpack.js.org/loaders/eslint-loader/)
  - **安装**
    ```dash
      npm i eslint-loader -D
    ```
  - **使用**
    ```js
      // webpack.config.js
      ...
      // 在浏览器上显示报错层，迅速确定错误位置
      overlay: true,
      module: {
        rules: [
          {
            test: /\.js$/,
            exclude: /node_modules/,
            use: ['babel-loader', {
              loader: 'eslint-loader',
              options: {
                // 自动修复一些简单的错误
                fix: true,
                cache: true
              },
              // 强制优先执行
              force: 'pre'
            }]
          }
        ]
      }
      ...
    ```

#### 多页面打包配置
  - **使用**
    ```js
      // webpack.config.js
      module.exports = {
        entry: {
          index: './src/index.js',
          sub: './src/sub.js'
        },
        ...
        plugins: [
          new HtmlWebpackPlugin({
            template: './src/index.html',
            filename: 'index.html',
            chunks: ['runtime', 'vendors', 'index']
          }),
          new HtmlWebpackPlugin({
            template: './src/index.html',
            filename: 'sub.html',
            chunks: ['sub']
          })
        ]
      }
    ```
  - **优化**
    ```js
      const makePlugins = (config) => {
        const plugins = [
          new CleanWebpackPlugin()
        ];
        // 根据entry字段，在plugins中添加多个插件实例及其配置属性
        Object.keys(config.entry).forEach(item => {
          plugins.push(new HtmlWebpackPlugin({
            template: './src/index.html',
            filename: `${item}.html`,
            chunks: ['runtime', 'vendors', item]
          }))
        });
        // 根据dll目录下文件类型来添加相应的插件实例
        const files = fs.readdirSync(path.resolve(__dirname, '../dll'))
        files.forEach(file => {
          if (/.*\.dll.js/.test(file)) {
            plugins.push(new AddAssetHtmlWebpackPlugin({
              filepath: path.resolve(__dirname, '../dll', file)
            }))
          }
          if (/.*\.manifest.json/.test(file)) {
            plugins.push(new webpack.DllReferencePlugin({
              manifest: path.resolve(__dirname, '../dll', file)
            }))
          }
        });
        return plugins;
      };
      const config = {
        ...
      };
      config.plugins = makePlugins(config);
      module.exports = config
    ```

#### 性能优化-提升打包速度
  - 跟上技术的迭代（Node, Npm, Yarn）
  - 在尽可能少的模块上应用Loader
    在使用loader时，明确代码的范围
    ```js
      {
        test: /\.js$/,
        // 排除
        exclude: /node_module/,
        // 包含
        include: path.resolve(__dirname, '../src'),
        use: ['babel-loader']
      }
    ```
  - Plugin尽可能精简并确保可靠
  - 合理配置resolve参数
    ```js
      // webpack.config.js
      module.exports = {
        resolve: {
          // 配置别名
          alias: {
            tool: path.resolve(__dirname, '../src/tool')
          },
          // 引入模块时，省略后缀，按照以下顺序依次匹配
          extensions: ['.js', '.jsx'],
          // 引入模块时，如果只写路径，则默认按照以下名称匹配
          // 一般不配置
          mainFiles: ['index', 'main', 'child']
        }
      }
    ```
  - 使用DllPlugin提高打包速度
    使用的第三方模块只打包一次，然后使用时，引入的是提前打包好的第三方模块（而不是在node_module中去引入）
    - **使用**
      创建一个独立的webpack配置文件，专门用来打包使用到的第三方模块，如下
      ```js
        // webpack.dll.js
        const path = require('path')
        const webpack = require('wepack')

        module.exports = {
          entry: ['react', 'react-dom', 'lodash'],
          output: {
            filename: '[name].dll.js',
            path: path.resolve(__dirname, '../dll'),
            // 打包好的文件，以变量的形式暴露给全局
            library: '[name]'
          },
          plugins: [
            new webpack.DllPlugin({
              name: '[name]',
              // 生成一个`manifest.json`文件，该文件用来让`DLLReferencePlugin`映射到相关的依赖上去
              path: path.resolve(__dirname, '../dll/[name].manifest.json') 
            })
          ]
        }
      ```
      在webpack主配置文件中，配置`DllReferencePlugin`，这样以后的每次打包，第三方模块就不会每次被打包，影响打包速度了。
      ```js
        // webpack.config.js
        ... 
        plugins: [
          new webpack.DllReferencePlugin({
            manifest: path.resolve(__dirname, '../dll/vendors.manifest.json')
          })
        ]
      ```
    - **改进**
      ```js
        // webpack.config.js
        const plugins = [
          new HtmlWebpackPlugin({
            template: './src/index.html',
            filename: 'index.html'
          }),
          new CleanWebpackPlugin()
        ]
        // 判断dll目录下的文件，将他们自动加入到plugins中
        const files = fs.readdirSync(path.resolve(__dirname, '../dll'))
        files.forEach(file => {
          if (/.*\.dll.js/.test(file)) {
            plugins.push(new AddAssetHtmlWebpackPlugin({
              filepath: path.resolve(__dirname, '../dll', file)
            }))
          }
          if (/.*\.manifest.json/.test(file)) {
            plugins.push(new webpack.DllReferencePlugin({
              manifest: path.resolve(__dirname, '../dll', file)
            }))
          }
        })
      ```
  - 控制包文件大小：对文件进行拆分；使用tree-shaking去除冗余代码
  - thread-loader、parallel-webpack（多页面打包）、happypack多进程打包
  - 合理使用sourceMap
  - 结合stats分析打包结果，详情参考打包分析章节内容
  - 开发环境内存编译
  - 开发环境剔除无用插件：开发环境下无需对代码进行压缩


## 自定义实现
#### 编写Loader
  一些API，可以点击[这里](https://www.webpackjs.com/api/loaders/#this-query)查看
  ```js
    // replaceLoader.js
    const loaderUtils = require('loader-utils')
    // 返回的函数不能是箭头函数，在函数体内需要使用到this的指向
    module.exports = function (source) {
      // this.query能拿到options中传过来的参数（已废弃）
      // 使用 loader-utils 中的 getOptions 方法来提取给定 loader 的 option。
      const options = loaderUtils.getOptions(this)
      const result =  source.replace('ysh', options.name)
      
      // this.callback除了能返回处理后的源文件之外，还能携带sourcemap及其他数据
      // this.callback(null, result)
      
      // 在loader中需要处理异步代码时
      // this.async返回的就是this.callback
      const callback = this.async()
      setTimeout(() => {
        callabck(null, result)
      }, 1000)
    }

    // webpack.config.js
    module.exports = {
      ...
      // 加载自定义loader的路径
      resolveLoader: {
        modules: ['node_modules', './loaders']
      },
      module: {
        rules: [
          {
            test: /\.js$/,
            use: [
              {
                loader: path.resolve(__dirname, './loaders/replaceLoader.js'),
                options: {
                  name: 'hello'
                }
              },
              {
                // 这样简写自定义的loader还需配置resolveLoader属性
                loader: 'replaceLoaderAsync',
                options: {
                  name: 'world'
                }
              }
            ]
          }
        ]
      }
      ...
    }
  ```

#### 编写Plugin
  **区别**
  - Loader：是用来处理模块的
  - Plugins：在进行打包过程中的某个具体的时间点上，进行一些操作
  ```js
    // copyright-webpack-plugin.js
    class CopyrightWebpackPlugin {
      // 参数options来接受new时配置的options
      constructor (options) {
        console.log(options.name)
      }
      // compiler是一个webpack的实例
      apply (compiler) {
        debugger;
        // emit：生成资源到 output 目录之前
        compiler.hooks.emit.tapAsync (compilation) {
          compilation.assets['copyright.txt'] = {
            source: function () {
              return 'copyright by ysh' 
            },
            size: function () {
              return 21
            }
          }
        }
      }
    }
  ```
  **调试**：在package.json中添加以下命令：
  ```json
    "scripts": {
      // 开启nodejs调试，运行后在浏览器上可进行调试。在对应的plugin代码中写上debugger即可
      "debug": "node --inspect --inspect-brk node_modules/webpack/bin/webpack.js"
    }
  ```

