# Vue2 响应式原理

> vue官方阐述：https://cn.vuejs.org/v2/guide/reactivity.html

**响应式数据的最终目标**，是当对象本身或对象属性发生变化时，将会运行一些函数，最常见的就是`render`函数。

> 在具体的实现上，Vue 做了几个**核心部分**：
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
>todo
## Scheduler
>todo