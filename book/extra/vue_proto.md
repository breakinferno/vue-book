# Vue原型方法整理

<!--sec data-title="Introduction" data-id="section0" data-collapse=true ces-->
```
// 1. initMixin(Vue)中定义，位于src/core/instance/init.js 
Vue.prototype._init = function (options?: Object) {}

// 2. stateMixin(Vue)中定义，位于src/core/instance/state.js 
Vue.prototype.$data
Vue.prototype.$props
Vue.prototype.$set = set
Vue.prototype.$delete = del
Vue.prototype.$watch = function (
  expOrFn: string | Function,
  cb: any,
  options?: Object
): Function {}

// 3. eventsMixin(Vue)中定义，位于src/core/instance/events.js 
Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {}
Vue.prototype.$once = function (event: string, fn: Function): Component {}
Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {}
Vue.prototype.$emit = function (event: string): Component {}

// 4. lifecycleMixin(Vue)中定义，位于src/core/instance/lifecycle.js
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {}
Vue.prototype.$forceUpdate = function () {}
Vue.prototype.$destroy = function () {}

// 5. renderMixin(Vue)中定义， 位于src/core/instance/render.js
Vue.prototype._o = markOnce
Vue.prototype._n = toNumber
Vue.prototype._s = toString
Vue.prototype._l = renderList
Vue.prototype._t = renderSlot
Vue.prototype._q = looseEqual
Vue.prototype._i = looseIndexOf
Vue.prototype._m = renderStatic
Vue.prototype._f = resolveFilter
Vue.prototype._k = checkKeyCodes
Vue.prototype._b = bindObjectProps
Vue.prototype._v = createTextVNode
Vue.prototype._e = createEmptyVNode
Vue.prototype._u = resolveScopedSlots
Vue.prototype._g = bindObjectListeners
Vue.prototype.$nextTick = function (fn: Function) {}
Vue.prototype._render = function (): VNode {}
```
<!--endsec-->