# 一些话
上节讲了初始化过程中的几个init方法，比如initLifecyle、initState等，但我们注意到Vue初始化时也提供了一些钩子函数，具体的表现在于：`callHook(vm, 'beforeCreate')`和`callHook(vm, 'created')`，即创建之前钩子函数和创建之后钩子函数
# 代码
```
export function callHook (vm: Component, hook: string) {
  pushTarget()
  const handlers = vm.$options[hook]
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      try {
        handlers[i].call(vm)
      } catch (e) {
        handleError(e, vm, `${hook} hook`)
      }
    }
  }
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook)
  }
  popTarget()
}
```

pushTarget和popTarget在初始化时没有什么用。这个代码逻辑很简单，就是如果用户传入options中有对应的beforeCreate和created回调函数就会在这里被执行。比如:
```
new Vue({
    el: '#app',
    beforeCreate: function(){
        // code here
    },
    created: function() {
        // code here
    }
})
```
值得注意的是可以传入函数数组。。这个我没有想到啊（官网也没说）。还有一点事这里使用call来绑定this对象，那么官网上说的不要使用箭头函数就可以理解了。