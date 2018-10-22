## Rollup 求生指南

by scrollbar-ww@cheap-pets, oct 2018



### 1. 前言

足够幸运的话，你们也许正好需要做这些事情：

* 单页面文件打包
* 多页面文件打包
* 复制一些静态文件
* 在每次打包前清理目标文件夹
* 能够按浏览器兼容性要求转换 es6 代码
* 按浏览器兼容性要求，将用某些预处理语法规格的样式代码转换为标准 css 代码，并生成 css 文件
* 正确转换 .vue 组件文件，且能处理 .vue 文件中的样式部分
* 自动往页面中注入生成的的脚本与样式，且包文件名随机产生或包含哈希值
* 项目中可能同时包含 node 服务端代码与站点前端代码
* 在开发时自动监测代码变化进行打包
* Rollup 插件开发

如果有这样的需要，希望这篇文章可以帮到你们。

当然，有可能你们在用一些非常强大的脚手架工具（比如 vue-cli），一些真正的大爷已经帮你们做完了这些事情。但如果你碰巧希望了解一下这方面的内容，也可以无聊时看看这篇。



如果查看本文档的工具支持的话，建议先查看本文档的大纲，方便查看所需的章节内容。

顺便推荐使用 Typora，很好用的 markdown 工具，这货用 electron 做的。



#### 开发建议

这里是一些我们习惯的开发套路。本文后续将贴出一些相关配置代码，亦会按照这些套路出牌。

###### 工程目录结构与相关配置

根目录结构

```
build/ (打包代码的目录，通常用来处理多入口多出口打包项目)
dist/  (生成目标目录)
src/   (源码目录)
.browserlistrc   (浏览器类型、版本配置文件)
babel.config.js  (babel 7 的配置文件)
rollup.config.js (Rollup 配置文件，也许有 0 ~ n 个)
```

单入口项目相关目录结构（纯前端）

```
dist/  (生成目标文件夹)
    assets/ (从 static-content 里复制过来的)
    lib/    (从 static-content 里复制过来的)
    [other-static-dirs]/  (从 static-content 里复制过来的)
    app.[hash].css
    app.[hash].js
    app.js.map
    index.html
src/  (源码目录)
	css/  （样式源码目录）
		[some-postcss-files].pcss
	static-contents/  (静态内容，里边内容直接复制)
		assets/  (一般放字体、图片等)
		lib/     (三方库，HTML 里直接引用的脚本或样式文件)
		[other-static-dirs]/  (其他的静态内容)
	[some-js-module-dirs]/  (按需划分的脚本目录)
	index.html   (页面模板文件)
	index.js     (打包入口文件)
```

多入口（多出口）项目相关目录结构（前端多个 HTML 文件）

```
dist/
    assets/
    lib/
    [other-static-dirs]/
    module-a.css
    module-a.[hash].js
    module-a.js.map
    module-b.css
    module-b.[hash].js
    module-b.js.map
    module-a.html
    module-b.html
src/
	css/ (公共的样式文件夹)
	js/  (公共的脚本文件夹)
	static-contents/  (公共的静态文件)
	modules/
		module-a/ (每个页面一个模块文件夹)
            css/  (模块的样式文件夹)
            static-contents/  (模块的静态内容文件夹)
			[some-js-module-dirs]/  (模块内划分的脚本目录)
			module-a.html  (模块页面模板文件)
			index.js       (模块打包入口文件)
		module-b/
```

多入口项目相关目录结构（服务 + 前端，web-content 目录中的结构，参照上边两个框中的）

```
dist/
	web-content/
	app.js
	app.js.map
src/
	service/
		[some-js-module-dirs]/
		index.js
	web-content/
```



###### 使用 postcss 进行样式开发

非常推荐使用 postcss 进行样式代码的开发，最大的方便之处在于可以自行创造许多方便的样式写法。

欢迎查看我编写的 postcss 相关文档。

本文中样式打包部分，均采用 postcss 相关配置作为示例，还会介绍一些打包 postcss 样式源码的技巧。



###### 其他

* 服务端不打包 node_modules 中的代码
* 开发模式下，不对目标代码进行压缩
* 开发模式下，打包生成的脚本与样式文件，文件名中不包含哈希片段



#### 模块依赖

要按本文所写的方法进行项目打包，在项目会依赖到若干三方包，这里贴出来 package.json  文件中相关依赖的片段：

``` json
"devDependencies": {
  "@babel/core": "^7.1.2",
  "@babel/plugin-transform-runtime": "^7.1.0",
  "@babel/preset-env": "^7.1.0",
  "core-js": "^2.5.7",
  "rollup": "0.66.6",
  "rollup-plugin-babel": "^4.0.1",
  "rollup-plugin-clear": "^2.0.7",
  "rollup-plugin-commonjs": "9.1.8",
  "rollup-plugin-copy": "^0.2.3",
  "rollup-plugin-gen-html": "^0.1.1",
  "rollup-plugin-json": "^3.1.0",
  "rollup-plugin-node-resolve": "^3.4.0",
  "rollup-plugin-postcss": "^1.6.2",
  "rollup-plugin-re": "^1.0.7",
  "rollup-plugin-uglify": "^6.0.0",
  "rollup-plugin-vue": "^4.3.2",
  "vue": "^2.5.17",
  "vue-template-compiler": "^2.5.17"
}
```

不要吓到，稍后会仔细说明用途。

嗯，这些依赖都放到 `devDependencies` 中。

而且本文假设你们对 package.json 很熟悉，都会使用 npm 或 yarn 命令。



#### 各路官方文档

本文仅作为一个入门文档，不太会涉及各路工具的原理、高级用法。

更详细的内容，还是建议各位大爷好好查看各路官方文档，很简单，用名称作为关键字，在 npm 或 github 都能找到，这里也不再一一贴出地址。



### 2. Rollup 基础

Rollup 有两种常见用法，CLI 与 API。

#### 1）CLI 命令行

通常情况下，使用 Rollup CLI 命令能满足绝大多数的打包场景。

##### 最简单的用法，直接指定入口文件与目标文件：

``` bash
rollup src/index.js --file dist/app.js --format umd --name "moudle_name"
```

这种用法通常可以用来打包一些简单组件或工具项目。

##### 使用配置文件

``` bash
rollup -c [rollup.config.js]
```

配置文件名可以省略，默认为 rollup.config.js。

配置文件的内容：

``` js
export default {
  input: 'src/index.js',
  plugins: [
    ...
  ],
  output: {
	file: 'dist/app.js',
    format: 'iife'，
    sourcemap: true
  }
}
```



配置文件，是个普通的 esm 脚本文件，可以在其中写一些代码进行复杂的操作，比如自己写一些插件逻辑。

配置内容和 CLI 参数稍后进行说明。



#### 2）使用 API

当我们需要处理多入口（多出口），或打包时需要配合进行一些代码进行其他操作时，可以使用 Rollup 的 API 进行打包。

``` js
const rollup = require('rollup')
...
const inputOptions = {
  input,
  plugins,
  external
}
const outputOptions = {
  file,
  format,
  sourcemap
}
const watchOptions = {
  ...inputOptions,
  output: outputOptions
}

// 配置输入参数，如入口文件、插件等，并生成打包对象
const bundle = await rollup.rollup(inputOptions)

// 配置输出参数，如目标文件、输出类型等，并生成打包结果
bundle.write(outputOptions)

// 配置输入、输出参数，进行打包，并监视文件变化随时打包
rollup.watch(watchOptions)
```

watch 方法本身就会执行一次初始打包动作，之前无需额外创建 bundle 对象并执行 write 方法。

初始打包若出错，进程会退出，但监视过程中若没有致命错误，监视打包仍将继续。



#### 3）基本参数

就 Rollup 本身而言，很少的几个配置参数（4个？）就可以解决大部分的问题。

这些参数，在命令行、配置文件、API 脚本中都能简单找到相应的位置。

##### input  入口

input 参数指向需要进行打包的入口文件路径，这是 Rollup 指明的唯一必不可少的参数。

打包到一起的所有的文件，都由这个入口文件开始，通过 ES Module、CommonJS 或其他引用方式，层层关联到一起。

由于每次 rollup 操作，只能指定 1 个入口（虽然有一些支持多入口的插件，但并不是针对生成多个目标，用起来并不太理想）。所以当我们需要在一个项目中处理多个入口并生成多个目标时，我们需要编写不同的配置，进行多次 Rollup 操作。



##### output 输出

输出参数是一组配置，主要包含几个主要的参数：

###### file

指向打包最终生成的脚本文件路径。

注意，这里只是表示生成的 js 脚本文件。有时候，我们通过一些插件，比如 postcss 样式处理插件或 HTML 插件，可能会生成 .css 或 .html 文件，这些都不是 Rollup 自己干的（当然，必须得感谢它提供的文件类型过滤处理机制）。

###### format

指定生成的目标脚本类型，包括如下几种：

- `amd` – AMD 模块，用于被 RequireJS 之类的加载器加载，现在用的人少了吧
- `cjs` – CommonJS 模块，用于 node 模块项目
- `es` – ES模块，通常用于组件或函数工具的分发
- `iife` – 自加载模块，一般用于立即执行函数或在浏览器中使用 <script> 标签加载
- `umd` – 通用模块定义，同时支持`amd` 和 `CommonJS` 加载，也可立即执行

###### sourcemap

布尔值，表示是否生成 sourcemap 文件。



##### external

设置需要保留外部引用的模块。

我们在打包 node 项目时，经常需要指定一些保留外部引用方式的模块。这些模块，仍然会保留 CommonJS 的方式来引用，不会打包到目标代码中。

external 参数可以设置为一个模块 id 列表数组，或者一个函数，如：

``` js
external: id => (id === 'fs' || id === 'path')
```

当我们希望保留三方包的文件作为外部依赖时，可以使用该配置，见示例代码：

``` js
// 这个写法表示希望外部引用的包，都定义在了 dependencies 中，这本身也是个很好的代码规范
const pkg = JSON.parse(readFileSync('./package.json', 'utf-8'))
const external = Object.keys(pkg.dependencies || {})
```



##### plugins

用来指定识别各类被引用的文件，并对他们进行转换的插件们。

Rollup 的打包功能，简单的说，它只是识别代码中的 ES Module 引用，并把其中真正用到的代码串到一起，它并不能处理 CommonJS 引用，不能把 ES6 代码转换成大多数浏览器能正常运行的 ES5 代码，不能进行样式预处理，甚至也不会做拷贝静态文件这类简单的操作。但通过插件体系，它可以做到这一切。

下一章节中，我们会详细介绍几个非常常用的插件，这也是本文的重头之一。



### 3. Rollup 插件与相关配置

在列出这一堆插件之前，我们拿开发一个单页应用为例，再描述一遍我们的需求：

1. 脚本代码是 ES6 语法和 相关 API 书写
2. 项目使用 Vue 框架，包含若干 .vue 组件文件
3. 样式用 POSTCSS 相关插件的规则书写
4. 应用需要在 IE9、Chrome 50、Firefox 50 浏览器中正常使用
5. 项目包含有一些静态的图片、字体
6. 项目代码中，通过 ES Module 方式引用了一些其他模块中的代码，这些模块通过 npm 或 yarn 安装在 node_modules 文件夹中。这些依赖到的文件代码，需要打包合并到目标文件中。
7. 项目中包含一些在页面中直接引用的三方脚本和样式，这些三方文件，最好能通过 npm 或 yarn 进行安装和更新。这些文件，不想被打包合并到目标文件中。
8. 开发过程中，所需生成的文件名称、数量可能有变化，每次打包前，希望清理目录，避免留下垃圾
9. 需要在 HTML 文件中自动按正确路径引用生成后的脚本、样式文件，且为了避免浏览器缓存，希望目标文件名称在每次生成后能有所变化。

咱们写贴一份完整的插件配置参数代码，后续的插件描述中，将阐述如何通过引入插件解决这些需求。插件将按在配置中书写的顺序介绍。

#### 1）完整配置

``` js
// rollup.config.js

import clear from 'rollup-plugin-clear'
import copy from 'rollup-plugin-copy'
import html from 'rollup-plugin-gen-html'
import vue from 'rollup-plugin-vue'
import json from 'rollup-plugin-json'
import resolve from 'rollup-plugin-node-resolve'
import commonjs from 'rollup-plugin-commonjs'
import re from 'rollup-plugin-re'
import babel from 'rollup-plugin-babel'
import uglify from 'rollup-plugin-uglify-es'

// 判断当前是否生产环境代码打包
const env = process.env.NODE_ENV || 'production'

// 配置插件
const plugins = [
  // 清空目标目录
  clear({
    targets: ['dist']
  }),
  // 复制静态文件
  copy({
    'src/assets': 'dist/assets'
  }),
  // 支持 .vue 代码打包
  vue(),
  // 支持代码中引用 .json 文件
  json(),
  // 支持 postcss 样式文件
  postcss({
    extract: true,
    plugins: [] // 这里有一堆 postcss 的插件定义略去
  })
  // 支持代码中引用的 node_modules 中的文件
  resolve({
    module: true,
    jsnext: true,
    main: true,
    browser: true
  }),
  // 支持代码中引用 CommonJS 规格的模块
  commonjs(),
  // 使用 Babel 转换代码
  babel({
    exclude: [/\/core-js\//],
    runtimeHelpers: true,
    sourceMap: env === 'production',
    extensions: ['.js', '.jsx', '.es6', '.es', '.mjs', '.vue']
  }),
  // 从模板生成 html 文件，并自动引入生成的脚本和样式
  html({
    template: 'src/index.html',
    target: '../index.html',
    hash: env === 'production',
    replaceToMinScripts: env === 'production'
  })
]
// 生产环境打包时，压缩代码
// if (env === 'production') plugins.push(uglify()) // 改为使用 babel-minify

// 输出 rollup 完整配置
export default {
  input: 'src/index.js',
  output: {
    file: 'dist/assets/app.js',
    format: 'iife',
    sourcemap: env === 'production'
  },
  plugins
}
```

这些插件的名称很天真，能一眼看出来对应的功用。



#### 2）rollup-plugin-clear

用来清理指定的目录或文件。

参数示例：

``` json
{
    "targets": [ "路径1", "路径2" ]
}
```

如果是用相对路径，则表示基于 `process.cwd()` 的相对路径。

如果不太确定运行的位置，建议使用 `path.resolve()` 生成基于 `__dirname` 的绝对路径。以下有多处与此类似。



#### 3）rollup-plugin-copy

用来复制文件到指定目录。

参数示例：

``` json
{
  "源文件夹或文件路径": "目标文件夹或文件路径"，
  ...
}
```



#### 4）rollup-plugin-vue

用来支持 .vue 文件转换。

参数示例：

``` json
{
    "css": true // true 表示将内置的 style 转换并嵌入 js 中，false 则输出给其他样式处理插件
}
```

最常用的参数也就这一个了，别的参数参见：

https://rollup-plugin-vue.vuejs.org/options.html

这里要特别说明一下插件版本，4.x 的版本与之前的版本有较大差异，特别是样式处理的方式。

以前采用 `node-sass` 进行样式转换，现在改为了使用 `postcss`，在安装插件依赖时确实方便了很多。

另外，当我们需要使用 `Rollup API` 方式进行打包时，在 `CommonJS` 里引用 `rollup-plugin-vue` 插件需要这么写（目前仅限 4.x 版本）：

``` js
const vue = require('rollup-plugin-vue').default
```

当我们需要把 `Vue` 包与我们的代码打包到一起时，我们可能在代码中这么引用 `Vue`:

``` js
import Vue from 'Vue'

new Vue({
  // 这里是主视图对象的创建参数
})
```

`Vue` 包的 `package.json` 是这么指定 esm 的入口的：

``` json
// node_modules/vue/package.json

"module": "dist/vue.runtime.esm.js"
```

`Vue` 的 `runtime` 包，不包含在运行期进行组件模板编译的功能（文件体积要小很多），所有 `Vue` 模板均在构建时进行编译。

这种情况下，我们的页面模板 (.html 文件) 或 js 代码中，不要包含 `Vue` 模板代码，因为无法被编译到，会造成运行出错。

html 文件中，通常只包含这一个 `Vue` 主视图入口元素：

``` html
<div id="app"></div>
```

对应的 index.js 和 app.vue：

``` js
// index.js
import App from './app.vue' // 引入主视图组件

new Vue({
  render: h => h(App)，
  // router, // 如果使用 vue-router 的话，指定路由对象
  // store,  // 如果使用 Vuex 的话，指定 store 对象
  // 其他参数
}).$mount('#app') // 渲染到 html中准备好的 div
```

``` vue
// app.vue
<template>
	<div>
        ...
    </div>
</template>

<script>
    export default {
        // 主视图组件的配置
    }
</script>
```

如果需要在运行期进行模板编译，可以使用非 `runtime` 版本的 `Vue` 文件，比如在 `<script>` 标签中引用 `vue.js` 或 `vue.min.js` 。 

该插件当前 4.x 版本（现在是 18 年 10 月），生成 vue 文件的 sourcemap 好像还有些问题，可能会影响调试。

npm 中，还有一个名为 `rollup-plugin-vue2` 的插件，功能不完善，且已经停止维护，大家尽量不要使用。



#### 5）rollup-plugin-json

支持 .json 文件的引用。

有时候，代码中需要使用 json 文件中的数据。

比如打包 axios 就会需要这个，不然报错。



#### 6）rollup-plugin-postcss

用来处理采用 postcss 及其各种预处理插件相关规格的样式代码。

参数示例：

``` js
// rollup.config.js

import postcss from 'rollup-plugin-postcss'
import postcssAutoprefixer from 'autoprefixer'
import postcssClean from 'postcss-clean'
import postcssColorMix from './rollup-custom-plugins/postcss-plugin-color-mix'
import postcssConditionals from 'postcss-conditionals'
import postcssFor from 'postcss-for'
import postcssImport from 'postcss-import'
import postcssMixins from 'postcss-mixins'
import postcssNested from 'postcss-nested'
import postcssSelectorNot from 'postcss-selector-not'
import postcssVars from 'postcss-simple-vars'
import postcssUnprefix from 'postcss-unprefix'
import colors from './src/color-variables'

export default {
  input,
  output,
  plugins: [
    postcss({
      extract: true, // true 表示生成与 output.file 相同 basename 的 .css 文件
      plugins: [
        // 一堆美妙的插件帮我们写很 cool 的样式代码
        postcssImport,
        postcssUnprefix,
        postcssSelectorNot,
        postcssMixins,
        postcssVars({ variables: colors }),
        postcssNested,
        postcssFor,
        postcssConditionals,
        postcssMixColor,
        postcssAutoprefixer,
        postcssClean
      ]
    }) 
  ]
}
```

postcss 处理样式代码时，会将所有的样式解析成结构化数据缓存起来，所以每个目标输出不可共用一个插件实例，否则后边打包的目标样式中会包含前序目标中的样式代码。



#### 7）rollup-plugin-node-resolve

支持从 node_modules 中引用文件，通常与 `rollup-plugin-commonjs` 一起使用。

参数示例：

``` json
{
  "module": true,
  "jsnext": true,
  "main": true,
  "browser": true
}
```

这几个参数，表示从 node_moduels/[pkg]/package.json 中查找对应的模块入口属性。

package.json 中，可以指定不同的 js 文件分别作为包依赖入口，包括：

* main - 通常是 CommonJS 入口
* module - ES Module 入口
* browser - 浏览器环境下入口
* jsnext:main - 兼容一些古老的包中指定 ES Module 入口的属性，现在很少见了



#### 8）rollup-plugin-commonjs

支持引用 CommonJS 规格模块。

该插件通常不需要什么参数。



#### 9）rollup-plugin-babel

该插件引入 `Babel` 来处理代码。

`Babel` 通常用来转换 ES6 代码，生成兼容大部分浏览器的 ES5 代码。

还可以通过若干插件支持，将很超前的 ES20xx 语法转换为对应运行环境能够执行的代码。

参数示例：

``` js
babel({
  'exclude': [/\/core-js\//],  // 不转换 polyfill 注入的代码，避免产生循环依赖警告
  'runtimeHelpers': true, // 如果要使用 transform-runtime 就设置为 true
  'externalHelpers': false,
  'sourceMap': true,
  'extensions': ['.js', '.jsx', '.es6', '.es', '.mjs', '.vue'] // 所需转换的文件后缀
})
```

`externalHelpers` 这个参数说明一下：

这个参数的默认值是 false，当各模块需要用到 Babel 的各类 helper 时，会在每个模块的顶部注入 helper 函数。

设置为 true 时，会单独引入一个包含对应 helper 的模块文件，避免重复。设置为 true 时，需要在 babel 插件中配置 `@babel/plugin-external-helpers`。

如果使用 `@babel/transform-runtime`（这个在 Babel 章节会介绍），这个值不需要设置为 true。



`Babel`相关参数可以写在该插件配置中，也可以写在单独的 babel.config.js 中。

通常，单入口应用的转换配置或通用的配置项，建议写在 babel.config.js；

多入口应用，且每个入口项目的类型差别较大，可以将差异部分写在插件配置中。

详细的 `Babel` 配置我们下节介绍。



早期的 `Rollup` 入门文档中，经常会出现一个叫做  `rollup-plugin-buble` 的插件介绍。

`Buble` 也是用来转换 ES6 代码的，速度很快，而且也在持续的频繁的维护。

但 `Buble` 目前仍对复杂的 ES6 项目无能为力，暂不推荐大家使用。

稍后一个大的章节将继续为大家介绍 Babel。



#### ~~10）rollup-plugin-uglify~~ 

~~用来压缩和混淆代码。且只能用于目标是 es5 的代码。~~

~~该插件通常也不需要额外参数。~~

~~在 `CommonJS` 里引用 `rollup-plugin-vue` 插件需要这么写：~~

```
const uglify = require('rollup-plugin-uglify').uglify
```



#### ~~11）rollup-plugin-uglify-es~~

~~也是用来压缩和混淆代码，支持 es6 代码。~~

<!--压缩代码建议使用 babel-minify preset 进行-->



#### 12）rollup-plugin-gen-html

哈哈，这个插件是我写的。

先贴参数示例：

``` json
{  
  "template": "src/index.html", // html 模板文件路径
  "target": "../index.html", // 目标文件路径，相对于 output.file 的路径
  "hash": true, // 是否在 output.file 文件名中加入 hash 片段，避免浏览器缓存
  "replaceToMinScripts": true // 是否自动将 <script> 标签中引用的 .js 文件替换成 .min.js 文件
}
```

index.html 模板示例：

``` html
<!DOCTYPE html>
<html lang="zh">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <title>my page</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="text/javascript" src="./lib/axios/axios.js"></script>
    <script type="text/javascript" src="./lib/vue/vue.runtime.js"></script>
    <script type="text/javascript" src="./lib/vuex/vuex.js"></script>
    <script type="text/javascript" src="./lib/vue-router/vue-router.js"></script>
  </body>
</html>
```

生成后：

``` html
<!DOCTYPE html>
<html lang="zh">
<head>
  <meta charset="utf-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <title>智慧工地门户</title>
  <link rel="stylesheet" type="text/css"
   href="assets\app.a08bc9c38e2f9ec591aa5a6d4baa47c8.css">
</head>
<body>
  <div id="app"></div>
  <script type="text/javascript" src="lib\axios\axios.min.js"></script>
  <script type="text/javascript" src="lib\vue\vue.runtime.min.js"></script>
  <script type="text/javascript" src="lib\vuex\vuex.min.js"></script>
  <script type="text/javascript" src="lib\vue-router\vue-router.min.js"></script>
  <script type="text/javascript" src="assets\app.e14a225c5fd710f87ac6a925dcec46f4.js">
  </script>
</body>
</html>
```

`replaceToMinScripts` 参数为 `true` 时，会在目标路径中检查是否包含对应的 .min.js 文件，存在的话才会进行替换。

建议仅在生成生产环境的代码时，设置 `hash` 和 `replaceToMinScripts` 参数为 `true` 。



### 4. Rollup 插件开发入门

随着项目复杂度得提高，对打包工作的要求也会提高。

虽然 Rollup 的生态还算不错，各类插件基本齐全，但总会遇到一些不顺手的事情。

比如说，我们希望生成 html 页面，并自动插入 生成的脚本。虽然有好几个 html 相关的插件，但不能满足我们的个性要求。这个时候，我们会需要制作一些简单的 Rollup 插件来帮忙。

制作 Rollup 插件，了解了如下几个基本知识后，其实很简单：

#### 1）插件代码格式

编写 Rollup 插件通常会要提供 commonjs 和 ES Module 两个版本的脚本文件，分别用于使用 Rollup 的 API 模式与 CLI 模式。

插件代码需要导出这样一个函数：

``` js
// es module
export default (options = {}) => {
  return {
    name: '插件名称',
    transform (code, id) {
    },
    onwrite (config, data) {
    }
  }
}
```



##### options 参数

其中，options 参数，是运行时指定给插件的参数。如我们之前贴过的 html 插件的参数：

``` js
html({
  template: 'src/index.html',
  target: '../index.html',
  hash: env === 'production',
  replaceToMinScripts: env === 'production'
})
```



##### transform 方法

transform 方法，会在 Rollup 进行模块代码解析时调用，具有两个关键参数：

###### code：

当前模块的代码内容

###### id：

当前模块的名称，通常是模块文件的物理路径，或由其他解析器摘出来的代码的逻辑路径，比如 .vue 文件中的样式代码的逻辑路径。

插件中一般需要根据 id 中的物理或逻辑文件的后缀名，判断代码是否需要当前插件解决。

###### 

transform 方法中，当判断出对应代码需要该插件进行处理时，且结果是 js 片段时，需要返回如下结果：

``` js
transform (code, id) {
  return {
      code,  // 转换后的代码
      map // sourcemap 数据
  }
}
```



##### onwrite 方法

onwrite 方法在 Rollup 进行 `bundle.write()` 时执行。

它的两个参数：

* config - rollup 的 output 配置参数，通常用来获取目标文件路径等；
* code - 最终产生的 js 代码内容。



#### rollup-pluginutils

这是一个开发 rollup 插件常用的辅助工具包，包含代码类型过滤、生成逻辑文件名等功能。



### 5. Babel

 `Babel` 其实功能很简单，就是将新的语法、新的对象或老的对象的新API 转换成目标浏览器、node、electron 等环境能够支持的老的语法与 API。

注意，上边这句话表示有两类东西需要转换：

* 语法
* 新的对象或老的对象的新API

为了帮助 `Babel` 完成这一工作，我们可能需要通过配置告诉它如下一些信息：

* 源代码有什么特征（包含哪些特别高级的语法需要转换）
* 目标代码需要在什么样的环境下运行
* 通过什么方式进行新特性对象、API 的转换（polyfill、transform-runtime）
* 其他的处理，比如压缩代码

这些配置过程通常都是通过引入 `babel` 插件来进行的。

利用 Babel 的插件机制，你们甚至可以定义自己喜好的各种奇葩变态语法或 API 。



#### 1）Babel 7

一个月之前，Babel 7 大版正式发布，与 Babel 6 相比，主要变化有：

* 核心包名改为名空间模式，@babel/xxx，比如 `babel-core -> @babel/core`。
* 废除了一些老旧过时的 presets，比如 `preset-es201x` 、`stage-x`，统一改为使用 `@babel/preset-env`，`preset-env` 变得更加方便好用。
* 一些包名更改

最直观的感受，Babel 7 比以前配置更简单了。

本文的配置代码，均按照 Babel 7 书写。

从 Babel 6 升级到 Babel 7，可以采用 `babel-upgrade` 工具，用法简单，自己去查。



#### 2）基本概念

开始学习使用 Babel，必须知道这几个基本概念，不然看各种网上的文档就会晕死，这里先解释得通俗点：

* 配置：babel 运行必须要有配置参数，一般写在名为 babel.config.js 或 .babelrc 的文件中
* plugin： 是完成单一功能的插件，有些插件需要进行配置
* preset：一组完成特定功能的 plugins 的组合
* polyfill：babel 的各路 preset 通常只是转换 JS 语法，不能直接提供一些新对象、新 API 的支持。 polyfill 是用来转换这些新鲜 API 的方式，它也是通过 plugin 来进行的。

使用 plugin 和 preset 之前，必须先安装相应的依赖。



#### 3）babel.config.js

这里先贴一个简单的配置文件（也可能是大多数情况下足够用的）：

``` js
// babel.config.js
module.exports = {
  presets: [
    [
      '@babel/preset-env',
      {
        modules: false,
        useBuiltIns: 'usage'
      }
    ],
    [
      'minify',
      { builtIns: false }
    ]
  ],
  plugins: [
    '@babel/plugin-external-helpers'
    // '@babel/plugin-transform-runtime' 若使用 babel-runtime，可以用这个插件
  ]
}
```

早期的 Babel 文档中，更多的是出现一个叫 .babelrc 的 json 配置文档。

在 Babel 7 时代，更推荐大家使用 babel.config.js 作为配置入口或整个配置，至少在代码中更方便针对当前环境进行配置变化。

配置文件一般放在项目根目录中。



#### 4）presets

Babel 配置中，presets 定义格式可以包含或不包含参数：

```js
presets: [
    'presetA', // 只包含 preset 名称，不含参数
    [ 'presetA', { } ] // 包含参数的定义
]
```

感谢 Babel 7，现在我们几乎只需要使用上边配置中的两个 presets 了。



##### env

全名 `@babel/preset-env`，以前叫做 `babel-preset-env`。

这也是以前很多文档中说的 “也许是你们唯一需要的 Babel 配置”。

官方描述大概是这样： “这是一个很聪明的预设，通过它，你可以在开发中随便使用最新的 js 语法，不用为每个新语法操心其是否可以在目标运行环境中正确执行。” （是的，以前确实是需要单独为很多新语法进行配置的。）

env 包含三个常用的配置参数：

* modules - 需要 Babel 处理的模块引入方式，默认是 `commonjs`，但在 Rollup 中一般设置为 `false`，表示不需要 Babel 进行模块处理。
* useBuiltIns - 这个与 polyfill 有关，这里先不解释。
* targets - 指定运行的环境，比如浏览器类型、版本等。其格式是 browserlist 规格的查询内容。本文会建议大家把这个配置单独写到 .browserlistrc 中，这里不要写。



##### minify

包名 `babel-preset-minify`，该包用来进行 js 代码的压缩。

比起 uglify 或 uglify-es 进行代码压缩，minify 能更简单的根据目标环境进行代码压缩，前两者应对较新的语法会比较吃力。

minify preset 也是整合了很多个针对代码不同特性进行压缩的插件，其配置参数，主要是指定各种特性是否进行压缩。

我们通常只设置 `builtIns: false`，这也算 minify 目前的坑之一，不设置这个参数很多情况下容易报错（Couldn't find intersection ）。



#### 5）browserlist

browserlist 并不是一段简简单单的制定浏览器的配置代码。

它的背后一个很有名的用来描述各类 js 运行环境（浏览器、node、electron）以及不同版本的特性差异的包。其主要开发者是 ai，也就是 postcss 的主要开发者。

有很多跟运行环境有关的转换器或代码检查器借助 browserlist 实现，比较常见的有 Babel 和 autoprefixer。

browserlist 的配置通常写在 `package.json`、`.babelrc` 或 单独的 `.browserlist.rc` 中。

写在 .browserlistrc 中的好处是，能用同一份配置，同时支持多个 转换、检查工具。

配置示意：

``` 
// .browserlistrc
last 1 version
> 1%
maintained node versions
not dead
```

上边这些看起来很高深但其实没什么卵用。作为 web 应用，你可以这么粗暴：

```
// .browserlistrc
chrome >= 36
firefox >= 50
ie >= 9
```

这样就很好理解了对吧。



#### 6）polyfill 与 transform-runtime

前边说了，只用 preset-env  的话，会将语法（比如说各种解构）转换掉。

要将用到的新对象（比如 Promise、Map），老对象的新方法 （比如 array.prototype.find）转换成目标环境能够运行的代码，Bebel 提供了两种方案，polyfill 与 runtime。这两方案不是完全的替代关系，虽然大多数情况下能解决相同的问题。



##### polyfill

包名 `@babel/polyfill`。

polyfill 的模式下，是全局引用转换的代码。一次引用之后，所有的模块中都能直接使用新 API。

使用时，需要在其他代码之前先引如 polyfill 文件。

最简单粗暴的方式，可以在页面中 <script> 标签内静态引入，

``` html
<script src="polyfill.js"></script>
```

`@babel/polyfill`  包中可以找到 dist/polyfill.js 与 dist/polyfill.min.js 。



也可以在打包文件中包含 polyfill 内容。

我们再回头关注一下 `preset-env ` 配置中的 `useBuiltIns` 参数的三个可选值：

- false - 默认值，表示不自动在每个模块文件中引入对应的 polyfill 模块文件，开发者进行手动引用
- entry - 自动在最开始引用全部所需的 polyfill
- usage - 自动在各模块引用所用到的 polyfill 模块，这个通常生成的文件会小一点，一般选这个

当设置为 entry 和 usage 时，Babel 会帮我们自动引入所需的 polyfill 文件。

如果是手动引入，可以在模块入口 js 中加入引用：

``` js
import '@babel/polyfill'
```

也可以直接引用 core-js：

``` js
import 'core-js'
```

`core-js` 这个包中，是 Babel 提供的各路 polyfill 函数，polyfill 和 runtime 也会使用其中的函数。



##### transform-runtime

这个插件的用途，是按需要将 babel 的 helper 函数、 core-js 中相关的函数、regenerator 函数放到模块代码的顶部作为引用。

参数示例（含默认值）：

```json
{
  "plugins": [
    [
      "@babel/plugin-transform-runtime",
      {
        "corejs": false,
        "helpers": true,
        "regenerator": true,
        "useESModules": false
      }
    ]
  ]
}
```

在 Babel 7 中，`corejs` 参数 具有两个值 false 与 2 。设置为 false 则不会引入 core-js 中的兼容代码，设置为 2 时，将会按需加入 @babel/runtime-core2 的引用。



使用 runtime 通常生成的代码比 polyfill 小，但也有它的问题，比如：

* 无法处理某些实例方法

```js
"foobar".includes("foo")
```

这样的代码就无法被 runtime 识别并添加兼容性代码。因为它不方便通过语法字面分析判断出 "foobar" 这个字符串常量是个什么东西。

这种情况下，可以用 polyfill 方式解决，或者在之前添加代码。

* 某些进行全局 API 判断的代码会失败

比如，Vuex 中，有代码判断 windows.Promise 是否存在，没有的话就歇了。

但 runtime 并不会凭空添加一个 window.Promise 出来。

这样的情况下可以添加一个“强迫” runtime 添加全局 API 的代码：

``` js
window.Promise = window.Promise || Promise
```



### 6. 多目标文件打包

Rollup 本身并不太适合进行多目标的打包，比如一个站点中存在多个页面的情况。

这个时候，我们有两个方法：

1) 可以编写多个 rollup 配置文件，用多个命令行串起来运行

2) 利用 Rollup API，编写一个通用的多入口、目标打包脚本

第一个没有什么好说的，按前边的方法配置就行，这里主要说第二种的思路。



多目标文件可能有这样的情况：

* 同时有不同类型的目标，如 服务端代码、前端页面代码
* 多个页面共享了相同的一组静态文件



大致思路如下：

1. 编写代码进行全局公用目录的清理，以及静态文件复制，这部分代码最先执行
2. 为每种类型的目标，编写一个生成 rollup 配置参数的函数。若不同类型目标若存在不同的 Babel 配置，将这部分配置写入 rollup-plugin-babel 配置，而不是 babel.config.js
3. 定义一个多入口、目标配置的数据格式，每个项目按需进行配置
4. 运行时遍历这个配置，按类型生成不同的 rollup 配置，调用 Rollup 命令



### 7. 关于 gulp

有不少大爷喜欢用上古神器 gulp，gulp 简单、方便，这个无疑。但有些事情这里需要提醒一下。

* 不要使用 gulp 的 watch 功能

因 rollup 自身的 watch 功能，是精确的根据 transform 过程中加载的文件进行 watch 的。

而 gulp 通常是根据整个文件夹进行 watch。

特别是多目标的项目中，rollup.watch 不会造成浪费。

其次，rollup 打包时，有缓存机制，会将转换的代码节点缓存起来，当文件变化时，只会重新转换变化的节点，这样性能高很多。但使用 gulp 不会用到缓存机制。

