# 浅析 v-node 

## 什么是虚拟DOM


>虚拟DOM 本质上是一个 **js 对象**，用于描述页面的结构，在 Vue 中，每个组件都有一个**render函数**，每一个 render 函数会返回一棵**虚拟DOM树**。

虚拟DOM 的部分属性：
``` 
  tag: string | void;
  data: VNodeData | void;
  children: ?Array<VNode>;
  text: string | void;
  elm: Node | void;
  ns: string | void;
  context: Component | void; // rendered in this component's scope
  key: string | number | void;
  componentOptions: VNodeComponentOptions | void;
  componentInstance: Component | void; // component instance
  parent: VNode | void; // component placeholder node
```
## 为什么需要虚拟DOM

> 1.当页面上的 **依赖数据** 变化时，组件重新渲染，重新运行**render函数**，得到一棵新的虚拟DOM树，将新旧两棵树进行对比，通过 **patch** 算法，做到最小量更新。
>
> 2.如果去操作真实的DOM，整个的DOM结构都得去创建，对真实DOM的创建、更新、插入是非常消耗性能的，降低渲染效率，用虚拟DOM代替真实DOM，解决了渲染效率的问题。


## 虚拟DOM是如何转换为真实DOM
>在一个组件实例首次被渲染时，它先生成虚拟DOM树，然后根据虚拟DOM树创建真实DOM，并把真实DOM挂载到页面中合适的位置，此时，每个虚拟DOM便会对应一个真实的DOM。
>如果一个组件受响应式数据变化的影响，需要重新渲染时，它仍然会重新调用render函数，创建出一个新的虚拟DOM树，用新树和旧树对比，通过对比，vue会找到最小更新量，然后更新必要的虚拟DOM节点，最后，这些更新过的虚拟节点，会去修改它们对应的真实DOM,这样一来，就保证了对真实DOM达到最小的改动。

