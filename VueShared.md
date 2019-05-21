

# Vue Shared目录源码阅读

[TOC]

# 1. shared 目录

工与善其事、必先利其器，先看下shared目录下的文件：

1. constants.js
2. utils.js

## 1.1 constants目录

一些需要全局用到的常量、如生命周期名称等，这边没什么好详细讲解的，文件内容如下：

```js
export const SSR_ATTR = 'data-server-rendered'

export const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]

export const LIFECYCLE_HOOKS = [
  'beforeCreate',
  'created',
  'beforeMount',
  'mounted',
  'beforeUpdate',
  'updated',
  'beforeDestroy',
  'destroyed',
  'activated',
  'deactivated',
  'errorCaptured',
  'serverPrefetch'
]

```

## 1.2 utils目录

utils文件里面存放着一些通用的函数方法、比如类型的判断、类型转换、等值比较等等。选择从utils文件开始解读，第一可以学习这些方法的实现方式，第二了解这些方法的功能，为接下来的其他部分源码阅读做准备。

### 1.2.1 emptyObject

```js
export const emptyObject = Object.freeze({})
```

这边利用了`Object`对象中的一个方法`freeze`，该方法通过修改对象的属性描述符，可以实现冻结一个对象，使对象不可修改、不可删除、并且该对象不可扩展其他属性。于此相关的API还有`Object.seal()`、`Object.preventExtensions()`可以前去查阅文档了解详细用法。

这边定义了一个冻结的空对象：TODO

### 1.2.2 isUndef()

```js
export function isUndef (v: any): boolean %checks {
  return v === undefined || v === null
}
```

判断一个变量是不是undefine或者null，用来判断空值。

#### 扩展

变量未定义，JS中的原始的未定义为未通过`var`、`let`、`const` 这三个关键字定义的变量则称为未定义，然而定义未初始化则默认初始化为undefined，例如：

```js
console.log(a) // -> ReferenceError: a is not defined

var b
console.log(b) // -> undefined

var c = 1
console.log(c) // -> 1

d // 等同 var d
console.log(d) // -> undefined

c = 'c' // 等同 var c = 'c'
console.log(c) // -> 'c'
```

由于js的特性，可以忽略关键字来实现定义一个变量、若定义在非全局定义域，则在非严格模式下变量挂在window下：

```js
function test() {
  a = 1 // 等效于 window.a = 1
}
test()
console.log(window.a) // -> 1
```

### 1.2.3 isDef()

```js
export function isDef (v: any): boolean %checks {
  return v !== undefined && v !== null
}
```

与`isUndef`相反用途，判断非undefined和null。

###1.2.4 isTrue(), isFalse()

```js
export function isTrue (v: any): boolean %checks {
  return v === true
}

export function isFalse (v: any): boolean %checks {
  return v === false
}
```

判断真假，前面这四个方法，封装起来，主要为了方便语句编写。

### 1.2.5 isPrimitive()

```js
export function isPrimitive (value: any): boolean %checks {
  return (
    typeof value === 'string' ||
    typeof value === 'number' ||
    // $flow-disable-line
    typeof value === 'symbol' ||
    typeof value === 'boolean'
  )
}
```

基本原始值判断、ES5最主要的三个常用原始值：string，number，boolean，ES6则增加了symbol，这边需要区分原始值和基本类型。

#### 扩展

基本数据类型（ES6）：

1. string
2. number
3. boolean
4. symbol
5. undefined
6. null

使用typeof运算符可用来判断数据类型，其中null的数据类型为'object'。

```js
typeof 'hello' // 'string'
typeof 1 // 'number'
typeof true // 'boolean'
typeof Symbol('s1') // 'symbol'
typeof undefined // 'undefined'
typeof null // 'object'
```

### 1.2.6 isObject()

```js
export function isObject (obj: mixed): boolean %checks {
  return obj !== null && typeof obj === 'object'
}
```

快速判断一个变量是否不为null的对象类型。

###1.2.7 toRawType()

```js
const _toString = Object.prototype.toString
```

定义_toString，减少代码量，方便引用。Object.prototype.toString方法，可以获得任何变量的内建类型信息，用该方式能够准确获得变量类型，例如：

```js
const toStr = Object.prototype.toString
toStr.call(new Date()) // '[object Date]'
toStr.call(1) // '[object Number]'
toStr.call(null) // '[object Null]'

// 面对复合类型，typeof 则显得有些乏力：
typeof new Date() // 'object'
typeof null // 'object'
```



```js
export function toRawType (value: any): string {
  return _toString.call(value).slice(8, -1)
}
```

截取Object.prototype.toString结果，如：'[object Object]' -> 'Object'。



###1.2.8 isPlainObject()

```js
export function isPlainObject (obj: any): boolean {
  return _toString.call(obj) === '[object Object]'
}
```

判断最基础的'Object'类型，如:

```js
const toStr = Object.prototype.toString
const obj1 = new Object()
const obj2 = new Object('abc')
toStr.call(obj1) // '[object Object]'
toStr.call(obj2) // '[object String]'
```

### 1.2.9 isRegExp()

```js
export function isRegExp (v: any): boolean {
  return _toString.call(v) === '[object RegExp]'
}
```

判断正则RegExp类型。

###1.2.10 isValidArrayIndex()

```js
export function isValidArrayIndex (val: any): boolean {
  const n = parseFloat(String(val))
  return n >= 0 && Math.floor(n) === n && isFinite(val)
}
```

判断一个值是不是常规有效的数组下标序列，不常规的序列如：-1，'abc' 等。

判断步骤：
1.将值转成number类型，利用parseFloat()和String()
2.判断序列大于等于0 且 为整数 且 有限

```js
const n = parseFloat(String(val))
// 由于parseInt，parseFloat传入的参数必须是string类型，所以先用String()方法将val转成字符串

n >= 0 // 下标必须从大于等于0

Math.floor(n) === n // 则可以判断n是否为整数

isFinite(val) 
// 值为有限值，这边加这句判断是因为 Infinity是number类型，并且>=0，然而这个值是不常规的无效下标
```

#### 扩展

由于JS中的类型是动态的，虽然复合类型有确切的值类型，但是都基于Object类，则都可以为其增加[key, value],就刚刚的数组类型Array举个例子

```js
const arr = [1, 2]
arr instanceof Array // true
arr instanceof Object // true

arr['name'] = 'My List'
console.log(arr) // -> [0: 1, 1: 2, name: 'My List']

// 是不是很眼熟，没错，Array只是特殊的Object，因为key为自然数，然而：
console.log(arr.length) // -> 2
arr.push(3)
console.log(arr) // -> 3

for (var i = 0; i < arr.length; i++) {
  console.log(i, arr[i])
}
// -> 0 1, 1 2, 2 3

arr.forEach(v => console.log(v)) // 1, 2, 3

// 当我们的传入的下标为无效时，这个值永远都不会被遍历到，除非使用Object的便利方式，如：
for (var key in arr) {
	console.log(key, arr[key])
}
// -> '0' 1, '1' 2, '2' 3, 'name' 'My List'
```

### 1.2.11 isPromise()

```js
export function isPromise (val: any): boolean {
  return (
    isDef(val) &&
    typeof val.then === 'function' &&
    typeof val.catch === 'function'
  )
}
```

判断是否是Promise实例

判断步骤：

1.已定义

2.拥有then方法

3.拥有catch方法

这边的判断步骤只是简单的判断，因为所在环境并非都原生支持Promise，大部分使用了polyfill，这边只初步判断拥有then和catch方法，其实若是钻牛角尖一点，这个方法在任何对象只要拥有then和catch方法都返回true。

### 1.2.12 toString()

```js
export function toString (val: any): string {
  return val == null
    ? ''
    : Array.isArray(val) || (isPlainObject(val) && val.toString === _toString)
      ? JSON.stringify(val, null, 2)
      : String(val)
}
```

这边定义了一个自用的toString()方法，按照自己的一套规则实现toString, 这边主要三大条输出逻辑：

1. null | undefined -> ''
2. […] | {...}  -> JSON.stringify(val)
3. others -> String(val)

### 1.2.13 toNumber()

```js
export function toNumber (val: string): number | string {
  const n = parseFloat(val)
  return isNaN(n) ? val : n
}
```

按照规则实现toNumber方法，主要逻辑：

1. 使用parseFloat解析，可解析则返回解析后的值，不能解析则返回原值



### 1.2.14 makeMap()

```js
export function makeMap (
  str: string,
  expectsLowerCase?: boolean
): (key: string) => true | void {
  const map = Object.create(null)
  const list: Array<string> = str.split(',')
  for (let i = 0; i < list.length; i++) {
    map[list[i]] = true
  }
  return expectsLowerCase
    ? val => map[val.toLowerCase()]
    : val => map[val]
}
```

创建一个创建目标关键字序列map，用来记录目标关键字是否存在，这边利用的闭包的特性：

```js
const map = Object.create(null)
// 利用Object.create()方法创建一个纯粹元对象map，该对象没有任何的属性

const list = str.split(',')
// 参数str按照`key,key,key,...`格式传入，并且利用split(',')分割成数组

// 遍历list并赋值map[key]为true

// 最后利用闭包的形式返回查询方法：
return val => map[val]

// 对于expectsLowerCase参数则设置是否只查询小写的key

/* 使用例子 */
const checkInMap = makeMap('a,b,A')
checkInMap('a') // true
checkInMap('b') // true
checkInMap('c') // undefined
```

### 1.2.15 isBuiltInTag()

```js
export const isBuiltInTag = makeMap('slot,component', true)
```

判断是不是内建标签，也就是关键字检查`slot,component`

### 1.2.16 isReservedAttribute()

```js
export const isReservedAttribute = makeMap('key,ref,slot,slot-scope,is')
```

判断是不是保留字属性，也就是关键字检查`key,ref,slot,slot-scope,is`

### 1.2.17 remove()

```js
export function remove (arr: Array<any>, item: any): Array<any> | void {
  if (arr.length) {
    const index = arr.indexOf(item)
    if (index > -1) {
      return arr.splice(index, 1)
    }
  }
}
```

用来删除数组中的一个值，这个方法只需要传入值来删除，而原生的splice则是传入确切的裁剪范。

### 1.2.18 hasOwn()

```js
const hasOwnProperty = Object.prototype.hasOwnProperty
export function hasOwn (obj: Object | Array<*>, key: string): boolean {
  return hasOwnProperty.call(obj, key)
}
```

Object.prototype.hasOwnProperty的精简化封装，减少代码量，用来判断对象是否拥有实例属性。

### 1.2.19 cached()

```js
export function cached<F: Function> (fn: F): F {
  const cache = Object.create(null)
  return (function cachedFn (str: string) {
    const hit = cache[str]
    return hit || (cache[str] = fn(str))
  }: any)
}
```

为纯函数做缓存，纯函数意味着没有副作用，同样的输入获得同样的结果。

对于map的创建方法可参考上文makeMap部分，这边提一下最后的返回值处理，关于短路求值：

对于逻辑运算会从左到右依次求值：

1. 多个&&运算，只要出现一个判定为false，则结果为判定为false时的计算结果，不继续下一个条件的计算。
2. 多个||运算，只要出现一个判定为true，则结果为判定为true时的计算结果，不继续下一个条件的计算。

```js
var a = 1
const b1 = 1 && false && a++
console.log(b1, a) // false, 1

const b2 = 1 && true && a++
console.log(b2, a) // 1, 2

const r1 = 1 || 'last'
const r2 = 0 || 'last'
console.log(r1) // 1
console.log(r2) // 'last'
```

所以在源码中：

```js
const hit = cache[str] // 查询是否对应的输入str已经存在cache中，也就是说对应的str这个值已被运算过
return hit || (cache[str] = fn(str)) // 若hit存在返回hit，不存在，将运算结果存到cache并返回
```

### 1.2.20 camelize()

```js
const camelizeRE = /-(\w)/g
export const camelize = cached((str: string): string => {
  return str.replace(camelizeRE, (_, c) => c ? c.toUpperCase() : '')
})
```

横杠连接字符串转驼峰，如：hello-world  -> helloWorld, 通常用来转换将组件属性名、类名转成HTML标准的属性名、类名。这边利用cache进行缓存。

```js
const camelizeRE = /-(\w)/g // 全局匹配一个横杠紧接着一个单词的
```

匹配替换replace方法中的第二个替换参数为函数，其中函数的使用方法和参数可以参考一下列表

| 变量名       | 代表的值                                                     |
| ------------ | ------------------------------------------------------------ |
| `match`      | 匹配的子串。（对应于上述的$&。）                             |
| `p1,p2, ...` | 假如replace()方法的第一个参数是一个[`RegExp`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/RegExp) 对象，则代表第n个括号匹配的字符串。（对应于上述的$1，$2等。）例如，如果是用 `/(\a+)(\b+)/` 这个来匹配，`p1` 就是匹配的 `\a+`，`p2` 就是匹配的 `\b+`。 |
| `offset`     | 匹配到的子字符串在原字符串中的偏移量。（比如，如果原字符串是 `'abcd'`，匹配到的子字符串是 `'bc'`，那么这个参数将会是 1） |
| `string`     | 被匹配的原字符串。                                           |

```js
// replacement
function (_, c) {
  // 若原字符串中-(\w)匹配不到，此时c为undefined，此处才需要三元判断
	return c ? c.toUpperCase() : '' 
}
```

### 1.2.21 capitalize()

```js
export const capitalize = cached((str: string): string => {
  return str.charAt(0).toUpperCase() + str.slice(1)
})
```

实现字符串首字母大写转换，通用实现了cache，这边利用了charAt(0)方法定位首字母，并且利用slice(1)切割从第二位到结束的字符串，最后完成拼接。

#### 扩展

1.charCodeAt()   // 返回字符串对应位置字符的UTF-16码值
2.codePointAt() // 返回一个Unicode 编码点值的非负整数
3.fromCharCode()  // 方法返回由指定的UTF-16代码单元序列创建的字符串
4.fromCodePoint() // 方法返回使用指定的Unicode 编码点值序列创建的字符串

### 1.2.22 hyphenate()

```js
const hyphenateRE = /\B([A-Z])/g
export const hyphenate = cached((str: string): string => {
  return str.replace(hyphenateRE, '-$1').toLowerCase()
})
```

将驼峰字符串转成横杠连接字符串，这边正则中利用了`\B`则表示`[A-Z]`前面必须有内容才能匹配到，然后替换字符串中利用`-$1`完成横杠添加并统一转成小写。

### 1.2.23 polyfillBind(), nativeBind(), bind()

```js
function polyfillBind (fn: Function, ctx: Object): Function {
  function boundFn (a) {
    const l = arguments.length
    return l
      ? l > 1
        ? fn.apply(ctx, arguments)
        : fn.call(ctx, a)
      : fn.call(ctx)
  }

  boundFn._length = fn.length
  return boundFn
}

function nativeBind (fn: Function, ctx: Object): Function {
  return fn.bind(ctx)
}

export const bind = Function.prototype.bind
  ? nativeBind
  : polyfillBind
```

第一，定义polyfillBind方法进行bind方法的兼容处理，该方法利用了闭包结构、通过Function.prototype中2个常用的上下文更改方法`call`,`apply`进行，其中内部实现逻辑为：

1.外部函数传入要绑定的函数fn,  上下文ctx
2.定义内部函数boundFn，并定义一个参数a
3.内部函数判断参数个数，采用对应的调用方式：

  0个：fn.call(ctx)

  1个：fn.call(ctx, a)

  1个以上：fn.apply(ctx, arguments)

4.设置内部函数boundFn._length = fn.length
5.最后返回内部函数，完成上下文绑定

第二，原生bind方法封装

第三，原生bind方法检测，进行bind方法的初始化

### 1.2.24 extend()

```js
export function extend (to: Object, _from: ?Object): Object {
  for (const key in _from) {
    to[key] = _from[key]
  }
  return to
}
```

从扩展对象属性，类似于Object.assign, 通过便利来源对象_from属性值并赋值到目标对象to。

### 1.2.24 toObject()

```js
export function toObject (arr: Array<any>): Object {
  const res = {}
  for (let i = 0; i < arr.length; i++) {
    if (arr[i]) {
      extend(res, arr[i])
    }
  }
  return res
}
```

合并数组中的多个对象到同一个新的对象，这个就是上文extend的批处理。

### 1.2.25 noop()

```js
export function noop (a?: any, b?: any, c?: any) {}
```

定义一个什么都不做的空函数，参数随便定义，就是什么都不做。

### 1.2.26 no()

```js
export const no = (a?: any, b?: any, c?: any) => false
```

总是返回false的一个函数。

### 1.2.27 toArray()

```js
export function toArray (list: any, start?: number): Array<any> {
  start = start || 0
  let i = list.length - start
  const ret: Array<any> = new Array(i)
  while (i--) {
    ret[i] = list[i + start]
  }
  return ret
}
```

将类数组变量转成数组，其中类数组有arguments，HTMLCollection等等，具有lenth属性，并且可利用下标index访问其元素的。

### 1.2.28 identity()

```js
export const identity = (_: any) => _
```

输入什么，返回什么～

### 1.2.29 genStaticKeys()

```js
export function genStaticKeys (modules: Array<ModuleOptions>): string {
  return modules.reduce((keys, m) => {
    return keys.concat(m.staticKeys || [])
  }, []).join(',')
}
```

将多个模块中的staticKeys合并成一个用','分隔的字符串。

### 1.2.30 looseEqual()

```js
export function looseEqual (a: any, b: any): boolean {
  if (a === b) return true
  const isObjectA = isObject(a)
  const isObjectB = isObject(b)
  if (isObjectA && isObjectB) {
    try {
      const isArrayA = Array.isArray(a)
      const isArrayB = Array.isArray(b)
      if (isArrayA && isArrayB) {
        return a.length === b.length && a.every((e, i) => {
          return looseEqual(e, b[i])
        })
      } else if (a instanceof Date && b instanceof Date) {
        return a.getTime() === b.getTime()
      } else if (!isArrayA && !isArrayB) {
        const keysA = Object.keys(a)
        const keysB = Object.keys(b)
        return keysA.length === keysB.length && keysA.every(key => {
          return looseEqual(a[key], b[key])
        })
      } else {
        /* istanbul ignore next */
        return false
      }
    } catch (e) {
      /* istanbul ignore next */
      return false
    }
  } else if (!isObjectA && !isObjectB) {
    return String(a) === String(b)
  } else {
    return false
  }
}
```

宽松的等值比较，比较两个值是否相等，这部分其实跟深拷贝deepClone逻辑非常相似，最主要有3大类判断逻辑：

1.基本值比较

2.数组比较

3.对象比较

让我们详细解析下：

```js
// 直接比较两个变量，简单粗暴，只要引用相同，就是同一份数据
if (a === b) return true

// 判断a、b是否为对象
const isObjectA = isObject(a)
const isObjectB = isObject(b)

// 很巧两个都是对象
if (isObjectA && isObjectB):

	// 判断两个是否为数组
  const isArrayA = Array.isArray(a)
  const isArrayB = Array.isArray(b)
  
  // 刚好两个都是数组
  if (isArrayA && isArrayB):
  	 // 判断两个数组的长度是否一致，并且每个元素都相等，这边直接用every方法然后再次调用
  	 // looseEqual进行递归判断
     return a.length === b.length && a.every((e, i) => {
       return looseEqual(e, b[i])
     })

 	// 其中一个非数组，且两个都是Date实例，直接获取时间戳比较
  if (a instanceof Date && b instanceof Date):
     return a.getTime() === b.getTime()

  // 两个都不是数组
  if (!isArrayA && !isArrayB):
    const keysA = Object.keys(a)
    const keysB = Object.keys(b)
    // 分别获取对应的keys比较keys个数、并且每个对应的key值相等
    // 再次利用looseEqual进行递归
    return keysA.length === keysB.length && keysA.every(key => {
      return looseEqual(a[key], b[key])
    })

	// 这边没有做例如RegExp的类型判断，而是直接套了try-catch,这边对于RegExp好像没做判断
  // 然而RegExp可以根据source和flags两个变量来进行比对，大概是默认每个RegExp都相等
  // 因为在keysA.length === keysB.length判断时候为true

// 两个都不是对象，也就是简单数值类型
if (!isObjectA && !isObjectB):
	// 返回基本数值类型对应的字符串形式
	// 利用String可以比较NaN，直接比较NaN将都为false
	return String(a) === String(b)

// 其他不符合情况都为false
```

### 1.2.31 looseIndexOf()

```js
export function looseIndexOf (arr: Array<mixed>, val: mixed): number {
  for (let i = 0; i < arr.length; i++) {
    if (looseEqual(arr[i], val)) return i
  }
  return -1
}
```

利用looseEqual查询目标值在数组中的index位置

### 1.2.32 once()

```js
export function once (fn: Function): Function {
  let called = false
  return function () {
    if (!called) {
      called = true
      fn.apply(this, arguments)
    }
  }
}
```

利用闭包模式，控制函数只运行一次