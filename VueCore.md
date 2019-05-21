# Vue Core目录源码阅读

[TOC]



## 1. 从入口index.js开始

```js
/* 引入了Vue实例相关 */
import Vue from './instance/index'
/* 全局API */
import { initGlobalAPI } from './global-api/index'
/* ssr环境识别 */
import { isServerRendering } from 'core/util/env'
/* ssr纯函数组建渲染上下文 */
import { FunctionalRenderContext } from 'core/vdom/create-functional-component'

initGlobalAPI(Vue) // 初始化全局API

/* 注入$isServer变量到原型 */
Object.defineProperty(Vue.prototype, '$isServer', {
  get: isServerRendering
})

/* 注入$ssrContext的getter到原型 */
Object.defineProperty(Vue.prototype, '$ssrContext', {
  get () {
    /* istanbul ignore next */
    return this.$vnode && this.$vnode.ssrContext
  }
})

// expose FunctionalRenderContext for ssr runtime helper installation
/* 在Vue挂在ssr环境下的functional组建渲染 */
Object.defineProperty(Vue, 'FunctionalRenderContext', {
  value: FunctionalRenderContext
})

// 定义Vue的版本信息
Vue.version = '__VERSION__'

export default Vue

```

从core/index.js入口文件，可以发现在core/instance基本已经完成了Vue的构造函数相关功能注入，这边最主要的是initGlobalAPI()方法，该方法将注入Vue下的全局静态方法，例如Vue.use(), Vue.extend(), Vue.mixin()等。这边先了解个大概，我们先看下同级目录下的config文件，配置相关，从这边入手可先了解整体得配置项有哪些，以便了解哪些功能是可配置的。

## 2. 了解配置项config

全局配置项挂在在Vue.config下，查看官方API文档可以了解到，Vue.config是个对象，并且这个对象的配置信息将会在全局生效。配置项有默认值，这是官方预先考虑到可能的使用场景或者更具普适性的配置。先来看下官方API文档中，对于Vue.config有哪些可配置项：



Global Config:

1.silent  # 是否关闭一切log、warning提示信息，设置为true则关闭

2.optionMergeStrategies # 配置项合并策略，

3.devtools # 开发者工具，是否使用开发者工具，默认true，在生产环境下为false

4.errorHandler # 定义全局错误处理函数

5.warnHandler # 定义全局提醒处理函数

6.ignoredElement # 定义忽略的组建标签名数组，主要为利用WebCompoent实现的自定义组件

7.keyCodes # 定义自定义按键事件名和keyCode的绑定

8.performance # 是否开启性能检测，在开发模式下配合DevTools使用

9.productionTip # 是否关闭在生产模式下的提示信息



以上为API文档中的全局配置参数，接下来我们来看下config文件中的具体内容：

```js
export default ({
  optionMergeStrategies: Object.create(null),
  silent: false,
  productionTip: process.env.NODE_ENV !== 'production',
  devtools: process.env.NODE_ENV !== 'production',
  performance: false,
  errorHandler: null,
  warnHandler: null,
  ignoredElements: [],
  keyCodes: Object.create(null),

  /**
   * Check if a tag is reserved so that it cannot be registered as a
   * component. This is platform-dependent and may be overwritten.
   */
  // 检查预留标签名
  isReservedTag: no,

  /**
   * Check if an attribute is reserved so that it cannot be used as a component
   * prop. This is platform-dependent and may be overwritten.
   */
  // 检查预留属性名
  isReservedAttr: no,

  /**
   * Check if a tag is an unknown element.
   * Platform-dependent.
   */
  // 检查未知标签
  isUnknownElement: no,

  /**
   * Get the namespace of an element
   */
  // 获取组件的命名空间
  getTagNamespace: noop,

  /**
   * Parse the real tag name for the specific platform.
   */
  parsePlatformTagName: identity,

  /**
   * Check if an attribute must be bound using property, e.g. value
   * Platform-dependent.
   */
  // 是否强制将属性名在props中定义，value等web标准下的属性名可自动获取到，取决平台
  mustUseProp: no,

  /**
   * Perform updates asynchronously. Intended to be used by Vue Test Utils
   * This will significantly reduce performance if set to false.
   */
  // 异步更新渲染
  async: true,

  /**
   * Exposed for legacy reasons
   */
  // 暴露出生命周期钩子函数名，为了兼容
  _lifecycleHooks: LIFECYCLE_HOOKS
}: Config)
```

可以看出，源码当中的config文件仍有一些配置项在API文档中未体现出来，这边原始的注释有提到Platform-dependent.说明不同的平台会有不同的需求，这边暂时知道有这些区别。

了解了配置项信息，我们知道配置在整个Vue的组合过程中，必定是全局使用到，因为涉及到不同模块的配置、功能开关等。了解了都有哪些配置项，我们边继续做一下前置的预备阅读：了解core/utils文件夹。阅读其他模块前先了解相关的utils是个不错的选择，因为utils里面的方法是独立解耦，且实现的功能目的简单，有利于后面其他模块的源码阅读。

所以，接下来就来了解下core/utils文件夹都有什么黑科技。

## 3. 预热准备，阅读core/utils工具集

先来看下该文件夹下都有哪些：

core/utils:

1.index.js # 入口文件

2.debug.js # 调试相关

3.env.js # 环境变量

4.error.js # 错误处理相关

5.lang.js # HTML语言相关

6.next-tick.js # 事件轮询相关

7.options.js # 配置项处理相关

8.perf.js # 性能检测相关

9.props.js # props属性处理相关



这边index入口文件没有其他的过多处理，就是简单的模块聚合，我们先从简单的env.js文件入手，从个文件可以了解到Vue的环境检测、Web API能力检测等一些方法、了解了env的内容，也对于其他使用到的代码阅读有所帮助，那么让我们详细看下吧。

### 3.1 env.js 环境检测

开门见山，直接看源码：

```js
export const hasProto = '__proto__' in {}
// 判断是否能够直接使用__proto__对象属性
// 这边直接用in操作符去判断Object的实例是否存在__proto__

export const inBrowser = typeof window !== 'undefined'
// 浏览器环境嗅探，直接找window

export const inWeex = typeof WXEnvironment !== 'undefined' && !!WXEnvironment.platform
// Weex环境嗅探，找Weex的全局变量WXEnvironment

export const weexPlatform = inWeex && WXEnvironment.platform.toLowerCase()
// 准确获取Weex的所在平台信息，利用上一步的isWeex先确定在weex平台，然后获取平台信息小写转换

export const UA = inBrowser && window.navigator.userAgent.toLowerCase()
// 在浏览器模式下，找到对应的UA信息，转成小写

// Web 浏览器内核检测，这边有很多方法，大致就是找到内核名称和内核版本
export const isIE = UA && /msie|trident/.test(UA)
export const isIE9 = UA && UA.indexOf('msie 9.0') > 0
export const isEdge = UA && UA.indexOf('edge/') > 0
export const isChrome = UA && /chrome\/\d+/.test(UA) && !isEdge
export const isFF = UA && UA.match(/firefox\/(\d+)/)
export const isPhantomJS = UA && /phantomjs/.test(UA) // 可利用于测试环境下

// Weex环境下，OS平台信息的检测，目前只提供了Android和iOS
export const isAndroid = (UA && UA.indexOf('android') > 0) || (weexPlatform === 'android')
export const isIOS = (UA && /iphone|ipad|ipod|ios/.test(UA)) || (weexPlatform === 'ios')

export const nativeWatch = ({}).watch
// Firefox浏览器实现了自家的一个watch、unwatch方法，用来查看对象属性名的属性值变化
// 参考资料：https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/watch

// 定义是否支持passive属性
export let supportsPassive = false
// 对于事件绑定addEventListener(type, handler, [options: object | isCapture: boolean])
// 第三个参数可以是boolean，也可以是一个option对象，旧版中为boolean，所以需要检测是否支持opiton传递
// 并且支持passive字段，其中passive字段表示在捕获阶段触发回调，与isCaputer=true效果一致
// 参考资料：https://developer.mozilla.org/zh-CN/docs/Web/API/EventTarget/addEventListener
if (inBrowser) {
  try {
    const opts = {}
    Object.defineProperty(opts, 'passive', ({
      get () {
        supportsPassive = true
      }
    }: Object)) // https://github.com/facebook/flow/issues/285
    // 这边直接利用try-catch去做passive的能力检测，没有异常则修改前面的supportsPassive = true
    window.addEventListener('test-passive', null, opts)
  } catch (e) {}
}

// this needs to be lazy-evaled because vue may be required before
// vue-server-renderer can set VUE_ENV
let _isServer
export const isServerRendering = () => {
  if (_isServer === undefined) {
    // 检测在Node环境下
    if (!inBrowser && !inWeex && typeof global !== 'undefined') {
      // 做成一个方法，在使用时判断，防止webpack打包时候注入global和process属性
      _isServer = global['process'] && global['process'].env.VUE_ENV === 'server'
    } else {
      _isServer = false
    }
  }
  return _isServer
}

// 检测开发者工具，这边 window.__VUE_DEVTOOLS_GLOBAL_HOOK__ 是跟devtool做了约定
export const devtools = inBrowser && window.__VUE_DEVTOOLS_GLOBAL_HOOK__

// 判断函数或者构造函数是否是内建原生函数，在调用toString()时候
// 会返回包含native code 字段信息的内容
// 如：Object.toString() -> 'function Object() { [native code] }'
export function isNative (Ctor: any): boolean {
  return typeof Ctor === 'function' && /native code/.test(Ctor.toString())
}

// symbol检测
export const hasSymbol =
  typeof Symbol !== 'undefined' && isNative(Symbol) &&
  typeof Reflect !== 'undefined' && isNative(Reflect.ownKeys)

let _Set
if (typeof Set !== 'undefined' && isNative(Set)) {
	// 原生支持Set
  _Set = Set
} else {
	// Set的polyfill
  _Set = class Set implements SimpleSet {
    set: Object;
    constructor () {
      this.set = Object.create(null)
    }
    has (key: string | number) {
      return this.set[key] === true
    }
    add (key: string | number) {
      this.set[key] = true
    }
    clear () {
      this.set = Object.create(null)
    }
  }
}

// Flow格式的接口定义，不参与打包，在检测到环境不支持Set的时则使用polyfill
interface SimpleSet {
  has(key: string | number): boolean;
  add(key: string | number): mixed;
  clear(): void;
}

export { _Set }
export type { SimpleSet }

```

阅读完这个文件，对于一些能力检测还是有必要的，毕竟原生实现的性能和规范的优势都大于polyfill。

### 3.2 开发模式下的辅助debug

```js
/* 引入相关依赖 */
import config from '../config'
import { noop } from 'shared/util'

// 定义warn、tip处理方法，初始化为noop
export let warn = noop
export let tip = noop

// 生成组件层级追踪信息，详细实现阅读后面代码
export let generateComponentTrace = (noop: any)
// 组件名称格式化，详细实现阅读后面代码
export let formatComponentName = (noop: any)

// 在非生产环境中
if (process.env.NODE_ENV !== 'production') {
  // 判断环境是否支持console
  const hasConsole = typeof console !== 'undefined'
  
  // 利用(?:)非捕获分组，可能的匹配字符串: 'a', 'a-b' -> 'b', 'a_b' -> 'b' 
  const classifyRE = /(?:^|[-_])(\w)/g
  
  // 定义转换类名函数,主要将'-_'连接的类名字符串转成驼峰形式
  // 如： 'a-b' -> 'aB', 'a_bc-de' -> 'aBcDe'
  const classify = str => str
    .replace(classifyRE, c => c.toUpperCase())
    .replace(/[-_]/g, '')
	
  // 定义warn，其中：msg代表需要打印的信息，vm代表所绑定的Vue实例上下文
  warn = (msg, vm) => {
    // 若存在vm实例，
    // 则利用generateComponentTrace生成具体的组件层级trace追踪信息
    // 否则为''
    const trace = vm ? generateComponentTrace(vm) : ''
		
    // 使用config中定义的warn处理函数
    if (config.warnHandler) {
      config.warnHandler.call(null, msg, vm, trace)
     
    // 这边利用默认的console.error打印信息
    } else if (hasConsole && (!config.silent)) {
      console.error(`[Vue warn]: ${msg}${trace}`)
    }
  }
	
  // 定义tip，其中：msg代表需要打印的信息，vm代表所绑定的Vue实例上下文
  // 实现方法参考上文中warn
  tip = (msg, vm) => {
    if (hasConsole && (!config.silent)) {
      console.warn(`[Vue tip]: ${msg}` + (
        vm ? generateComponentTrace(vm) : ''
      ))
    }
  }
	
  // 格式化组件名称，利用实例中的options查询
  formatComponentName = (vm, includeFile) => {
    // 若是根实例，则直接返回'<Root>'
    // vm.$root保存着这个应用实例的根组件引用，若等于vm则当前vm等于根组件
    if (vm.$root === vm) {
      return '<Root>'
    }
    // vm.$root保存着整个组件层级的根实例, 非根实例会通过上级实例，向上递归获得最终的根实例
    // 逻辑为：vm.$root = parent ? parent.$root : vm
    
    // 获取options，即new Vue(options)后传入的options于原始Vue的options的混合opitons
    // 这边可以到instance/initMixin.js中的_init方法具体了解初始化过程
    
    // 判断vm是否是个组件的构造函数，组件构造函数必有cid属性,
    const options = typeof vm === 'function' && vm.cid != null
			// 若是，则获取挂在在构造函数的配置，vm.options
    	? vm.options
    	// 非构造函数，判断是否为Vue默认实例,而非VNode
      : vm._isVue
    		// 获取挂在默认Vue实例的配置或者构造函数的配置
        ? vm.$options || vm.constructor.options
        // 返回组件实例
        : vm
    // 名称获取，优先使用配置中name属性，没有则获取组件的标签名
    let name = options.name || options._componentTag
    // 获取配置中的__file字段，该字段用来保存组件文件的实际路径
    const file = options.__file

    // 若获取不到name信息, 且文件路径存在，则匹配文件路径信息
    if (!name && file) {
      const match = file.match(/([^/\\]+)\.vue$/)
      name = match && match[1]
    }
		
    // 返回组件名
    return (
      (name ? `<${classify(name)}>` : `<Anonymous>`) + // 那么不存在则为匿名组件
      (file && includeFile !== false ? ` at ${file}` : '') // 存在file路径则附加上去
    )
  }
	
  // 字符串重复方法
  // res += res 运算后字符串内容x2，如： 'a' -> 'aa'
  // n >>= 1 向右位移动一位则/2, 如： 4 -> 2
  // 循环直到n === 0
  const repeat = (str, n) => {
    let res = ''
    while (n) {
      if (n % 2 === 1) res += str
      if (n > 1) str += str
      n >>= 1
    }
    return res
  }
	
  // 生成组件信息层级追踪内容
  generateComponentTrace = vm => {
    // vm是vue实例并且具有父级实例引用
    if (vm._isVue && vm.$parent) {
      const tree = [] // 用来保存组件树每个层级的vm
      let currentRecursiveSequence = 0 // 定义当前vm的递归次数
      
      while (vm) {
        // 第一次，tree为空则跳过
        if (tree.length > 0) {
          
          // 获取tree最后一个，也就是上次while循环添加的vm，当前vm的字组件
          const last = tree[tree.length - 1]
          
          // 当前vm与上一个vm是同样的一个组件
          if (last.constructor === vm.constructor) {
            
            // 递归次数+1
            // 为了记录当前组件实例递归引用了几次，方便后面重复信息输出
            currentRecursiveSequence++
            
            // 设置把当前vm的父级实例赋值于vm，继续向上遍历
            // （这是一种常见的层级向上递归遍历方式）
            vm = vm.$parent
            continue
            
          // 当前vm与上一个vm不是相同组件
          // 判断是否需要递归重复上一个组件信息
          } else if (currentRecursiveSequence > 0) {
            
            // 此时tree的最后一个元素数据结构变成 [..., vm] -> [..., [vm, n]] (n为重复次数)
            tree[tree.length - 1] = [last, currentRecursiveSequence]
            currentRecursiveSequence = 0 // 计数清零
          }
        }
        
        // 将当前vm添加到tree数组
        tree.push(vm)
        
        // 设置vm为当前vm的父组件，实现向上递归查询
        vm = vm.$parent
      }
      
      // 实现tree树层级信息打印
      return '\n\nfound in\n\n' + tree
        .map((vm, i) => `${
          i === 0 ? '---> ' : repeat(' ', 5 + i * 2) // 按照层级增加，添加前置空格缩进
        }${
          // 生成组件名称层级信息
          // 若是递归调用，则提示递归调用的次数
          Array.isArray(vm)
            ? `${formatComponentName(vm[0])}... (${vm[1]} recursive calls)`
            : formatComponentName(vm)
        }`)
        .join('\n')
    } else {
      // 没有父级组件引用，则说明该vm为单纯的js实例
      return `\n\n(found in ${formatComponentName(vm)})`
    }
  }
}

```

### 3.3 了解错误处理机制error

在core/utils/error.js文件中，主要有几个函数：

1.handleError # 错误处理

2.invokeWithErrorHandling # 关联错误处理

3.logError # 打印输出错误信息

4.globalHandleError # 全局错误处理



那么，接下来我们先从打印错误日志logError函数开始，这个函数主要是输出当前vm下产生的错误信息：

```js
// 三个参数，分别是异常错误err，组件实例vm， 提示信息info
function logError (err, vm, info) {

  if (process.env.NODE_ENV !== 'production') {
    // 前面debug中定义的warn是使用了console.error进行错误输出，
    // 然后通过传入的vm进行组件信息向上递归
    warn(`Error in ${info}: "${err.toString()}"`, vm)
  }
	// 若在web、weex环境下和console可用，输出原始错误信息
  // 否则抛出异常
  if ((inBrowser || inWeex) && typeof console !== 'undefined') {
    console.error(err)
  } else {
    throw err
  }
}

```

有了前面的错误信息输出方法，接着，我们来了解下全局错误处理globalHandleError

```js
function globalHandleError (err, vm, info) {
  // 若是在全局配置config中配置了自定义的errorHanler方法
  // 则使用该方法进行错误处理，这边使用了try-catch进行保险处理，捕捉自定义处理方法可能的异常
  if (config.errorHandler) {
    try {
      return config.errorHandler.call(null, err, vm, info) // 调用自定义的错误处理
    } catch (e) {
			// 因为前面的logError也可能会抛出错误，所以这边进行错误判断
      // 当catch的错误e不为当前处理的错误err，则直接打印捕获的错误e，防止打印err两次
      if (e !== err) {
        logError(e, null, 'config.errorHandler')
      }
    }
  }
  
  // 没有设置errorHandler，则输出错误err
  logError(err, vm, info)
}

```

接下来了解错误处理关联方法：

```js
export function invokeWithErrorHandling (
  handler: Function, // 传入的具体处理方法
  context: any, // 上下文, 为handler绑定this用
  args: null | any[], // bind时候的具体参数
  vm: any, // 实例
  info: string // 打印的信息
) {
  let res
  try {
    // 判断是否有为handler传入额外的参数args来使用不同的调用方式
    res = args ? handler.apply(context, args) : handler.call(context)
    
    // 若handler返回时promise，则增加catch处理
    if (res && !res._isVue && isPromise(res) && !res._handled) {
      res.catch(e => handleError(e, vm, info + ` (Promise/async)`))
      
      // 增加res.catch完成绑定识别，防止在多层引用时候，重复绑定多个catch而触发多次
      res._handled = true
    }
  } catch (e) {
    handleError(e, vm, info)
  }
  return res
}
```

最后的一个错误处理handleError：

```js
export function handleError (err: Error, vm: any, info: string) {
	// 推入一个空值，设置Dep.target = undefined
  // 防止无限渲染问题
  pushTarget()
  
  try {
    if (vm) {
      let cur = vm
      // 向上递归查询vm实例
      while ((cur = cur.$parent)) {
        // 获取当前错误处理钩子数组
        const hooks = cur.$options.errorCaptured
        if (hooks) {
          // 循环钩子数组
          for (let i = 0; i < hooks.length; i++) {
            try {
              // 调用钩子，若钩子捕获到错误会返回false
              // 否则抛异常
              const capture = hooks[i].call(cur, err, vm, info) === false
              if (capture) return
            } catch (e) {
              // 处理当前vm的错误
              globalHandleError(e, cur, 'errorCaptured hook')
            }
          }
        }
      }
    }
    // 最后调用最顶层vm的错误处理
    globalHandleError(err, vm, info)
  } finally {
    // 推出，Dep.target = 原先的
    popTarget()
  }
}
```

### 3.4 了解HTML标签名、属性名相关处理

```js
/**
 * unicode letters used for parsing html tags, component names and property paths.
 * using https://www.w3.org/TR/html53/semantics-scripting.html#potentialcustomelementname
 * skipping \u10000-\uEFFFF due to it freezing up PhantomJS
 */
// HTML标签Unicode匹配的正则，未处于此区间，则无效
export const unicodeRegExp = /a-zA-Z\u00B7\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u037D\u037F-\u1FFF\u200C-\u200D\u203F-\u2040\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD/

// 判断是否是保留的字符串，$ 和 _ 开头
export function isReserved (str: string): boolean {
  const c = (str + '').charCodeAt(0)
  return c === 0x24 || c === 0x5F
}

// Object.defineProperty 简单封装
export function def (obj: Object, key: string, val: any, enumerable?: boolean) {
  Object.defineProperty(obj, key, {
    value: val,
    enumerable: !!enumerable,
    writable: true,
    configurable: true
  })
}

// 这个方法主要用来解析例如：'data.obj.name', 'this.$emit()'等具有属性获取的字符串
// 并依次解析获取对应路径下的属性值
const bailRE = new RegExp(`[^${unicodeRegExp.source}.$_\\d]`)
export function parsePath (path: string): any {
  // 包含bailRE匹配的信息以外的字符，则返回
  if (bailRE.test(path)) {
    return
  }
  const segments = path.split('.') // 分片
  
  return function (obj) {
    for (let i = 0; i < segments.length; i++) {
      if (!obj) return // undefined则返回
      obj = obj[segments[i]] // 依次赋值给obj
    }
    return obj // 返回最终路径下的值
  }
}
```

### 3.5 了解new Vue(option)时的配置项处理

前文了解到，options.js文件存放着Vue默认的一些全局配置信息，而在组件实例化时候，往往需要传入自定义的配，这配置有别于全局配置，但优先级高于全局配置。

####3.5.1 前置依赖引入，配置项使用约束

```js
import config from '../config' // 默认全局配置项
import { warn } from './debug'
import { set } from '../observer/index' // 设置响应属性
import { unicodeRegExp } from './lang' // 有效unicode正则
import { nativeWatch, hasSymbol } from './env'

import {
  ASSET_TYPES,
  LIFECYCLE_HOOKS
} from 'shared/constants'

// 用到的通用函数
import {
  extend,
  hasOwn,
  camelize,
  toRawType,
  capitalize,
  isBuiltInTag,
  isPlainObject
} from 'shared/util'

/**
 * Option overwriting strategies are functions that handle
 * how to merge a parent option value and a child option
 * value into the final value.
 */
// 配置项合并规则，这边规则来自全局配置项
const strats = config.optionMergeStrategies

/**
 * Options with restrictions
 */
// 配置项的使用约束，非生产环境，必须在实例化中使用el，也就
if (process.env.NODE_ENV !== 'production') {
  strats.el = strats.propsData = function (parent, child, vm, key) {
    if (!vm) {
      warn(
        `option "${key}" can only be used during instance ` +
        'creation with the `new` keyword.'
      )
    }
    // 调用默认的规则处理
    return defaultStrat(parent, child)
  }
}
```

#### 3.5.2 数据的合并处理

```js
// data的合并规则，其实就是两个object的递归合并
function mergeData (to: Object, from: ?Object): Object {
  if (!from) return to // 未有合并来源，则直接返回当前的data对象
	
  // 三个关键变量
  let key, toVal, fromVal

  // 如果有Reflect则使用Reflect.ownKeys
  // 否则使用Object.keys来获取合并源对象的属性名
  // 使用Reflect来防止Object.keys的行为可能有偏差问题
  const keys = hasSymbol
    ? Reflect.ownKeys(from)
    : Object.keys(from)
	
  for (let i = 0; i < keys.length; i++) {
    key = keys[i]
    // in case the object is already observed...
		// 若是观察者相关属性，则跳过该属性的合并
    // 因为合并后的新的data会重新注入__ob__属性，来实现新data自有的响应规则
    if (key === '__ob__') continue
    
    toVal = to[key]
    fromVal = from[key]
    
    // 目标data该key不存在，则直接利用set方法注入，实现响应
    if (!hasOwn(to, key)) {
      set(to, key, fromVal)
    } else if (
      // 若两个值不想等且都是Object，则递归调用mergeData
      toVal !== fromVal &&
      isPlainObject(toVal) &&
      isPlainObject(fromVal)
    ) {
      mergeData(toVal, fromVal)
    }
  }
  
  // 返回合并后的新data
  return to
}
```

#### 3.5.6 数据或者函数合并处理

```js
export function mergeDataOrFn (
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  // vm不存在，则认定为extend等方式合并
  if (!vm) {
    // in a Vue.extend merge, both should be functions
    if (!childVal) {
      return parentVal
    }
    if (!parentVal) {
      return childVal
    }
    
    // 这边childVal、parentVal都可能是function
    // 因为在定义组件时候data、computed、watch等都可能是一个函数返回最终的data对象
    return function mergedDataFn () {
      
      // 两个若是函数则执行函数获得最终返回的对象值，然后调用mergeData返回合并后的结果
      return mergeData(
        typeof childVal === 'function' ? childVal.call(this, this) : childVal,
        typeof parentVal === 'function' ? parentVal.call(this, this) : parentVal
      )
    }
  } else {
    // 实例当中的合并，实例中对象
    return function mergedInstanceDataFn () {
      // instance merge
      const instanceData = typeof childVal === 'function'
        ? childVal.call(vm, vm)
        : childVal
      const defaultData = typeof parentVal === 'function'
        ? parentVal.call(vm, vm)
        : parentVal
      
     	// 子级合并数据为空，则直接使用父级数据，否则合并两个数据
      if (instanceData) {
        return mergeData(instanceData, defaultData)
      } else {
        return defaultData
      }
    }
  }
}
```

#### 3.5.7 默认data的合并规则

```js
// 这边直接定义了data的默认合并规则
// 在全局config中，重写Vue.config.optionMergeStrategies.data将覆盖当前默认规则
strats.data = function (
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  if (!vm) {
    if (childVal && typeof childVal !== 'function') {
      // 这个是不是很眼熟，就是在定义组件或者extend、mixin时候，
      // 若data直接定义称一个对象，则会有此提醒，因为这时候还未实例化
      process.env.NODE_ENV !== 'production' && warn(
        'The "data" option should be a function ' +
        'that returns a per-instance value in component ' +
        'definitions.',
        vm
      )
			// childVal非函数，则直接使用parentVal
      return parentVal
    }
    // 合并
    return mergeDataOrFn(parentVal, childVal)
  }

	// 实例中合并
  return mergeDataOrFn(parentVal, childVal, vm)
}
```

####3.5.8 生命周期钩子合并处理

官方文档中说明，生命周期合并将合并成一个数组，具体看一下源码实现：

```js
// 钩子函数数组去重，防止相同钩子执行多次
function dedupeHooks (hooks) {
  const res = []
  for (let i = 0; i < hooks.length; i++) {
    if (res.indexOf(hooks[i]) === -1) {
      res.push(hooks[i])
    }
  }
  return res
}

function mergeHook (
  parentVal: ?Array<Function>,
  childVal: ?Function | ?Array<Function>
): ?Array<Function> {
  const res = childVal
    ? parentVal
      ? parentVal.concat(childVal) // 父级与子级别直接合并
      : Array.isArray(childVal) // 父级钩子不存在，直接使用子级钩子

        ? childVal
        : [childVal] // childVal非数组，则构建成数组结构

    : parentVal	// 子级钩子未定义，则使用父级钩子
  return res
    ? dedupeHooks(res) // 钩子去重
    : res
}
```

#### 3.5.9 生命周期钩子合并规则

```js
// 直接遍历`LIFECYCLE_HOOKS`的钩子名称赋值到合并规则里
LIFECYCLE_HOOKS.forEach(hook => {
  strats[hook] = mergeHook
})
```

####3.5.10 静态属性合并处理

从constants.js中得知ASSET_TYPES中定义的类型有：component、directive、filter，这些不会发生改变的静态属性，我们来看下具体的处理逻辑：

```js
// 判断是否是Object并提示
function assertObjectType (name: string, value: any, vm: ?Component) {
  if (!isPlainObject(value)) {
    warn(
      `Invalid value for option "${name}": expected an Object, ` +
      `but got ${toRawType(value)}.`,
      vm
    )
  }
}

// 合并静态资源
function mergeAssets (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): Object {
  const res = Object.create(parentVal || null)
  if (childVal) {
    process.env.NODE_ENV !== 'production' && assertObjectType(key, childVal, vm)
    // 直接实现extend，继承父级静态属性，并且同名覆盖
    return extend(res, childVal)
  } else {
    return res // childVal不存在，则直接使用父级静态属性
  }
}
```

#### 3.5.11 静态属性合并规则

```js
ASSET_TYPES.forEach(function (type) {
  strats[type + 's'] = mergeAssets
})
```

####3.5.12 观察器watch的合并规则

```js
strats.watch = function (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
  // work around Firefox's Object.prototype.watch...
	// 检查是否使用Object.prototype.watch
  if (parentVal === nativeWatch) parentVal = undefined
  if (childVal === nativeWatch) childVal = undefined

	// childVal空则返回父级watch属性，或者{}
  if (!childVal) return Object.create(parentVal || null)

  if (process.env.NODE_ENV !== 'production') {
    // 检查Object类型
    assertObjectType(key, childVal, vm)
  }
	
	// childVal存在、parentVal不存在，直接使用childVal
  if (!parentVal) return childVal
	
	// childVal，parentVal都存在
  const ret = {}
  extend(ret, parentVal) // 拷贝parentVal到ret
	
  for (const key in childVal) {
    let parent = ret[key]
    const child = childVal[key]
    
    // 若父级也存在当前key都属性
    // 判断父级属性值是否为数组并构造成数组结构
    if (parent && !Array.isArray(parent)) {
      parent = [parent]
    }
    
    ret[key] = parent
      ? parent.concat(child)
      : Array.isArray(child) ? child : [child] // 构造成数组结构
  }
  return ret // 合并后的watch属性
}
```

#### 3.5.13 其他属性合并规则（props、methods、inject、computed、provide）

```js
// 这几个属性直接使用了extend方式进行合并
strats.props =
strats.methods =
strats.inject =
strats.computed = function (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
	// 还是检查Object类型
  if (childVal && process.env.NODE_ENV !== 'production') {
    assertObjectType(key, childVal, vm)
  }
  if (!parentVal) return childVal
  const ret = Object.create(null)
  extend(ret, parentVal)
  if (childVal) extend(ret, childVal)
  return ret
}

strats.provide = mergeDataOrFn // 这边使用了mergeDataOrFn函数
```

#### 3.5.14 相关函数

```js
// 默认规则获取，优先级childVal > parentVal
const defaultStrat = function (parentVal: any, childVal: any): any {
  return childVal === undefined
    ? parentVal
    : childVal
}

// 检查components字段下的组件名称合法性，validateComponentName的批处理
function checkComponents (options: Object) {
  for (const key in options.components) {
    validateComponentName(key)
  }
}

// 检查组件名称合法性
export function validateComponentName (name: string) {
  // 按一下规则检测
  if (!new RegExp(`^[a-zA-Z][\\-\\.0-9_${unicodeRegExp.source}]*$`).test(name)) {
    warn(
      'Invalid component name: "' + name + '". Component names ' +
      'should conform to valid custom element name in html5 specification.'
    )
  }
  // 检查是否非内建标签名、预留标签名
  if (isBuiltInTag(name) || config.isReservedTag(name)) {
    warn(
      'Do not use built-in or reserved HTML elements as component ' +
      'id: ' + name
    )
  }
}

// 将props属性名定义中的option字段统一转成Object格式
function normalizeProps (options: Object, vm: ?Component) {
  const props = options.props
  if (!props) return
  
  const res = {}
  let i, val, name
  
  // porps以数组形式定义
  // 如: props: ['p1', 'p2'] 
  if (Array.isArray(props)) {
    i = props.length
    while (i--) {
      val = props[i]
      if (typeof val === 'string') {
        name = camelize(val) // 转成驼峰形式 'is-ok' -> 'isOk'
        
        // 由于只定义了字段名，类型未知，所以设置成null，其实就是任意类型
        res[name] = { type: null } 
        
      } else if (process.env.NODE_ENV !== 'production') {
        // 以数组形式，内容必须是字符串
        warn('props must be strings when using array syntax.')
      }
    }
    
  // props以对象形式定义
  // 如：props: { total: { type: Number, default: 0 } }
  // props: { msg: String }
  } else if (isPlainObject(props)) {
    for (const key in props) {
      val = props[key]
      name = camelize(key) // 转成驼峰
      res[name] = isPlainObject(val)
        ? val
        : { type: val } // 非对象形式，直接将val（规定是类型的构造函数）构造成Object形式
    }
  
  // 非生产环境，则会报错提示
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `Invalid value for option "props": expected an Array or an Object, ` +
      `but got ${toRawType(props)}.`,
      vm
    )
  }
  
  // 重新更新赋值
  options.props = res
}

// 将inject的参数转成统一的Object结构，道理与上文props处理类似
function normalizeInject (options: Object, vm: ?Component) {
  const inject = options.inject
  if (!inject) return
  const normalized = options.inject = {}
  
  // 数组形式定义
  if (Array.isArray(inject)) {
    for (let i = 0; i < inject.length; i++) {
      // from为原始值，因为inject以数组方式定义，所以没有default字段
      normalized[inject[i]] = { from: inject[i] }
    }
  
  // 对象形式定义
  } else if (isPlainObject(inject)) {
    for (const key in inject) {
      const val = inject[key]
      normalized[key] = isPlainObject(val)
        // 若对应值是Object，则注入from字段
        ? extend({ from: key }, val) 
      	// 非Object类型，则构造成Object
        : { from: val }
    }
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `Invalid value for option "inject": expected an Array or an Object, ` +
      `but got ${toRawType(inject)}.`,
      vm
    )
  }
}

// 将directives指令转换成Object结构
function normalizeDirectives (options: Object) {
  const dirs = options.directives
  if (dirs) {
    for (const key in dirs) {
      const def = dirs[key]
      // 对应值类型为function才判定为有效指令
      if (typeof def === 'function') {
        // 默认两种触发方式，bind，update
        dirs[key] = { bind: def, update: def }
      }
    }
  }
}

// 判断Object类型
function assertObjectType (name: string, value: any, vm: ?Component) {
  if (!isPlainObject(value)) {
    warn(
      `Invalid value for option "${name}": expected an Object, ` +
      `but got ${toRawType(value)}.`,
      vm
    )
  }
}

// 子实例需要获取到祖先静态资源链
// 例如在自组件a中使用了过滤器 {{ msg | 'fit' }}
// 先从当前a组件的option去查询是否是自有属性
// 不是自有属性，则去原型链查询
export function resolveAsset (
  options: Object,
  type: string,
  id: string,
  warnMissing?: boolean
): any {
  
  if (typeof id !== 'string') {
    return
  }
  const assets = options[type] // 获取options中的静态属性，例如：filter
  
  // 检查组件是否定义
  if (hasOwn(assets, id)) return assets[id]

  const camelizedId = camelize(id) // id转成驼峰形式查询
  if (hasOwn(assets, camelizedId)) return assets[camelizedId]

  const PascalCaseId = capitalize(camelizedId) // id转成双驼峰形式查询
  if (hasOwn(assets, PascalCaseId)) return assets[PascalCaseId]

  // 从原型assets的原型链查询
  const res = assets[id] || assets[camelizedId] || assets[PascalCaseId]

  // 未找到则报错
  if (process.env.NODE_ENV !== 'production' && warnMissing && !res) {
    warn(
      'Failed to resolve ' + type.slice(0, -1) + ': ' + id,
      options
    )
  }
  return res
}
```

#### 3.5.15 最终option的合并处理

option配置对象有不同的类别：

1.默认option

2.父级option

3.当前opiton

利用上面的三个option，最终生成是实例化后的option，挂在到当前实例中，合并到优先级也是从当前option高于上层option，具体的合并逻辑看一下源码：

```js
// 合并两个option生产一个新option，实现组合与继承
export function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  if (process.env.NODE_ENV !== 'production') {
    checkComponents(child) // 检查组件标签名合法性
  }
	
  // 如果是构造函数，非option对象，获取构造函数的option属性
  // 这边的场景是extend、mixin操作
  if (typeof child === 'function') {
    child = child.options
  }
	
  // 统一格式化child的props，inject，directives定义格式为Object
  normalizeProps(child, vm)
  normalizeInject(child, vm)
  normalizeDirectives(child)

  // Apply extends and mixins on the child options,
  // but only if it is a raw options object that isn't
  // the result of another mergeOptions call.
  // Only merged options has the _base property.
	// 进行extens、mixins的原始option合并
  // 原始option指的是传入传入构造函数Vue、extend、mixin的配置对象
  // 且配置对象不包含_base属性
  // 注意：Vue.opiton._base表示着根构造函数的配置项，用这个配置项来扩展所有配置项
  if (!child._base) {
    if (child.extends) {
      // 递归合并extends
      parent = mergeOptions(parent, child.extends, vm)
    }
    if (child.mixins) {
      for (let i = 0, l = child.mixins.length; i < l; i++) {
        // 递归合并mixins
        parent = mergeOptions(parent, child.mixins[i], vm)
      }
    }
  }
  
  const options = {}
  let key
  
  // 按照规则来进行字段值的合并
  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  
  // 合并parent所有字段属性用child做参考，合并到新的option
  for (key in parent) {
    mergeField(key)
  }
  
  // 合并child不存在与parent的属性，合并到新的option
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  
  // 分会最终合并生产的新配置
  return options
}
```

### 3.6. 了解Props的定义、解析、初始化

props属性定义方式有两种: 1.数组，2.对象。数组方式无法获知props的具体属性，对象形式可以指定props属性的类型，默认值，和有效性检测。我们先来看下一些处理Props中要用到的函数方法：

#### 3.6.1 prop处理用到的帮助方法

```js
// 获取内建函数名
// 在以对象方式定义props时候，会有定义type字段，形如：{ type: String }
// String为内建的字符串构造方法
// 运行getType(String) -> 'String'
function getType (fn) {
  const match = fn && fn.toString().match(/^\s*function (\w+)/)
  return match ? match[1] : ''
}

// 相同类型判断
function isSameType (a, b) {
  return getType(a) === getType(b)
}

// 定义在传入的类型预设列表中，查询传入类型的index
function getTypeIndex (type, expectedTypes): number {
  if (!Array.isArray(expectedTypes)) {
    return isSameType(expectedTypes, type) ? 0 : -1
  }
  for (let i = 0, len = expectedTypes.length; i < len; i++) {
    if (isSameType(expectedTypes[i], type)) {
      return i
    }
  }
  return -1
}

// 数值风格格式化
// 方便与其他字符串结合时候，直观地突出类型
// 'abc' -> '"abc"'
// 123 -> '123'
// true -> 'true'
function styleValue (value, type) {
  if (type === 'String') {
    return `"${value}"`
  } else if (type === 'Number') {
    return `${Number(value)}`
  } else {
    return `${value}`
  }
}

// 判断value的类型是否在在可理解范围内，也就是原始值
// 可理解的三个类型['string', 'number', 'boolean']
function isExplicable (value) {
  const explicitTypes = ['string', 'number', 'boolean']
  return explicitTypes.some(elem => value.toLowerCase() === elem)
}

// 判断是否包含布尔类型'boolean'
function isBoolean (...args) {
  return args.some(elem => elem.toLowerCase() === 'boolean')
}

// 获取具体无效类型的信息
function getInvalidTypeMessage (name, value, expectedTypes) {
  let message = `Invalid prop: type check failed for prop "${name}".` +
    ` Expected ${expectedTypes.map(capitalize).join(', ')}`
  
  const expectedType = expectedTypes[0]
  const receivedType = toRawType(value) // 获取原始类型信息 toRawType(1) -> 'Number'
  const expectedValue = styleValue(value, expectedType) // 类型风格化
  const receivedValue = styleValue(value, receivedType) // 数值风格化
  
  // check if we need to specify expected value
  // 是否提醒需要传入的参数类型
  // 这边需要满足三个条件：
  if (expectedTypes.length === 1 && // 条件1：期待的类型就一个
      isExplicable(expectedType) && // 条件2：期待类型为string, number, boolean三个之一
      !isBoolean(expectedType, receivedType)) { // 条件3：并且类型不为boolan
    message += ` with value ${expectedValue}` // 满足三个条件则主动提示期待的具体数值
  }
  message += `, got ${receivedType} ` // 提示实际传入的类型
  
	// 传入的类型为stirng，number，boolean三个之一，则添加提示
  if (isExplicable(receivedType)) {
    message += `with value ${receivedValue}.` // 提示实际传入的值
  }
  
  return message
}
```

#### 3.6.2 判断prop类型有效性

```js
const simpleCheckRE = /^(String|Number|Boolean|Function|Symbol)$/
function assertType (value: any, type: Function): {
  valid: boolean;
  expectedType: string;
} {
  let valid
  const expectedType = getType(type)
  // 检查传入的type构造函数是否为指定的类型
  if (simpleCheckRE.test(expectedType)) {
    const t = typeof value
    valid = t === expectedType.toLowerCase()
		// 检查非原始值，检查数值是否为该类型的实例
    // 如: n = new Number(1) -> typeof n 为 'object'
    if (!valid && t === 'object') {
      valid = value instanceof type
    }
  // 对应的Object、Array类型判断
  } else if (expectedType === 'Object') {
    valid = isPlainObject(value)
  } else if (expectedType === 'Array') {
    valid = Array.isArray(value)
  } else {
    // 其他自定义类型判断，直接判断value是否为该类实例
    valid = value instanceof type
  }
  return {
    valid,
    expectedType
  }
}
```

#### 3.6.3 判断prop的有效性

```js
function assertProp (
  prop: PropOptions,
  name: string,
  value: any,
  vm: ?Component,
  absent: boolean
) {
    
  // 检查prop是否必须且有传
  if (prop.required && absent) {
    warn(
      'Missing required prop: "' + name + '"',
      vm
    )
    return
  }
  
  // 检查vlaue有效
  // 即，在prop不做必传前提下，值不为undefined或者null
  if (value == null && !prop.required) {
    return
  }
    
  let type = prop.type // 可能未定义，则为undefined
  let valid = !type || type === true
  // !undefined -> true
  // 由于没定义type则不做类型限制，valid为true
	// !String -> false -> String === true -> false
  // 做了类型定义，valid设为false，需要进一步判断包括类型在内的其他条件有效性
  // 如：default数值验证，validate验证
  
  const expectedTypes = []
  if (type) {
    if (!Array.isArray(type)) {
      type = [type] // type有定义，非数组则构造成数组
    }
    for (let i = 0; i < type.length && !valid; i++) { // valid为false才进行类型遍历判断
      const assertedType = assertType(value, type[i]) // 验证value对应的type[i]是否有效
      expectedTypes.push(assertedType.expectedType || '') // 获取真正要传的value类型
      valid = assertedType.valid // 每次遍历都获取当前类型是否有效，若存在有效类型，则结束循环
    }
  }
	
  // 传入的value类型无效
  if (!valid) {
    warn(
      getInvalidTypeMessage(name, value, expectedTypes), // 打印无效信息
      vm
    )
    return
  }

  // 传入vlaue类型有效，进一步做value的验证
  // 如：type: Number, value: 10, 但prop实际期待的值为[0, 5], 则需要进一步验证
  const validator = prop.validator
  if (validator) {
    if (!validator(value)) {
      warn(
        'Invalid prop: custom validator check failed for prop "' + name + '".',
        vm
      )
    }
  }

  // 这边没有返回true，只做验证与提示，prop在实际使用中，就算无效也可获得
}
```

#### 3.6.4 获取prop默认值

```js
function getPropDefaultValue (vm: ?Component, prop: PropOptions, key: string): any {
  // no default, return undefined
  if (!hasOwn(prop, 'default')) {
    return undefined
  }
  
  const def = prop.default
  // warn against non-factory defaults for Object & Array
  // Object与Array的默认值需要是工厂函数，返回具体的值
  if (process.env.NODE_ENV !== 'production' && isObject(def)) {
    warn(
      'Invalid default value for prop "' + key + '": ' +
      'Props with type Object/Array must use a factory function ' +
      'to return the default value.',
      vm
    )
  }
  // the raw prop value was also undefined from previous render,
  // return previous default value to avoid unnecessary watcher trigger
  // 若props传入的值仍然为undefined，则返回改组件上一次渲染的default值，防止多触发一次无用的更新
  if (vm && vm.$options.propsData &&
    vm.$options.propsData[key] === undefined &&
    vm._props[key] !== undefined
  ) {
    return vm._props[key]
  }
	
  // 最终根据default的类型来判断最终返回的值
  return typeof def === 'function' && getType(prop.type) !== 'Function'
    ? def.call(vm) // 执行工厂函数，并返回结果
    : def // 非Object，Array，直接返回default
}
```


#### 3.6.5 对于配置项中props的解析过程

```js
export function validateProp (
  key: string,
  propOptions: Object,
  propsData: Object,
  vm?: Component
): any {
  const prop = propOptions[key]
  const absent = !hasOwn(propsData, key)
  let value = propsData[key]
  
  // boolean casting
  const booleanIndex = getTypeIndex(Boolean, prop.type)
  if (booleanIndex > -1) {
    if (absent && !hasOwn(prop, 'default')) {
      value = false
    } else if (value === '' || value === hyphenate(key)) {
      // only cast empty string / same name to boolean if
      // boolean has higher priority
      const stringIndex = getTypeIndex(String, prop.type)
      if (stringIndex < 0 || booleanIndex < stringIndex) {
        value = true
      }
    }
  }
  
  // check default value
  if (value === undefined) {
    value = getPropDefaultValue(vm, prop, key)
    // since the default value is a fresh copy,
    // make sure to observe it.
    const prevShouldObserve = shouldObserve
    toggleObserving(true)
    observe(value)
    toggleObserving(prevShouldObserve)
  }
  
  if (
    process.env.NODE_ENV !== 'production' &&
    // skip validation for weex recycle-list child component props
    !(__WEEX__ && isObject(value) && ('@binding' in value))
  ) {
    assertProp(prop, key, value, vm, absent)
  }
  return value
}
```

### 3.7 事件轮询nextTick的封装

```js
import { noop } from 'shared/util'
import { handleError } from './error'
import { isIE, isIOS, isNative } from './env'

export let isUsingMicroTask = false // 是否使用微任务，默认设为false，使用宏任务

const callbacks = [] // 回调队列
let pending = false // 阻塞标识

// 执行回调队列
function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0 // 清空
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}

// Here we have async deferring wrappers using microtasks.
// In 2.5 we used (macro) tasks (in combination with microtasks).
// However, it has subtle problems when state is changed right before repaint
// (e.g. #6813, out-in transitions).
// Also, using (macro) tasks in event handler would cause some weird behaviors
// that cannot be circumvented (e.g. #7109, #7153, #7546, #7834, #8109).
// So we now use microtasks everywhere, again.
// A major drawback of this tradeoff is that there are some scenarios
// where microtasks have too high a priority and fire in between supposedly
// sequential events (e.g. #4521, #6690, which have workarounds)
// or even between bubbling of the same event (#6566).

// 这段注释大概说了microtasks和macrotasks使用方案的折衷选择，最终优先选择微任务
// 由于微任务级别更高，会打断悬挂的事件和原生的冒泡事件队列，所以先往下看，具体是怎么样的一个实现方案。

let timerFunc // 定义timer函数

// The nextTick behavior leverages the microtask queue, which can be accessed
// via either native Promise.then or MutationObserver.
// MutationObserver has wider support, however it is seriously bugged in
// UIWebView in iOS >= 9.3.3 when triggered in touch event handlers. It
// completely stops working after triggering a few times... so, if native
// Promise is available, we will use it:

// 这段注释说了nextTick用何种API来实现，有Promise.then和MutationObserver，由于兼容性问题，
// 就采用了Promise.then如果环境支持的话

// 检测Promise能力
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve() // 生成一个Promise的resoved实例
  timerFunc = () => {
    p.then(flushCallbacks)
    // In problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    // 这种牛逼的黑科技～～
    // 在ios下推入一个空的回调到宏任务队列，微任务队列才会开始轮询
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true // 微任务能力可用，则修改标识
  
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  // 在Promise不可用时候，使用MutationObserver
  let counter = 1
  const observer = new MutationObserver(flushCallbacks) // 创建观察者，绑定回调
  const textNode = document.createTextNode(String(counter)) // 创建DOM
  // 实现DOM的观察者绑定
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2 // 交替赋值1， 0变更状态
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // setImmediate没有timeout的时间间隔、响应优先级高于setTimeout
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0) // 是在不行，就用setTimeout
  }
}

export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  // 推入函数到callbacks
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx) // 调用cb
      } catch (e) {
        handleError(e, ctx, 'nextTick') // 错误捕捉
      }
    // 若支持Promise，则调用Promise的resove
    } else if (_resolve) {
      _resolve(ctx)
    }
  })

  if (!pending) {
    pending = true // 改变pendding
    timerFunc() // 直接执行队列，this.$nextTick调用后将会实现一次
    // 通过清空当前列队，由于最后一个是当前的回调，所以TaskStack最后一个函数
    // 就是下次事件轮询的开始
    // 以此实现nextTick的效果
  }
	
  // cb未传，若支持Promise，则返回promise实例，且改变_resolve的引用
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}

```

###3.7 开发模式下的性能检测

```js
// 参考文档：https://developer.mozilla.org/zh-CN/docs/Web/API/Performance
import { inBrowser } from './env'

export let mark // timestamp标记
export let measure // 测量结果数组

if (process.env.NODE_ENV !== 'production') {
  const perf = inBrowser && window.performance // 利用window的performance API
  if (
    perf &&
    perf.mark &&
    perf.measure &&
    perf.clearMarks &&
    perf.clearMeasures
  ) {
    mark = tag => perf.mark(tag) // 给定当前位置的时间戳
    measure = (name, startTag, endTag) => {
      perf.measure(name, startTag, endTag) // 测量对应命名的
      perf.clearMarks(startTag) // 清除开始标记
      perf.clearMarks(endTag) // 清除结束标记
      // perf.clearMeasures(name)
    }
  }
}

```




