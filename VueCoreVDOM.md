# Vue VDOM源码阅读

VDOM是虚拟DOM技术，在内存中构造类似于DOM的节点、树结构，最终渲染成真实的DOM。VDOM的优点就是一切节点操作修改都在内存中进行，大大提升了DOM的操作速度，而且结构与真实的DOM树对应。



## 1.VDOM核心数据结构VNode

###1.1 VNode的数据结构定义

```js
export default class VNode {
  tag: string | void; // 标签名
  data: VNodeData | void; // 数据
  children: ?Array<VNode>; // 存放子节点引用
  text: string | void; // 文本内容
  elm: Node | void; // 关联的DOM元素
  ns: string | void; // 命名空间
  context: Component | void; // rendered in this component's scope // 上下文
  key: string | number | void; // 唯一标示
  componentOptions: VNodeComponentOptions | void; // 组件option
  componentInstance: Component | void; // component instance // 实例VM
  parent: VNode | void; // component placeholder node // 父节点引用

  // strictly internal
  raw: boolean; // contains raw HTML? (server only)
  isStatic: boolean; // hoisted static node
  isRootInsert: boolean; // necessary for enter transition check
  isComment: boolean; // empty comment placeholder? 增加空白注释占位，方便内容插值替换
  isCloned: boolean; // is a cloned node?
  isOnce: boolean; // is a v-once node?
  asyncFactory: Function | void; // async component factory function
  asyncMeta: Object | void; // 异步元信息
  isAsyncPlaceholder: boolean; // 异步占位
  ssrContext: Object | void; // 服务端渲染上下文
  fnContext: Component | void; // real context vm for functional nodes
  fnOptions: ?ComponentOptions; // for SSR caching
  devtoolsMeta: ?Object; // used to store functional render context for devtools
  fnScopeId: ?string; // functional scope id support
	
	// 构造函数中，传的内容并不多
	// (tag, data, children, text, elm, context, componentOptions, asyncFactory)
  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions,
    asyncFactory?: Function
  ) {
    this.tag = tag
    this.data = data
    this.children = children
    this.text = text
    this.elm = elm
    this.ns = undefined
    this.context = context
    this.fnContext = undefined
    this.fnOptions = undefined
    this.fnScopeId = undefined
    this.key = data && data.key
    this.componentOptions = componentOptions
    this.componentInstance = undefined
    this.parent = undefined
    this.raw = false
    this.isStatic = false
    this.isRootInsert = true
    this.isComment = false
    this.isCloned = false
    this.isOnce = false
    this.asyncFactory = asyncFactory
    this.asyncMeta = undefined
    this.isAsyncPlaceholder = false
  }
	
	// 兼容低版本
  // DEPRECATED: alias for componentInstance for backwards compat.
  get child (): Component | void {
    return this.componentInstance
  }
}
```

### 1.2 创建一个空白VNode

```js
export const createEmptyVNode = (text: string = '') => {
  const node = new VNode()
  node.text = text
  node.isComment = true // 增加comment空白节点占位
  return node
}
```

###1.3 创建文本VNode

```js
export function createTextVNode (val: string | number) {
  // (tag, data, children, text, elm, context, componentOptions, asyncFactory)
  // tag = undefined
  // data = undefined
  // children = undefined
  // text = String(val)
  return new VNode(undefined, undefined, undefined, String(val))
}
```

### 1.4 拷贝一个VNode

```js
// optimized shallow clone
// 拷贝节点为了方便节点的多次使用，以及防止错误产生的关联到其他节点
// 如果在不同的地方只使用同一个节点引用，在其中一个地方产生错误时候，将会影响所有
export function cloneVNode (vnode: VNode): VNode {
  const cloned = new VNode(
    vnode.tag,
    vnode.data,
    vnode.children && vnode.children.slice(), // 拷贝子节点数组，防止修改原始节点影响新节点
    vnode.text,
    vnode.elm,
    vnode.context,
    vnode.componentOptions,
    vnode.asyncFactory
  )
  cloned.ns = vnode.ns
  cloned.isStatic = vnode.isStatic
  cloned.key = vnode.key
  cloned.isComment = vnode.isComment
  cloned.fnContext = vnode.fnContext
  cloned.fnOptions = vnode.fnOptions
  cloned.fnScopeId = vnode.fnScopeId
  cloned.asyncMeta = vnode.asyncMeta
  cloned.isCloned = true // 这边手动修改该节点是拷贝获得的
  return cloned
}
```

