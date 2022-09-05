## 浅析 computed

:star: 计算属性就是对`data`里面的数据进行计算得到的一个新的属性。它依赖于用于计算的数据，模板近似于函数调用，那么它和函数有什么区别，内部又是怎么实现的，让我们一起来探索一下计算属性。

### 计算属性和函数有什么区别

1. 在使用时，`computed`可以作为属性，而函数则当做方法调用
2. `computed`可以配置`getter`和`setter`，因此可以赋值，而函数不行
3. `computed`无法接收多个参数，而函数可以
4. `computed`具有缓存，而函数没有

接下来深入了解一下 **computed** 是怎么实现的吧。

### 初始化computed
```js
function initComputed (vm: Component, computed: Object) {
  const watchers = vm._computedWatchers = Object.create(null)
  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    }
  }
}

function initWatch (vm: Component, watch: Object) {
  for (const key in watch) {
    const handler = watch[key]
    if (Array.isArray(handler)) {
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}
```
:rainbow:当初始化`computed`时，为每一个属性创建一个`Watcher`对象，通过`Object.defineProperty`配置`getter`，`getter`运行过程中就会收集依赖。

:star:但是计算属性的`Watcher`不会立即执行，要看模板中是否有使用到该计算属性，也就是依赖该计算属性，如果模板中没有使用，`Watcher`就不会去执行。由于，在给每个属性创建`Watcher`实例的时候，`Vue`配置了`lazy`，可以让`Watcher`不立即执行。

#### 实现缓存
Watcher内部维护两个属性用来做缓存：`value`和`dirty`
1. `value`属性用于保存Watcher运行的结果，受`lazy`的影响，`value` 在最开始是`undefined`
2. `dirty`属性用于指示当前的`value`是否为脏值，受`lazy`的影响，该值在最开始是`true`
```js
const computedWatcherOptions = { lazy: true }

function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```

### 工作流程
:star:当页面依赖计算属性时，vue首先检查其对应的`Watcher`中的`dirty`属性，判断该值是否是脏值，如果是，则运行`getter`进行计算，并得到对应的值，保存在`Watcher`的`value`中，然后把`Watcher`中的`dirty`改为`false`，然后返回。如果多次使用属性，也是先判断该值是否为脏值，如果不是脏值，直接读取`value`值，做到数据缓存。
:star:当计算属性的依赖数据变化时，就会先触发计算属性的`Watcher`运行，把`dirty`配置改为`true`，不做任何处理。当组件渲染的时候，重新读取计算属性，`dirty`为`true`，因此会重新运行`getter`进行计算、缓存。
