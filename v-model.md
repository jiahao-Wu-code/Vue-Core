## 浅析 v-model

v-model 是在表单、`input`、`textarea` 及 `select `元素上创建双向数据绑定的语法糖。而数据的双向绑定无疑是在元素(input 、textarea、select)上绑定了一个change(input)事件，来对`model`和`view`进行动态的修改.

:rainbow: 接下来让我们一起来探索一下吧！

其实数据双向绑定和数据响应式的实现没有太大的区别，都是监听数据的变化，派发更新。
可以参考一下这篇文章:point_right:[猛戳这里](https://blog.csdn.net/Yi_CO_coding/article/details/126670664?spm=1001.2014.3001.5502)

### 思路整理
要实现数据双向绑定，首先要将数据变成响应式，通过Object.defineProperty来实现对每个属性的劫持。
需要做到以下几点：
1. 实现一个数据监听器 Observer，对对象的每个属性进行监听，当读取属性就收集依赖，当数据变化则通知订阅者。
2. 实现一个订阅器 Dep，当使用属性，去收集依赖；属性变化就通知订阅者。
3. 实现一个指令解析器Compile，对每个元素节点的指令进行扫描和解析，根据指令模板替换数据，以及绑定相应的更新函数。
4. 实现一个Watcher，作为连接Observer和Compile的桥梁，能够订阅并收到每个属性变动的通知，执行指令绑定的相应回调函数，从而更新视图。

具体实现如下:point_down::

#### Observer
:rainbow:要实现的目标非常简单，就是把一个普通的对象转换为响应式的对象
为了实现这一点，Observer把对象的每个属性通过`Object.defineProperty`转换为带有`getter`和`setter`的属性
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

#### Dep
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

#### Complie
compile主要做的事情是解析模板指令，将模板中的变量替换成数据，然后初始化渲染页面视图，并将每个指令对应的节点绑定更新函数，添加监听数据的订阅者，一旦数据有变动，收到通知，更新视图，


#### Watcher
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