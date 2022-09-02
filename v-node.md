# 浅析 v-node 
:rainbow: 本节我们来探索一下虚拟DOM，将会从以下4个方面进行阐述：

1. 虚拟DOM的本质
2. 虚拟DOM的优势
3. 虚拟DOM转换真实DOM 过程
4. 虚拟DOM树 diff算法

## 虚拟DOM的本质


:star:虚拟DOM 本质上是一个 **js 对象**，用于描述页面的结构，在 Vue 中，每个组件都有一个**render函数**，每一个 `render` 函数运行会返回一棵**虚拟DOM树**。

打印一下虚拟DOM:point_down:
![v-node](https://github.com/jiahao-Wu-code/Imags/blob/main/imgs/20220902094823.png?raw=true)

## 虚拟DOM的优势

1. 当组件的 **依赖数据** 变化时，该组件需要重新渲染，去重新运行**render函数**，得到一棵新的虚拟DOM树，将新旧两棵树进行对比，通过 **patch** 算法，做到最小量更新。
2. 如果每次渲染去对真实DOM的创建、更新、插入是非常消耗性能的，降低渲染效率，用虚拟DOM代替真实DOM，解决了渲染效率的问题。


## 虚拟DOM转换真实DOM 过程
1. 当组件首次渲染时，它会先生成虚拟DOM树，然后根据虚拟DOM树创建真实DOM，并把真实DOM挂载到页面中合适的位置，此时，每个虚拟DOM便会对应一个真实的DOM。(首次渲染，虚拟DOM优势不是很大，比操作真实DOM多了一个构建虚拟DOM树的过程，在后续更新过程中，优势变得更大。)
2. 如果组件不是首次渲染，依赖数据发生变化，则需要重新渲染，重新调用render函数，创建出一个新的虚拟DOM树，用新树和旧树对比，通过内部`patch`函数，vue会找到最小更新量，然后更新必要的虚拟DOM节点，最后，这些更新过的虚拟节点，会去修改它们对应的真实DOM,做一些属性上的更新，保证了树的稳定和最小量更新。

## 虚拟DOM树 diff 算法

1. `diff`发生的时机

   是在组件实例创建时，以及依赖的属性或数据变化时，会运行一个函数，该函数会做两件事：

   - 运行`_render`生成一棵新的虚拟dom树（vnode tree）
   - 运行`_update`，传入虚拟dom树的根节点，对新旧两棵树进行对比，最终完成对真实dom的更新

   核心代码如下：

   ```js
   // vue构造函数
   function Vue(){
     // ... 其他代码
     var updateComponent = () => {
       this._update(this._render())
     }
     // ... 其他代码
   }
   ```
   `diff`就发生在`_update`函数的运行过程中

   

2. `_update`函数在干什么

   `_update`函数接收到一个`vnode`参数，这就是**新**生成的虚拟dom树

   同时，`_update`函数通过当前组件的`_vnode`属性，拿到**旧**的虚拟dom树

   `_update`函数首先会给组件的`_vnode`属性重新赋值，让它指向新树

   然后会判断旧树是否存在：

   - 不存在：说明这是第一次加载组件，于是通过内部的`patch`函数，直接遍历新树，为每个节点生成真实DOM，挂载到每个节点的`elm`属性上
   - 存在：说明之前已经渲染过该组件，于是通过内部的`patch`函数，对新旧两棵树进行对比，以达到下面两个目标：

     - 完成对所有真实dom的最小化处理
     - 让新树的节点对应合适的真实dom
  
3. `patch`函数的对比流程

   1. **相同**：是指两个虚拟节点的标签类型、`key`值均相同，但`input`元素还要看`type`属性，具体实现如下:point_down:：
    ```js
    function sameVnode (a, b) {
        return (
            a.key === b.key && (
            (
                a.tag === b.tag &&
                a.isComment === b.isComment &&
                isDef(a.data) === isDef(b.data) &&
                sameInputType(a, b)
            ) || (
                isTrue(a.isAsyncPlaceholder) &&
                a.asyncFactory === b.asyncFactory &&
             isUndef(b.asyncFactory.error)
                )
            )
        )
    }
    function sameInputType (a, b) {
        if (a.tag !== 'input') return true
        let i
        const typeA = isDef(i = a.data) && isDef(i = i.attrs) && i.type
        const typeB = isDef(i = b.data) && isDef(i = i.attrs) && i.type
        return typeA === typeB || isTextInputType(typeA) && isTextInputType(typeB)
    } 
    ```
   2. **新建元素**：根据一个虚拟节点提供的信息，创建一个真实dom元素，并且挂载到该虚拟节点的`elm`属性上
   ``` js
      fucntion create(){
        // 其他代码... 
        // ns 表示 namespace
        vnode.elm = vnode.ns
        ? nodeOps.createElementNS(vnode.ns, tag)
        : nodeOps.createElement(tag, vnode)
      }
   ```
   1. **销毁元素**：通过`vnode.elm.remove()`方法进行销毁
   2. **更新**：两个虚拟节点进行对比更新，它**仅发生**在两个虚拟节点**相同**的情况下，做一些属性的更新
   3. **对比子节点**：两个虚拟节点的子节点进行对比
   ```js
   function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
    // 为新旧两棵树分别创建首位两个指针，遍历的同时做 删除和新建操作
    let oldStartIdx = 0 
    let newStartIdx = 0
    let oldEndIdx = oldCh.length - 1
    let newEndIdx = newCh.length - 1
    // 首先判断 key值，tag类型，再做相应的 创建 删除 更新操作
    // 其他代码....
    }
   ```

    **diff过程：**
    1. **根节点对比**
      `patch`函数先对根节点进行对比
      如果两个节点
      - **相同**，则进入**更新**流程
        1. 将旧节点的真实dom赋值到新节点：`newVnode.elm = oldVnode.elm`
        2. 对比新节点和旧节点的属性，有变化的更新到真实dom中
        3. 当前两个节点处理完毕，开始**对比子节点**
      - **不相同**
        1. 新节点**递归 新建元素**
        2. 旧节点**销毁元素**
    2.  **对比子节点**

      在**对比子节点**时，vue一切的出发点，都是为了最小量更新，尽量做到只是元素的移动，属性的更新，实在不行只能新建和删除元素。

