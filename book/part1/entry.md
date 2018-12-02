****# 概览
话不多说，来看看`package.json`中的第一个构建项：(dev => vue.js:编译+加运行时),了解一下完整版的vue.js入口。按照上一节讲的，其脚本如下：
```
"scripts": {
    "dev": "rollup -w -c scripts/config.js --environment TARGET:web-full-dev",
},
```
继续找`web-full-dev`配置项，在`scripts/config.js`中找到其配置：
```
  // Runtime+compiler development build (Browser)
  'web-full-dev': {
    entry: resolve('web/entry-runtime-with-compiler.js'),
    dest: resolve('dist/vue.js'),
    format: 'umd',
    env: 'development',
    alias: { he: './entity-decoder' },
    banner
  },
```
上节讲了resovle解析的是`alias.js`中web部分，即解析到`src/platforms/web/entry-runtime-with-compiler.js`这个文件，我们瞅瞅这个文件吧。
```
/* @flow */

import config from 'core/config'
import { warn, cached } from 'core/util/index'
import { mark, measure } from 'core/util/perf'

// 正如上节所说完整版包含运行时和编译器，这里是Vue运行时部分
import Vue from './runtime/index'
import { query } from './util/index'
// 猜都知道这里是编译器
import { compileToFunctions } from './compiler/index'
import { shouldDecodeNewlines, shouldDecodeNewlinesForHref } from './util/compat'

const idToTemplate = cached(id => {
  const el = query(id)
  return el && el.innerHTML
})

// mount缓存了原来运行时$mount方法
const mount = Vue.prototype.$mount
// 更新为带编译器的$mount方法，
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)

  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
      `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
    )
    return this
  }

  const options = this.$options
  // resolve template/el and convert to render function
  if (!options.render) {
    let template = options.template
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          template = idToTemplate(template)
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && !template) {
            warn(
              `Template element not found or is empty: ${options.template}`,
              this
            )
          }
        }
      } else if (template.nodeType) {
        template = template.innerHTML
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
        return this
      }
    } else if (el) {
      template = getOuterHTML(el)
    }
    if (template) {
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile')
      }

      const { render, staticRenderFns } = compileToFunctions(template, {
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      options.render = render
      options.staticRenderFns = staticRenderFns

      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile end')
        measure(`vue ${this._name} compile`, 'compile', 'compile end')
      }
    }
  }
  return mount.call(this, el, hydrating)
}

/**
 * Get outerHTML of elements, taking care
 * of SVG elements in IE as well.
 */
function getOuterHTML (el: Element): string {
  if (el.outerHTML) {
    return el.outerHTML
  } else {
    const container = document.createElement('div')
    container.appendChild(el.cloneNode(true))
    return container.innerHTML
  }
}

Vue.compile = compileToFunctions

export default Vue

```

Emmmm,我们先看运行时做了什么吧，编译器部分以后再说。那我们来到`./runtime/index.js`,
```
/* @flow */

import Vue from 'core/index'
import config from 'core/config'
import { extend, noop } from 'shared/util'
import { mountComponent } from 'core/instance/lifecycle'
import { devtools, inBrowser, isChrome } from 'core/util/index'

import {
  query,
  mustUseProp,
  isReservedTag,
  isReservedAttr,
  getTagNamespace,
  isUnknownElement
} from 'web/util/index'

import { patch } from './patch'
import platformDirectives from './directives/index'
import platformComponents from './components/index'

// install platform specific utils
Vue.config.mustUseProp = mustUseProp
Vue.config.isReservedTag = isReservedTag
Vue.config.isReservedAttr = isReservedAttr
Vue.config.getTagNamespace = getTagNamespace
Vue.config.isUnknownElement = isUnknownElement

// install platform runtime directives & components
extend(Vue.options.directives, platformDirectives)
extend(Vue.options.components, platformComponents)

// install platform patch function
Vue.prototype.__patch__ = inBrowser ? patch : noop

// public mount method
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}

// devtools global hook
/* istanbul ignore next */
if (inBrowser) {
  setTimeout(() => {
    if (config.devtools) {
      if (devtools) {
        devtools.emit('init', Vue)
      } else if (
        process.env.NODE_ENV !== 'production' &&
        process.env.NODE_ENV !== 'test' &&
        isChrome
      ) {
        console[console.info ? 'info' : 'log'](
          'Download the Vue Devtools extension for a better development experience:\n' +
          'https://github.com/vuejs/vue-devtools'
        )
      }
    }
    if (process.env.NODE_ENV !== 'production' &&
      process.env.NODE_ENV !== 'test' &&
      config.productionTip !== false &&
      typeof console !== 'undefined'
    ) {
      console[console.info ? 'info' : 'log'](
        `You are running Vue in development mode.\n` +
        `Make sure to turn on production mode when deploying for production.\n` +
        `See more tips at https://vuejs.org/guide/deployment.html`
      )
    }
  }, 0)
}

export default Vue
```
正如vue项目结构中表示的那样，vue的核心代码肯定在在core文件夹，而这里就引入了该模块。我们粗略的看一下这里的注释，写的很明白了，这里是对vue核心部分的扩展，比如说添加一些指令、组件、工具方法之类的。最后让我们把关注点转移到`core/index`。
```
import Vue from './instance/index'
import { initGlobalAPI } from './global-api/index'
import { isServerRendering } from 'core/util/env'
import { FunctionalRenderContext } from 'core/vdom/create-functional-component'

initGlobalAPI(Vue)

Object.defineProperty(Vue.prototype, '$isServer', {
  get: isServerRendering
})

Object.defineProperty(Vue.prototype, '$ssrContext', {
  get () {
    /* istanbul ignore next */
    return this.$vnode && this.$vnode.ssrContext
  }
})

// expose FunctionalRenderContext for ssr runtime helper installation
Object.defineProperty(Vue, 'FunctionalRenderContext', {
  value: FunctionalRenderContext
})

Vue.version = '__VERSION__'

export default Vue
```
不按套路出牌啊，这里Vue又双叒叕引用的是其实例，好像没什么不对的样子。再来看`initGlobalAPI(Vue)`是嘛玩意儿,项目结构那里说了是公共API,具体是啥以后再看。后面的三个`Object.defineProperty()`都是ssr相关的，不管它。来来来，接着看看`./instance/index`
```
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue
```
卧槽，这么爽的吗？这个也太简洁了吧。仔细研究一下，声明了构造函数Vue,new Vue()时接受来自使用者的配置项，然后初始化Vue实例。下面的`***Mixin`方法应该就是给Vue添加基本的扩展用的，比如`_init`方法。然后我们来看看这几个`***Mixin`的顺序，简单的猜想其执行流程是这样的：**初始化vue->初始化state->初始化事件系统->初始化声明周期->开始渲染**。感觉流程很清晰呀。

再继续研究这几个mixin方法之前，我们先回到`core/index`，来看看`initGlobalAPI(Vue)`这个东东。
```
/* @flow */

import config from '../config'
import { initUse } from './use'
import { initMixin } from './mixin'
import { initExtend } from './extend'
import { initAssetRegisters } from './assets'
import { set, del } from '../observer/index'
import { ASSET_TYPES } from 'shared/constants'
import builtInComponents from '../components/index'

import {
  warn,
  extend,
  nextTick,
  mergeOptions,
  defineReactive
} from '../util/index'

export function initGlobalAPI (Vue: GlobalAPI) {
  // config
  const configDef = {}
  configDef.get = () => config
  if (process.env.NODE_ENV !== 'production') {
    configDef.set = () => {
      warn(
        'Do not replace the Vue.config object, set individual fields instead.'
      )
    }
  }
  Object.defineProperty(Vue, 'config', configDef)

  // exposed util methods.
  // NOTE: these are not considered part of the public API - avoid relying on
  // them unless you are aware of the risk.
  Vue.util = {          // 一些工具方法
    warn,
    extend,
    mergeOptions,
    defineReactive
  }

  Vue.set = set       // Vue.set()
  Vue.delete = del    // Vue.delete()
  Vue.nextTick = nextTick //Vue.nextTick()

  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {                       // ASSET_TYPES = ['component', 'directive', 'filter']
    Vue.options[type + 's'] = Object.create(null)
  })

  // this is used to identify the "base" constructor to extend all plain-object
  // components with in Weex's multi-instance scenarios.
  Vue.options._base = Vue

  extend(Vue.options.components, builtInComponents)

  initUse(Vue)      // Vue.use()
  initMixin(Vue)    // Vue.mixin()
  initExtend(Vue)   // Vue.extend()
  initAssetRegisters(Vue)  // Vue.component Vue.filter Vue.directive
}
```
可以很清楚的看到所有的vue的全局的api都可以在这里找到，比如`initUse(Vue)`就定义了我们使用组件常用的`Vue.use()`方法,当然还有其他的比如mixin、extend方法等。但是要注意的是全局component、filter、directive等方法是在`initAssetRegisters`中定义.下面是initUse(Vue)的源码：
```

import { toArray } from '../util/index'

export function initUse (Vue: GlobalAPI) {
  Vue.use = function (plugin: Function | Object) {
    const installedPlugins = (this._installedPlugins || (this._installedPlugins = []))
    if (installedPlugins.indexOf(plugin) > -1) {  // 已存在插件就不在添加
      return this
    }

    // additional parameters
    const args = toArray(arguments, 1)
    args.unshift(this)
    if (typeof plugin.install === 'function') {   // 是否实现了install方法
      plugin.install.apply(plugin, args)
    } else if (typeof plugin === 'function') {
      plugin.apply(null, args)
    }
    installedPlugins.push(plugin)                 // 安装插件
    return this
  }
}
```

好了我们现在对global api有了一个大致的了解，这里并不会详细的介绍每个api，以后使用到时才详细的介绍。下一章我们会详细的介绍Vue的详细的初始化过程，这章只是大致了解了整个项目的结构和简单的初始化流程。