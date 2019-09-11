# 为数组方法打补丁，实现observer特性

在Vue中的数组是响应式的，当你对数组进行变更也将触发UI变动，例如：

```vue
<template>
	<ul>
    <li for="name in list" :key="name">{{name}}</li>
  </ul>
</template>
<script>
export default {
	data() {
    return {
      list: []
    }
  },
  mouted() {
    // li项目中会自动添加<li>Apple</li>元素
    this.list.push('Apple');
  }
}
</script>
```



```js
/*
 * not type checking this file because flow doesn't play well with
 * dynamically accessing methods on Array prototype
 */
// Object.defineProperty封装
import { def } from '../util/index'

const arrayProto = Array.prototype

// 创建Array.prototype对象拷贝
export const arrayMethods = Object.create(arrayProto)

// 需要打补丁的Array函数名称
const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

/**
 * Intercept mutating methods and emit events
 * 拦截变更方法并且触发ob事件
 */
methodsToPatch.forEach(function (method) {
  // cache original method
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator (...args) {
    const result = original.apply(this, args) // 调用原始方法
    const ob = this.__ob__ // 获取当前数组变量中的__ob__对象，
    let inserted
    // 当执行push、unshift、splice来插入新元素时候，需要对插入的元素实现ob注入
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    if (inserted) ob.observeArray(inserted)
    // notify change 通知变更
    ob.dep.notify()
    return result
  })
})

```

