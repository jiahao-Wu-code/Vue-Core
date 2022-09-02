# Vue2 响应式原理

:star: 参考vue官方阐述：https://v2.cn.vuejs.org/v2/guide/reactivity.html

**响应式数据的最终目标**，是当对象本身或对象属性发生变化时，将会运行一些函数，最常见的就是`render`函数。

在具体的实现上，Vue 做了几个**核心模块**：
1. Observer
2. Dep
3. Watcher
4. Scheduler

:star: 接下来让我们一起来探索一下吧！:rainbow:

## Observer

:rainbow:Observer要实现的目标非常简单，就是把一个普通的对象转换为响应式的对象
为了实现这一点，Observer把对象的每个属性通过`Object.defineProperty`转换为带有`getter`和`setter`的属性，这样一来，当访问或设置属性时，`vue`就可以做一些其他的事情。下面看一段代码:point_down:

```js
// 核心代码
Object.defineProperty(obj,key,{
    enumerable: true,
    configurable: true,
    get: function() {
        // 其他处理逻辑...
        return value;
    },
    set: function(newVal) {
        // 其他处理逻辑...
        val = newVal;
    },
})
```

:star: 在组件生命周期中，这件事发生在`beforeCreate`之后，`created`之前。
具体实现上，它会递归遍历对象的所有属性，以完成深度的属性转换。

:rainbow:由于遍历时只能遍历到对象的当前属性，因此无法监测到将来动态增加或删除的属性，因此vue提供了`$set`和`$delete`两个实例方法，让开发者通过这两个实例方法对已有响应式对象添加或删除属性。具体实现如下:point_down: 
```js
function set(target: Array<any> | Object, key: any, val: any){
    // 其他处理逻辑...
    target[key] = val;
    return val
}

function del (target: Array<any> | Object, key: any){
    // 其他处理逻辑...
    if (!hasOwn(target, key)) {
        return
    }
    delete target[key]
}

```

:star: 对于数组的处理，`vue`会更改它的隐式原型，之所以这样做，是因为vue需要监听那些可能改变数组内容的方法

```js
const arrayProto = Array.prototype
export const arrayMethods = Object.create(arrayProto)
const methodsToPatch = ['push','pop','shift','unshift','splice','sort','reverse']

methodsToPatch.forEach(function (method) {
  // cache original method
  const original = arrayProto[method]
  // Define a property.
  def(arrayMethods, method, function mutator (...args) {
    const result = original.apply(this, args)
    switch (method) {
      case 'push':
      case 'unshift':
        // 其他处理逻辑...
        break
      case 'splice':
        // 其他处理逻辑...
        break
    }
    // 其他处理逻辑...
    return result
  })
})
```
:rainbow: 总之，`Observer`设计的目的就是要让一个对象的属性的读取、赋值，内部数组的变化vue能够监测的到。


## Dep 
:star:当vue读取属性时要做什么处理，而属性变化时要做什么处理，这些问题需要借助**Dep**来解决。

Dep即`Dependency`，表示依赖的意思。

:star:在上面讲到的 `Observer` 会给每个属性去设置 `getter` 和 `setter`，这两个函数内部并不是简单的将值修改或返回，内部会实例化一个dep，每个`Dep`实例都有能力做以下两件事：

1. **记录依赖**：记录哪些地方用到了我
2. **派发更新**：我变了，需要通知那些用到我的地方

```js
class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  notify () {
    // 其他处理逻辑...
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}

``` 
:rainbow:当访问属性值时，运行`getter`,调用`dep.depend()`,递归收集依赖，保存在 `subs` 数组中
```js
if (Dep.target) {
    dep.depend()
    if (childOb) {
        childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
        }
    }
}
```
:rainbow:当修改属性值时，运行`setter`,调用`dep.notify()`,派发更新，通知用到的地方更新。


## Watcher
观察者（订阅者），当页面的依赖数据变化，需要去通知用到该数据的地方更新,通过`dep.notify()`派发更新，`watcher`就是用来做这个数据的订阅者。数据变化不会立即执行函数，而是将函数交给`watcher`，通过`dep.depend()`记录依赖，`Dep`上就会有记录，表示有一个`watcher`用到了这个数据。`watcher`内部需要自己实例化一个`dep`，当访问数据时把`watcher`添加到`dep`的`subs`中，调用 `update函数` 更新数据。
```js
class Watcher {
  constructor (vm, key, cb) {
    this.vm = vm

    // data 中的属性名称
    this.key = key
    // 当数据变化的时候，调用 cb 更新视图
    this.cb = cb
    // 在 Dep 的静态属性上记录当前 watcher 对象，当访问数据的时候把 watcher 添加到dep 的 subs 中
    Dep.target = this
    // 触发一次 getter，让 dep 为当前 key 记录 watcher
    this.oldValue = vm[key]
    // 清空 target
    Dep.target = null
  }
  update () {
    const newValue = this.vm[this.key]
    if (this.oldValue === newValue) {
      return
    }
    this.cb(newValue)
  }
}
```

## Scheduler

:rainbow: 调度器。当`dep`通知`watcher`去运行相应的函数时，有可能导致这个函数多次运行，换句话说就是同一个`watcher`添加了多次。为了解决这个问题，Vue做了一个`Scheduler`来处理这个问题。
```js
const MAX_UPDATE_COUNT = 100
// 定义每一个Watcher数组（队列）
const queue: Array<Watcher> = []

function queueWatcher(watcher: Watcher) {
  // 将 watcher添加到队列中，有相同id的watcher跳过
  const id = watcher.id
  if (has[id] != null) {
    return
  }

  if (watcher === Dep.target && watcher.noRecurse) {
    return
  }
  // 其他处理逻辑...
  queue.push(watcher)
  // 内部通过nextTick方法加到为队列
  nextTick(flushSchedulerQueue)
}
```

:star: 综上所述，`Scheduler` 内部维护一个`Watcher`的队列，该队列中的同一个`watcher`只会出现一次，并且`Watcher`队列不是立即去执行，而是通过`nextTick`方法将这些`watcher`放到微队列中依次执行。

## 总结

:rainbow:数据响应式首先是将数据通过`Object.defineProperty`把数据添加`getter`和`setter`，再通过`Dep`运行`getter`去收集依赖，运行`setter`去派发更新，`Watcher`接到通知，不立即执行函数，而是交给一个`Schedule`调度器去运行函数，做到数据、页面的更新。:star:
