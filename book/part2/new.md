## New

我们在上章简单的了解了Vue初始化的运行流程。还记得这段代码吗？位于`/src/core/instance/index.js`
```
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
```
可以看到这里简单的定义了一个函数Vue作为构造函数，这里为什么不使用ES6的class来定义成类Vue呢？我猜想是这样写可以很明确的告诉我们初始化做了什么以及其流程，代码可读性大大增强了。写成类的话需要把所有方法都定义好，反而可读性降低了。当然如果你也想像这样把需要定义的方法抽出来，但那样为啥不直接使用构造函数呢?

不说这些了，从这里可以看出来我们需要使用new关键字来实例化Vue对象，此时会调用`this._init(options)`,该方法在`initMixin`中定义了，来看看这个**迷信**方法吧。
```
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    // a uid
    vm._uid = uid++

    let startTag, endTag
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      startTag = `vue-perf-start:${vm._uid}`
      endTag = `vue-perf-end:${vm._uid}`
      mark(startTag)
    }

    // a flag to avoid this being observed
    vm._isVue = true
    // merge options
    if (options && optiowns._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(                                       // 合并options
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }
    // expose real self
    vm._self = vm
    initLifecycle(vm)                                                   // 初始化生命周期
    initEvents(vm)                                                      // 初始化事件系统
    initRender(vm)                                                      // 初始化渲染函数
    callHook(vm, 'beforeCreate')                                        // 生命周期钩子
    initInjections(vm) // resolve injections before data/props          // 初始化一些注入工具等
    initState(vm)                                                       // 初始化化state props 等
    initProvide(vm) // resolve provide after data/props                 // 初始化
    callHook(vm, 'created')                                             // 生命周期钩子

    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      vm._name = formatComponentName(vm, false)
      mark(endTag)
      measure(`vue ${vm._name} init`, startTag, endTag)
    }

    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```

可以看到这个_init方法所做的是合并配置，初始化生命周期，初始化事件中心，初始化渲染，初始化 data、props、computed、watcher 等等。

等等等等，所有初始化操作是在这里做的，那么那么多的mixin是做什么的？还记得我们前一章我们以为初始化生命周期、事件系统等都在initMixin之后做的？这里我们必须知道mixin(迷信)是干什么的了。mixin--混入（混入其中）所做的本质就是组合般的对对象进行扩展，不同于继承这种强调我是的做法，这种做法更加强调我能。你可以把它理解为插件的做法那么这里就很容易理解了。首先在initMixin中初始化事件系统，生命周期等，然后使用对应的mixin方法对这些功能进行扩展。

