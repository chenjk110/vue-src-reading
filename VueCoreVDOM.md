# Vue VDOM源码阅读

VDOM是虚拟DOM技术，在内存中构造类似于DOM的节点、树结构，最终渲染成真实的DOM。VDOM的优点就是一切节点操作修改都在内存中进行，大大提升了DOM的操作速度，而且结构与真实的DOM树对应。



## 1.VDOM核心数据结构VNode

### 1.1 VNode相关的数据接口定义

```ts
declare type VNodeChildren = Array<?VNode | string | VNodeChildren> | string;

// 定义了VNode的组件配置数据结构
declare type VNodeComponentOptions = {
  Ctor: Class<Component>; // 组件的构造函数
  propsData: ?Object; // 组件的props配置
  listeners: ?Object; // 组件的事件监听
  children: ?Array<VNode>; // 子节点
  tag?: string; // 标签名
};

// 挂在后的组件VNode
declare type MountedComponentVNode = {
  context: Component; // 拥有具体的组件上下文
  componentOptions: VNodeComponentOptions; // 组件的option配置
  componentInstance: Component; // 组件引用
  parent: VNode; // 父级引用
  data: VNodeData; // data数据
};

// interface for vnodes in update modules
declare type VNodeWithData = {
  tag: string;
  data: VNodeData;
  children: ?Array<VNode>;
  text: void;
  elm: any;
  ns: string | void;
  context: Component;
  key: string | number | void;
  parent?: VNodeWithData;
  componentOptions?: VNodeComponentOptions;
  componentInstance?: Component;
  isRootInsert: boolean;
};

// VNode指令结构
declare type VNodeDirective = {
  name: string; // 不包含v-指令名称， 如：bind，for等
  rawName: string; // 包含v-的完整指令名，如：v-band，v-for等
  value?: any; // 
  oldValue?: any;
  arg?: string;
  oldArg?: string;
  modifiers?: ASTModifiers; // 修饰符v-on:click.stop, stop就是修饰符, {[key string]: bolean}
  def?: Object;
};

declare type ScopedSlotsData = Array<{ key: string, fn: Function } | ScopedSlotsData>;

```

### 1.2 了解VNode的data属性的接口定义

```ts
// 
declare interface VNodeData {
  key?: string | number;
  slot?: string;
  ref?: string;
  is?: string;
  pre?: boolean;
  tag?: string;
  // 静态类名，不参与动态修改的，如：:class="['a', isTrue && 'b']"
  // 其中，a为静态类名
  staticClass?: string; 
  class?: any; // 参与动态计算的类名，可能是函数，对象等等
  // 静态样式，与类名同理
  staticStyle?: { [key: string]: any };
  style?: string | Array<Object> | Object; // 参与动态计算的样式，可能是函数，对象等等
  normalizedStyle?: Object; // 最终合并后的style对象
  props?: { [key: string]: any }; // 定义props的属性
  attrs?: { [key: string]: string }; // 非定义props的传入的属性
  domProps?: { [key: string]: any }; // 挂在dom上的原生props属性，
  hook?: { [key: string]: Function }; // 钩子，生命周期钩子，可用在transition等场景
  on?: ?{ [key: string]: Function | Array<Function> }; // 事件注册对象
  nativeOn?: { [key: string]: Function | Array<Function> }; // 原生事件注册对象
  transition?: Object; // 动画配置相关
  show?: boolean; // marker for v-show
  inlineTemplate?: { // 行内模版
    render: Function;
    staticRenderFns: Array<Function>;
  };
  directives?: Array<VNodeDirective>; // 指令
  keepAlive?: boolean; // 实例长存
  scopedSlots?: { [key: string]: Function };
  model?: {
    value: any;
    callback: Function;
  };
};
```



###1.3 VNode的数据结构定义

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

## 2 VDOM处理相关模块Modules

###2.1创建VNode的相关DOM引用ref

```js
import { remove, isDef } from 'shared/util'

// 注册一个Ref引用
// 其中vnode为目标vnode节点，isRemoval判断是否为删除操作，也就是去除ref关联
export function registerRef (vnode: VNodeWithData, isRemoval: ?boolean) {
  
  const key = vnode.data.ref
  if (!isDef(key)) return // 该组件先前未有ref绑定，则直接返回

  const vm = vnode.context // 获取上下文，也就是vnode关联的vue实例
  const ref = vnode.componentInstance || vnode.elm // 具体的dom节点引用
  const refs = vm.$refs // 实例中存放dom节点引用map
	
  // 删除关联操作
  if (isRemoval) {
    // 在多个组件上使用相同的ref属性值，会生成对应的数组
    // <comp-a ref="comps" />
    // <comp-b ref="comps" />
    // this.$refs['comps'] -> [{...}, {...}]
    if (Array.isArray(refs[key])) {
      // 在数组中删除对应的vnode
      remove(refs[key], ref)
    } else if (refs[key] === ref) {
      // 单个引用，则直接设为undefeated
      refs[key] = undefined
    }
    
  // 注册、修改操作
  } else {
    // 在for循环中的ref引用
    if (vnode.data.refInFor) {
      if (!Array.isArray(refs[key])) {
        // 将对应的ref构造成数组，并存入ref引用
        refs[key] = [ref]
      } else if (refs[key].indexOf(ref) < 0) {
        // 增加，直接push
        refs[key].push(ref)
      }
    } else {
      // 不在for循环中，则直接赋值，完成引用关联
      refs[key] = ref
    }
  }
}

// 定义了对象包含了一下三个方法用来操作VNode的ref属性
export default {
  // VNodeWithData标明该VNode具有实际效用且具有关联性的VNode，context、data等具有真实数据
  // 创建引用
  create (_: any, vnode: VNodeWithData) {
    registerRef(vnode)
  },
  // 更新引用
  update (oldVnode: VNodeWithData, vnode: VNodeWithData) {
    // 判断两个ref是否相等，避免多余操作
    if (oldVnode.data.ref !== vnode.data.ref) {
      registerRef(oldVnode, true) // 删除原始引用
      registerRef(vnode) // 注册新引用
    }
  },
  // 删除引用
  destroy (vnode: VNodeWithData) {
    registerRef(vnode, true)
  }
}
```

