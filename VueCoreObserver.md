# Vue的观察者机制Observer

Vue的观察者模式有三个概念，一个是观察器watcher，一个是订阅处理器Dep，一个是观察者Observer

他们之间的关系是，属性变量绑定会生成一个观察器watcher，只要表达式中的变量改变就会触发，多个观察器可以统一订阅到Dep订阅处理器下，最后由Observer统一处理函数回调的执行。

## 1. Watcher观察器

