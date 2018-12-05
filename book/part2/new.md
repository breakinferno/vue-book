# 前言
我们使用最简单的情形来分析整个初始化流程。即：

```
<div id="app">{{message}}</div>

var vm = new Vue({
    el: '#app',
    data: {
        message: 'hello vue'
    }
})

```

## 开始

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
<!--sec data-title="initMixin" data-id="initMixin" data-collapse=true ces-->
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
<!--endsec-->

可以看到这个_init方法所做的是合并配置，初始化生命周期，初始化事件中心，初始化渲染，初始化 data、props、computed、watcher 等等。

等等,等等，所有初始化操作是在这里做的，那么那么多的mixin函数是做什么的？还记得我们前一章我们以为初始化生命周期、事件系统等都在initMixin之后做的？这里我们必须知道mixin(迷信)是干什么的了。mixin--混入（混入其中）所做的本质就是组合般的对对象进行扩展，不同于继承这种强调我是的做法，这种做法更加强调我能。你可以把它理解为插件的做法那么这里就很容易理解了。首先在initMixin中初始化事件系统，生命周期等，然后使用对应的mixin方法对这些功能进行扩展。

我们简单的对剩下的各个Mixin方法进行整理吧：

## 1. stateMixin
<!--sec data-title="stateMixin" data-id="stateMixin" data-collapse=true ces-->
```
export function stateMixin (Vue: Class<Component>) {
  const dataDef = {}
  dataDef.get = function () { return this._data }
  const propsDef = {}
  propsDef.get = function () { return this._props }
  if (process.env.NODE_ENV !== 'production') {
    dataDef.set = function () {
      warn(
        'Avoid replacing instance root $data. ' +
        'Use nested data properties instead.',
        this
      )
    }
    propsDef.set = function () {
      warn(`$props is readonly.`, this)
    }
  }
  Object.defineProperty(Vue.prototype, '$data', dataDef)
  Object.defineProperty(Vue.prototype, '$props', propsDef)

  Vue.prototype.$set = set
  Vue.prototype.$delete = del

  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
      // ...
  }
}
```
<!--endsec-->

## 2. eventsMixin(Vue)
<!--sec data-title="eventsMixin" data-id="eventsMixin" data-collapse=true ces-->
```
export function eventsMixin (Vue: Class<Component>) {
  const hookRE = /^hook:/
  Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {}
  Vue.prototype.$once = function (event: string, fn: Function): Component { }
  Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {}
  Vue.prototype.$emit = function (event: string): Component {}
```
原型定义了$set、$delete、$watch方法以及$data和$props属性(设置了getter,开发环境还设置了setter)，把这些属性方法记在小本本上。[附录在这里](/book/extra/vue_proto.md)
<!--endsec-->

## 3. lifecycleMixin(Vue)
<!--sec data-title="lifecycleMixin" data-id="lifecycleMixin" data-collapse=true ces-->
```
export function lifecycleMixin (Vue: Class<Component>) {
  Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const prevActiveInstance = activeInstance
    activeInstance = vm
    vm._vnode = vnode
    // Vue.prototype.__patch__ is injected in entry points
    // based on the rendering backend used.
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    activeInstance = prevActiveInstance
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    // if parent is an HOC, update its $el as well
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
    // updated hook is called by the scheduler to ensure that children are
    // updated in a parent's updated hook.
  }

  Vue.prototype.$forceUpdate = function () {
    const vm: Component = this
    if (vm._watcher) {
      vm._watcher.update()
    }
  }

  Vue.prototype.$destroy = function () {
    const vm: Component = this
    if (vm._isBeingDestroyed) {
      return
    }
    callHook(vm, 'beforeDestroy')
    vm._isBeingDestroyed = true
    // remove self from parent
    const parent = vm.$parent
    if (parent && !parent._isBeingDestroyed && !vm.$options.abstract) {
      remove(parent.$children, vm)
    }
    // teardown watchers
    if (vm._watcher) {
      vm._watcher.teardown()
    }
    let i = vm._watchers.length
    while (i--) {
      vm._watchers[i].teardown()
    }
    // remove reference from data ob
    // frozen object may not have observer.
    if (vm._data.__ob__) {
      vm._data.__ob__.vmCount--
    }
    // call the last hook...
    vm._isDestroyed = true
    // invoke destroy hooks on current rendered tree
    vm.__patch__(vm._vnode, null)
    // fire destroyed hook
    callHook(vm, 'destroyed')
    // turn off all instance listeners.
    vm.$off()
    // remove __vue__ reference
    if (vm.$el) {
      vm.$el.__vue__ = null
    }
    // release circular reference (#6759)
    if (vm.$vnode) {
      vm.$vnode.parent = null
    }
  }
}
```
<!--endsec-->
把这些属性方法记在小本本上。[附录在这里](/book/extra/vue_proto.md)
## 4. renderMixin(Vue)
<!--sec data-title="RenderMixin" data-id="RenderMixin" data-collapse=true ces-->

```
export function renderMixin (Vue: Class<Component>) {
  // install runtime convenience helpers
  installRenderHelpers(Vue.prototype)

  Vue.prototype.$nextTick = function (fn: Function) {
    return nextTick(fn, this)
  }

  Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const { render, _parentVnode } = vm.$options

    if (_parentVnode) {
      vm.$scopedSlots = _parentVnode.data.scopedSlots || emptyObject
    }
    vm.$vnode = _parentVnode
    // ...... 省略部分代码
    // set parent
    vnode.parent = _parentVnode
    return vnode
  }
}

// installRenderHelpers(Vue.prototype)
export function installRenderHelpers (target: any) {
  target._o = markOnce
  target._n = toNumber
  target._s = toString
  target._l = renderList
  target._t = renderSlot
  target._q = looseEqual
  target._i = looseIndexOf
  target._m = renderStatic
  target._f = resolveFilter
  target._k = checkKeyCodes
  target._b = bindObjectProps
  target._v = createTextVNode
  target._e = createEmptyVNode
  target._u = resolveScopedSlots
  target._g = bindObjectListeners
}
```
<!--endsec-->
把这些属性方法记在小本本上。[附录在这里](/book/extra/vue_proto.md)

总结： new Vue实例之前使用各个Mixin方法对Vue构造函数进行扩展包装，全局挂载了一些属性或者方法，new Vue实例时才开始进行初始化操作。

下面我们一个一个来分析`_init()`中具体做了啥。

