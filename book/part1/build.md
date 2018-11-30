# 概览

这个部分主要讲一下vue源码的构建流程，当然只针对web环境下。

# 开始

首先从`package.json`开始，我们去掉ssr、test和weex部分，得到下面的web文件相关配置

    "scripts": {
        "dev": "rollup -w -c scripts/config.js --environment TARGET:web-full-dev",
        "dev:cjs": "rollup -w -c scripts/config.js --environment TARGET:web-runtime-cjs",
        "dev:esm": "rollup -w -c scripts/config.js --environment TARGET:web-runtime-esm",
        "dev:compiler": "rollup -w -c scripts/config.js --environment TARGET:web-compiler ",
        "build": "node scripts/build.js",
    },

可以看到开发分为4种情况，构建就只是node执行`build.js`即可，下面慢慢分析开发和构建情况。

## Dev

dev分为四种情况，分别是`full-dev`, `runtime-cjs`, `runtime-esm`, `compiler`,这些是什么意思呢？官网给出介绍：

| | UMD | CommonJS | ES Module |
| -- | -- | -- | -- |
| 完整版 |vue.js | vue.common.js | vue.esm.js |
| 只包含运行时版 | vue.runtime.js|vue.runtime.common.js|	vue.runtime.esm.js
| 完整版 (生产环境) | vue.min.js | | |
| 只包含运行时版 (生产环境)| vue.runtime.min.js | | |

**说明**

- **完整版**：同时包含编译器和运行时的版本。

- **编译器**：用来将模板字符串编译成为 JavaScript 渲染函数的代码。

- **运行时**：用来创建 Vue 实例、渲染并处理虚拟 DOM 等的代码。基本上就是除去编译器的其它一切。

- **UMD**：UMD 版本可以通过 \<script\> 标签直接用在浏览器中。jsDelivr CDN 的 https://cdn.jsdelivr.net/npm/vue 默认文件就是运行时 + 编译器的 UMD 版本 (vue.js)。

- **CommonJS**：CommonJS 版本用来配合老的打包工具比如 Browserify 或 webpack 1。这些打包工具的默认文件 (pkg.main) 是只包含运行时的 CommonJS 版本 (vue.runtime.common.js)。

- **ES Module**：ES module 版本用来配合现代打包工具比如 webpack 2 或 Rollup。这些打包工具的默认文件 (pkg.module) 是只包含运行时的 ES Module 版本 (vue.runtime.esm.js)。

也就是说**完整版就是编译器加运行时**，这个编译器作用是解析`template`标签为render函数。如果你需要在客户端编译模板 (比如传入一个字符串给 template 选项，或挂载到一个元素上并以其 DOM 内部的 HTML 作为模板)，就将需要加上编译器，即完整版：

    // 需要编译器
    new Vue({
        template: '<div>{{ hi }}</div>'
    })

    // 不需要编译器
    new Vue({
        render (h) {
            return h('div', this.hi)
        }
    })
所以使用了vue-loader、vueify之类预编译器时，`.vue`文件已经被编译为js了，此时就不需要引入完整版，只需运行时版即可，这样可减少打包体积。

dev生成的文件分别如下：

    dev => vue.js:编译+加运行时
    dev:cjs => vue.runtime.common.js: commonjs格式的运行时
    dev:esm => vue.runtime.esm.js: es module格式的运行时

`dev:compiler`是一个独立的模块，用以实现模板编译为render函数功能，以后专门分析。

## Build

开发的时候肯定是使用dev一个一个模块和系统进行开发，但是构建的时候肯定是将开发中相关的模块和系统都进行打包甚至某些做压缩丑化操作最后生成一系列的立即可使用的目标文件。本质就是运行多个相关构建任务。这里只考虑web情况下，所以会生成所有web有关的构建文件，可以说build是dev的整合和压缩优化。下面是该命令的生成文件：

    dist
    ├── vue.common.js
    ├── vue.esm.js
    ├── vue.js
    ├── vue.min.js
    ├── vue.runtime.common.js
    ├── vue.runtime.esm.js
    ├── vue.runtime.js
    └── vue.runtime.min.js
可以看到一次生成了所有web需要的文件。而不是像dev一样只生成一个。(废话)

# 源码分析

## build.js

build命令可以生成所需的所有文件，dev是其子集，所以我们从build命令开始分析。

先来看`scripts/build.js`吧：

```
let builds = require('./config').getAllBuilds()

// filter builds via command line arg
if (process.argv[2]) {
  const filters = process.argv[2].split(',')
  builds = builds.filter(b => {
    return filters.some(f => b.output.file.indexOf(f) > -1 || b._name.indexOf(f) > -1)
  })
} else {
  // filter out weex builds by default
  builds = builds.filter(b => {
    return b.output.file.indexOf('weex') === -1
  })
}

build(builds)

// 根据配置进行构建
function build (builds) {
    ......
}
// 具体的构建过程
function buildEntry (config) {
    ......
}
// 输出文件
function write (dest, code, zip) {
    ......
}
```
从上面可以看出大概流程就是获取所有构建配置，然后根据脚本参数(使用process.argv)过滤指定配置，最后分别对这些配置进行构建。很简单自然的流程。

有意思的是`build函数`

```
function build (builds) {
  let built = 0
  const total = builds.length
  const next = () => {
    buildEntry(builds[built]).then(() => {
      built++
      if (built < total) {
        next()
      }
    }).catch(logError)
  }

  next()
}
```
这里使用的是递归进行所有构建配置的处理，而非使用循环进行处理。这样感觉逼格比较高。但是由于是一个构建成功之后才进行下一个构建，所以整个过程比较缓慢，可以使用Promise.all来进行优化。

比如：

    // 普通版
    builds.reduce((p, build) => {
        return p.then(() => {
            return buildEntry(build).then(() => {
                console.log('success')
            }).catch(logError)
        })
    }, Promise.resolve())

    // promise.all
    Promise.all(builds.map(build => {
        return buildEntry(build).then(() => console.log('success')).catch(logError)
    })).then(() => {
        console.log('build success')
    })

## config.js

接着我们来看一下`config.js`:

```
// 别名
const aliases = require('./alias')
// 路径解析函数
const resolve = p => {
  const base = p.split('/')[0]
  if (aliases[base]) {
    return path.resolve(aliases[base], p.slice(base.length + 1))
  } else {
    return path.resolve(__dirname, '../', p)
  }
}
// 所有的构建配置项
const builds = {
  // Runtime only (CommonJS). Used by bundlers e.g. Webpack & Browserify
  'web-runtime-cjs': {
    entry: resolve('web/entry-runtime.js'),
    dest: resolve('dist/vue.runtime.common.js'),
    format: 'cjs',
    banner
  },
  // Runtime+compiler CommonJS build (CommonJS)
  'web-full-cjs': {
    entry: resolve('web/entry-runtime-with-compiler.js'),
    dest: resolve('dist/vue.common.js'),
    format: 'cjs',
    alias: { he: './entity-decoder' },
    banner
  },
   // 省略
   ......
}
// 根据名称生成构建配置
function genConfig (name) {
  const opts = builds[name]
  const config = {
    input: opts.entry,
    external: opts.external,
    plugins: [
      replace({
        __WEEX__: !!opts.weex,
        __WEEX_VERSION__: weexVersion,
        __VERSION__: version
      }),
      flow(),
      buble(),
      alias(Object.assign({}, aliases, opts.alias))
    ].concat(opts.plugins || []),
    output: {
      file: opts.dest,
      format: opts.format,
      banner: opts.banner,
      name: opts.moduleName || 'Vue'
    }
  }
    if (opts.env) {
    config.plugins.push(replace({
      'process.env.NODE_ENV': JSON.stringify(opts.env)
    }))
  }

  Object.defineProperty(config, '_name', {
    enumerable: false,
    value: name
  })

  return config
}

// 如果有target参数，则使用target参数对应配置，否则暴露获取配置的方法。
if (process.env.TARGET) {
  module.exports = genConfig(process.env.TARGET)
} else {
  exports.getBuild = genConfig
  exports.getAllBuilds = () => Object.keys(builds).map(genConfig)
}
```

这个文件代码也比较简单，就是如果是通过node repl进行构建操作的话就直接返回构建配置（比如dev的那几个构建命令），否则这个文件返回两个可以生成构建配置的函数。

这个文件值得注意的是resolve方法，该方法考虑了别名的情况，如果是别名则获取别名路径然后再将剩余路径进行拼接得到最后的路径，否则直接从根路径寻找。

## alias.js

而最后`alias.js`则更简单了，就是别名的路径对应的散列。

```
const resolve = p => path.resolve(__dirname, '../', p)
module.exports = {
  vue: resolve('src/platforms/web/entry-runtime-with-compiler'),
  compiler: resolve('src/compiler'),
  core: resolve('src/core'),
  shared: resolve('src/shared'),
  web: resolve('src/platforms/web'),
  weex: resolve('src/platforms/weex'),
  server: resolve('src/server'),
  entries: resolve('src/entries'),
  sfc: resolve('src/sfc')
}
```