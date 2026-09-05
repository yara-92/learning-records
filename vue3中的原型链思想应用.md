可以，而且这一块非常值得深入。

不过先纠正一个很重要的点：**Vue 3 的 `reactive()` 核心并不是靠 `Object.create()` 创建“代理对象”**。Vue 3 的响应式对象核心是 **ES Proxy**；`Object.create()` 是理解原型继承/委托的工具，但不是 `reactive()` 创建代理的主要机制。Vue 当前源码里，`reactive()` 最终围绕 `Proxy`、`Reflect.get/set`、依赖追踪，以及针对 Array/Map/Set 的特殊 instrumentation 展开。([GitHub][1])

而真正有意思的是：

> **Vue 3 响应式系统并不是“原型链实现的”，而是大量利用了 JavaScript 的对象模型、原型查找、Proxy、Reflect、函数调用和属性描述符等机制。**

我们可以把它拆成一层一层来看。

---

# 一、先建立 Vue 3 响应式系统的全景图

一个最普通的 Vue 代码：

```js
import { reactive, effect } from 'vue'

const state = reactive({
  count: 0
})

effect(() => {
  console.log(state.count)
})

state.count++
```

表面上：

```text
state.count++
```

非常简单。

实际上背后至少涉及：

```text
                    reactive()
                        │
                        ↓
                原始对象 raw
                        │
                        ↓
                     Proxy
                        │
             ┌──────────┴──────────┐
             ↓                     ↓
          get trap              set trap
             │                     │
             ↓                     ↓
           track()              trigger()
             │                     │
             ↓                     ↓
        记录依赖关系             找到依赖
             │                     │
             └──────────┬──────────┘
                        ↓
                      effect
                        ↓
                    重新执行
```

而原型链在哪里？

它其实深入到了：

```text
Proxy
 ↓
Reflect.get()
 ↓
对象属性查找
 ↓
[[Prototype]]
 ↓
Array.prototype / Object.prototype
 ↓
Vue 的 instrumentation
```

尤其是**数组**这里，非常精彩。

---

# 二、先从最简单的 `reactive()` 看

假设：

```js
const raw = {
  count: 0
}

const state = reactive(raw)
```

很多初学者会误以为：

```text
raw
 ↓
复制一份
 ↓
reactiveObject
```

不是。

更准确地说：

```text
raw object
   ↑
   │ target
   │
Proxy
```

也就是说：

```js
state !== raw
```

但：

```js
toRaw(state) === raw
```

Vue 官方文档也明确说明，`reactive()` 返回的是 JavaScript Proxy，并且默认进行深层响应式转换。([GitHub][2])

可以把结构理解成：

```text
                 ┌─────────────────┐
                 │      raw        │
                 │ { count: 0 }    │
                 └────────▲────────┘
                          │
                          │ target
                          │
                 ┌────────┴────────┐
                 │     Proxy       │
                 │                 │
                 │ get / set ...   │
                 └────────┬────────┘
                          │
                          ↓
                        state
```

注意：

> Proxy 不是“继承” raw。

它是：

> **代理 raw 的操作。**

这和传统 prototype inheritance 是两种不同机制。

---

# 三、那么原型链到底怎么进来了？

关键就在：

```js
state.foo
```

JavaScript 会执行一次属性访问。

Vue 可以通过 Proxy 拦截：

```js
get(target, key, receiver)
```

当前 Vue 3 源码的 `baseHandlers.ts` 中，确实可以看到 `get` handler 最终使用：

```js
Reflect.get(target, key, receiver)
```

而且在此之前针对数组进行了特殊处理。([GitHub][1])

也就是说：

```text
state.foo
   ↓
Proxy [[Get]]
   ↓
Vue get trap
   ↓
Reflect.get(target, key, receiver)
   ↓
JavaScript 原生属性查找
   ↓
own property
   ↓
[[Prototype]]
   ↓
...
```

所以 Vue 并没有自己重新实现 JavaScript 的整个属性查找系统。

它利用了：

> **Proxy + Reflect + JavaScript 原生对象模型。**

---

# 四、为什么 Vue 使用 `Reflect.get()` 而不是 `target[key]`？

这是非常值得理解的。

简单写：

```js
get(target, key, receiver) {
  return target[key]
}
```

当然可以。

但是 Vue 使用：

```js
Reflect.get(target, key, receiver)
```

原因之一就是：

> **需要正确保留 JavaScript 的 receiver 语义。**

看一个 getter：

```js
const obj = {
  get name() {
    return this._name
  },

  _name: 'Alice'
}
```

如果：

```js
Reflect.get(obj, 'name', receiver)
```

getter 中：

```js
this
```

可以是：

```text
receiver
```

这对于 Proxy 特别重要。

因为 Vue 不是简单地：

```text
“帮你读取 target”
```

而是要让：

```text
this
getter
setter
prototype
Proxy
```

这些机制尽可能保持 JavaScript 原生语义。

所以：

```js
Reflect.get(target, key, receiver)
```

在 Proxy handler 里非常常见。

---

# 五、这就产生了一个很漂亮的组合

你可以把 Vue 的属性读取理解成：

```text
                    state.foo
                       │
                       ↓
                     Proxy
                       │
                       ↓
                  get(target)
                       │
                       ↓
                Vue track(...)
                       │
                       ↓
              Reflect.get(...)
                       │
                       ↓
             JavaScript 属性查找
                       │
                ┌──────┴──────┐
                ↓             ↓
             own prop      prototype
                              │
                              ↓
                          prototype...
```

所以：

> **Vue 响应式系统并没有替代原型链，而是在原生属性访问机制外面“包了一层”。**

这个理解非常重要。

---

# 六、真正精彩的地方来了：数组

普通对象：

```js
const state = reactive({
  name: 'Alice'
})
```

比较简单。

但是：

```js
const state = reactive([])
```

事情马上复杂很多。

因为数组不只是：

```text
0
1
2
length
```

它还拥有：

```text
Array.prototype
```

上面大量的方法：

```js
map
filter
find
includes
indexOf
push
pop
shift
unshift
splice
slice
forEach
reduce
...
```

也就是说：

```text
state
 ↓
Proxy
 ↓
Array.prototype
 ↓
Object.prototype
```

Vue 必须处理：

> **数组原生方法 + Proxy + 依赖追踪**

这就是 `arrayInstrumentations` 存在的原因。

当前 Vue 源码专门维护了一个 `arrayInstrumentations` 对象，用来处理大量数组方法。([GitHub][3])

---

# 七、普通数组调用到底发生了什么？

例如：

```js
const arr = []
arr.push(1)
```

JavaScript 本来会：

```text
arr.push
   ↓
Array.prototype.push
```

也就是说：

```js
arr.push === Array.prototype.push
```

通常为：

```text
true
```

因为：

```text
arr
 ↓
Array.prototype
 ↓
push
```

---

# 八、但 Vue 的 reactive array 不希望简单地这么干

假设：

```js
const arr = reactive([])
```

然后：

```js
arr.push(1)
```

如果完全不处理：

```text
Proxy
 ↓
Array.prototype.push
 ↓
内部操作 length
 ↓
触发 get/set
 ↓
track/trigger
```

很容易产生一些非常复杂的依赖问题。

特别是：

```text
length
```

是数组响应式系统中的一个特殊问题。

Vue 源码因此专门对：

```text
push
pop
shift
unshift
splice
```

等会修改数组长度的方法进行了特殊处理。

当前源码中的 `noTracking()` 会：

```text
pauseTracking()
startBatch()
...
endBatch()
resetTracking()
```

然后对 raw 数组执行对应方法。([GitHub][3])

---

# 九、为什么 `push()` 要暂停 tracking？

这是一个非常好的例子。

假设：

```js
const state = reactive([])

effect(() => {
  state.push(1)
})
```

如果 Vue 在执行：

```js
push()
```

过程中把所有内部读取也拿去建立依赖：

```text
effect
 ↓
push
 ↓
读取 length
 ↓
track(length)
```

然后：

```text
push
 ↓
修改 length
 ↓
trigger(length)
 ↓
effect
 ↓
push
 ↓
...
```

可能形成：

```text
effect
 ↓
push
 ↓
trigger
 ↓
effect
 ↓
push
 ↓
trigger
 ↓
...
```

所以 Vue 对这些长度变化型数组方法做特殊处理。

源码注释也明确指出，这么做是为了避免跟踪 `length` 导致无限循环。([GitHub][3])

这就是：

> **Proxy 并不是“拦截一下 get/set 就完事了”。**

真正的响应式系统必须理解 JavaScript 内建对象的复杂行为。

---

# 十、Vue 的数组 instrumentation 本质是什么？

可以把它理解成：

```text
原生 Array.prototype
        │
        │
        ↓
Vue instrumentation layer
        │
        ↓
Proxy get trap
```

例如：

```js
arr.push
```

Vue 的 Proxy `get` handler 会先检查：

```text
这是数组吗？
       │
       ↓
有没有对应的 instrumentation？
       │
       ↓
有
       │
       ↓
返回 Vue 特殊包装的方法
```

当前 `baseHandlers.ts` 中确实存在这样的逻辑：如果 target 是数组，并且 `arrayInstrumentations[key]` 存在，就优先返回 instrumentation。([GitHub][1])

这非常关键。

---

# 十一、这其实就是“原型查找 + Proxy 拦截”的组合

假设：

```js
const arr = reactive([])
```

执行：

```js
arr.push
```

普通 JS：

```text
arr
 ↓
[[Prototype]]
 ↓
Array.prototype
 ↓
push
```

Vue reactive：

```text
arr
 ↓
Proxy get trap
 ↓
检查 arrayInstrumentations
 ↓
找到 push
 ↓
返回 Vue 包装后的 push
```

所以 Vue 在某种意义上：

> **抢在 JavaScript 原型链真正找到 `Array.prototype.push` 之前，替换了这个属性访问结果。**

这是非常漂亮的设计。

---

# 十二、注意：不是修改了 `Array.prototype`

这点非常重要。

Vue 并没有：

```js
Array.prototype.push = ...
```

否则整个页面所有数组都会被污染。

它实际上做的是：

```text
reactive array
      ↓
Proxy
      ↓
get('push')
      ↓
Vue instrumentation
```

所以：

```js
const a = []
const b = reactive([])

a.push
```

和：

```js
b.push
```

可以表现不同。

但：

```js
Array.prototype.push
```

本身并没有被 Vue 全局替换。

这是一个非常重要的工程设计：

> **局部代理，而不是全局 monkey patch。**

---

# 十三、Vue 为什么自己维护一个 `arrayInstrumentations` 对象？

当前源码里：

```js
export const arrayInstrumentations = {
    ...
}
```

而且值得注意的是，它本身初始化时用了：

```js
__proto__: null
```

也就是说：

```js
const arrayInstrumentations = {
    __proto__: null
}
```

这就又回到了我们上一轮讲的：

> `Object.create(null)` / null prototype。

当前 Vue 源码中的 `arrayInstrumentations` 明确采用：

```js
__proto__: null
```

这样的对象字面量初始化方式。([GitHub][3])

---

# 十四、为什么这里喜欢 null prototype？

假设：

```js
const map = {
    push: fn
}
```

那么：

```text
map
 ↓
Object.prototype
```

意味着：

```js
map.toString
map.constructor
map.hasOwnProperty
```

等属性也存在。

而：

```js
const map = {
    __proto__: null,

    push: fn
}
```

变成：

```text
map
 ↓
null
```

它就是一个更加纯粹的：

```text
key → handler
```

字典。

Vue 这里把：

```text
array method name
        ↓
instrumentation function
```

作为一个内部 lookup table。

所以 null prototype 很合适。

---

# 十五、这就是原型链在 Vue 里面一个非常直接的实际应用

我们上一轮说：

```js
Object.create(null)
```

适合构造没有 Object.prototype 干扰的字典。

现在 Vue 内部就出现了非常类似的设计：

```js
{
    __proto__: null,

    push() {},
    pop() {},
    includes() {},
    indexOf() {},
    ...
}
```

然后：

```js
arrayInstrumentations[key]
```

直接查找对应的包装方法。

所以你可以看到：

> **原型链不是只存在于面试题里。Vue 这种大型框架源码里，会大量利用 JavaScript 对象模型的细节。**

---

# 十六、再看一个非常漂亮的例子：`includes()` 和 `indexOf()`

这两个方法有一个特别麻烦的问题：

```js
includes
indexOf
lastIndexOf
```

它们涉及：

> **对象身份比较。**

考虑：

```js
const raw = {
  id: 1
}

const state = reactive([raw])
```

那么：

```js
state[0] === raw
```

通常：

```text
false
```

因为：

```text
raw
```

和：

```text
Proxy(raw)
```

不是同一个对象。

---

# 十七、于是一个非常诡异的问题出现了

假设：

```js
const raw = { id: 1 }

const arr = reactive([raw])
```

然后：

```js
arr.includes(raw)
```

用户的直觉：

> “数组里明明有 raw，为什么找不到？”

因为数组内部可能存的是：

```text
Proxy(raw)
```

而你查的是：

```text
raw
```

对象身份：

```text
Proxy(raw) !== raw
```

于是：

```text
=== 
```

失败。

---

# 十八、Vue 如何解决？

当前 Vue 的 `arrayInstrumentations` 中，对：

```text
includes
indexOf
lastIndexOf
```

专门走 `searchProxy()`。

源码的策略很漂亮：

```text
第一次：
使用原始参数搜索

      ↓

失败？

      ↓

如果参数是 Proxy

      ↓

toRaw(argument)

      ↓

再次搜索
```

源码注释明确说明，这些方法需要特殊处理 reactive proxy 的身份问题。([GitHub][3])

简化理解：

```js
const arr = toRaw(self)

const result = arr[method](...args)

if (failed && isProxy(args[0])) {
    args[0] = toRaw(args[0])

    return arr[method](...args)
}
```

这就是 Vue 为了保持：

> **raw object 与 reactive proxy 的使用体验尽可能一致**

而做的兼容层。

---

# 十九、这时候你会发现：Vue 响应式真正复杂的地方不是 Proxy

很多人学习 Vue 3 源码时认为：

```text
reactive
=
new Proxy
```

其实远远不是。

真正的：

```text
reactive
```

大概可以理解成：

```text
                  reactive()
                      │
                      ↓
                    Proxy
                      │
         ┌────────────┼─────────────┐
         ↓            ↓             ↓
       Object        Array       Map/Set
         │            │             │
         ↓            ↓             ↓
      baseHandler  arrayInstr.   collection
         │            │             │
         └────────────┼─────────────┘
                      ↓
                    track
                      ↓
                  dependency
                      ↓
                   trigger
                      ↓
                    effect
```

Vue 当前源码也明确把普通对象/数组与 Map/Set/WeakMap/WeakSet 区分成不同 target 类型。([GitHub][4])

---

# 二十、再看 Vue 的 `get` handler

可以把它简化成：

```js
const mutableHandlers = {
  get(target, key, receiver) {

    // 1. Vue 自己的特殊 key
    // 2. 数组 instrumentation
    // 3. Reflect.get
    // 4. track
    // 5. ref 解包
    // 6. nested object reactive
  }
}
```

实际源码比这个复杂得多，但整体思路就是这样。

当前源码中可以看到：

```text
ReactiveFlags
arrayInstrumentations
Reflect.get
track
ref
reactive
```

这些机制都在这个 `get` handler 中汇合。([GitHub][1])

---

# 二十一、这里又出现了一个非常有意思的原型链细节

源码里有：

```js
const isNonTrackableKeys =
  makeMap(`__proto__,__v_isRef,__isVue`)
```

也就是说：

```text
__proto__
__v_isRef
__isVue
```

等 key 有特殊处理。

尤其：

```text
__proto__
```

刚好就是我们上一轮讨论的：

> JavaScript 原型访问相关机制。

Vue 必须非常谨慎处理这些特殊属性。

否则响应式系统可能：

```text
track prototype internals
```

造成奇怪的依赖关系。

所以：

> **连 `__proto__` 这种原型链细节，都进入了 Vue reactive handler 的设计范围。**

---

# 二十二、再看 Vue 的深层响应式

例如：

```js
const state = reactive({
  user: {
    profile: {
      name: 'Alice'
    }
  }
})
```

执行：

```js
state.user.profile.name
```

并不是：

```text
reactive()
一次性把整个树复制一遍
```

而是访问过程中逐层转换。

大概：

```text
state
 ↓ Proxy
user
 ↓
reactive(user)
 ↓ Proxy
profile
 ↓
reactive(profile)
 ↓ Proxy
name
```

所以：

```text
Proxy
  ↓
Proxy
  ↓
Proxy
```

会形成一层一层的代理。

Vue 官方文档也明确说明 `reactive()` 是 deep reactive，嵌套对象会在访问时被转换为 reactive。([GitHub][2])

---

# 二十三、这和 prototype 是什么关系？

假设：

```js
const raw = {
  user: {}
}
```

那么：

```text
Proxy(raw)
```

本身并不是：

```text
Proxy(raw)
 ↓
raw
```

这样的原型继承关系。

而是：

```text
Proxy
  │
  │ [[ProxyTarget]]
  ↓
raw
```

这是非常重要的区别。

所以不要把：

```text
Proxy
```

理解成：

```text
prototype
```

两者完全不同：

| 机制              | 含义                   |
| --------------- | -------------------- |
| `[[Prototype]]` | 对象属性查找/委托关系          |
| Proxy           | 拦截对象操作               |
| Reflect         | 调用标准内部对象操作语义         |
| reactive        | Vue 利用 Proxy 构建响应式代理 |

---

# 二十四、Proxy 和 prototype 可以同时存在

这是理解 Vue 源码的关键。

例如：

```js
const proto = {
  hello() {
    console.log('hello')
  }
}

const raw = Object.create(proto)

const state = reactive(raw)
```

此时：

```text
state
 ↓ Proxy
raw
 ↓ [[Prototype]]
proto
 ↓
Object.prototype
 ↓
null
```

于是：

```js
state.hello()
```

会发生：

```text
state
 ↓
Proxy get
 ↓
Reflect.get(raw, 'hello', state)
 ↓
raw 自己没有 hello
 ↓
[[Prototype]]
 ↓
proto.hello
 ↓
调用
```

这就是：

> **Proxy 和原型链叠加起来工作。**

---

# 二十五、为什么 `Reflect.get(target, key, receiver)` 特别重要？

继续：

```js
const proto = {
  get name() {
    return this._name
  }
}

const raw = Object.create(proto)

raw._name = 'Alice'

const state = reactive(raw)
```

访问：

```js
state.name
```

Vue：

```text
Proxy
 ↓
Reflect.get(raw, 'name', state)
 ↓
prototype getter
 ↓
getter this = state
 ↓
state._name
 ↓
再次进入 Proxy get
 ↓
track(_name)
```

注意这个非常漂亮的效果：

```text
getter
  ↓
this
  ↓
reactive proxy
  ↓
再次触发 reactive
```

这就是为什么 Vue 的 Proxy handler 必须非常认真地处理：

```text
receiver
```

而不是简单：

```js
target[key]
```

---

# 二十六、这实际上让“原型方法”也具备响应式能力

例如：

```js
const proto = {
  get total() {
    return this.price * this.count
  }
}

const state = reactive(
  Object.assign(
    Object.create(proto),
    {
      price: 100,
      count: 2
    }
  )
)
```

然后：

```js
effect(() => {
  console.log(state.total)
})
```

执行：

```text
state.total
 ↓
Proxy
 ↓
Reflect.get
 ↓
proto.total getter
 ↓
this === state
 ↓
state.price
 ↓
track(price)
 ↓
state.count
 ↓
track(count)
```

所以：

```js
state.price++
```

会重新触发：

```text
state.total
```

这就是：

> **Vue 响应式 + JavaScript 原型链 + getter/setter + receiver**

共同工作的结果。

---

# 二十七、这也解释了 Vue 为什么不能简单“自己实现属性查找”

假设 Vue 不使用：

```js
Reflect.get()
```

而自己写：

```js
if (target.hasOwnProperty(key)) {
    return target[key]
}

return undefined
```

那会破坏大量 JavaScript 语义：

```text
prototype
getter
setter
receiver
Symbol
property descriptor
继承
```

因此 Vue 更合理的策略是：

> **让 JavaScript 自己负责对象语义，Vue 只负责在关键位置插入 track/trigger。**

这其实是一个非常优秀的框架设计思想。

---

# 二十八、再深入一步：Vue 的响应式依赖到底是什么？

例如：

```js
const state = reactive({
  count: 0
})

effect(() => {
  console.log(state.count)
})
```

当执行：

```js
state.count
```

Vue 会建立：

```text
target
  ↓
key
  ↓
Dep
  ↓
effect
```

可以粗略理解：

```text
WeakMap
 └── raw object
      └── Map
           └── "count"
                └── Set
                     └── effect
```

即：

```text
raw target
   ↓
property key
   ↓
dependency set
   ↓
effects
```

这部分实际上已经进入：

> **数据结构设计**

而不是单纯的原型链。

---

# 二十九、所以 Vue 3 响应式其实是“四层技术叠加”

这是我认为最值得你建立的整体认识：

### 第一层：JavaScript 对象模型

```text
property
[[Prototype]]
getter
setter
receiver
```

↓

### 第二层：Proxy / Reflect

```text
get
set
has
deleteProperty
ownKeys
```

↓

### 第三层：Vue Dependency Tracking

```text
track()
trigger()
Dep
effect
```

↓

### 第四层：框架调度

```text
scheduler
batch
component update
render
DOM patch
```

最终：

```text
JavaScript
   ↓
Proxy / Reflect
   ↓
Vue Reactivity
   ↓
Component
   ↓
Renderer
   ↓
DOM
```

这才是 Vue 3 响应式真正的体系。

---

# 三十、把数组再深入一层：`map()` 为什么也需要 instrumentation？

看：

```js
const users = reactive([
  { name: 'Alice' },
  { name: 'Bob' }
])

effect(() => {
  console.log(
    users.map(user => user.name)
  )
})
```

这里涉及：

```text
users
 ↓
map
 ↓
遍历数组
 ↓
每个元素
 ↓
user.name
```

Vue 必须知道：

```text
这个 effect 依赖：
    users 的迭代
    users[0].name
    users[1].name
```

所以当前 Vue 的 array instrumentation 对：

```text
every
filter
find
findIndex
forEach
map
reduce
reduceRight
...
```

都有特殊处理。([GitHub][3])

---

# 三十一、这里出现一个非常关键的概念：数组迭代依赖

Vue 源码里有：

```text
ARRAY_ITERATE_KEY
```

这是非常值得注意的设计。

为什么？

因为：

```js
users.map(...)
```

不是单纯读取：

```js
users[0]
```

它还表达了：

> **“我关心这个数组整体的迭代结果。”**

所以 Vue 需要一种依赖：

```text
ARRAY_ITERATE_KEY
```

来表示：

> 这个 effect 依赖数组的迭代状态。

当前源码的 `reactiveReadArray()` 会对数组的 `ARRAY_ITERATE_KEY` 做 tracking。([GitHub][3])

---

# 三十二、为什么不能只 track `length`？

因为：

```js
users.map(...)
```

关心的不只是：

```text
length
```

还关心：

```text
元素
顺序
新增
删除
替换
```

例如：

```js
users.push(newUser)
```

数组长度变化。

但：

```js
users[0] = anotherUser
```

长度可能完全不变。

如果只追踪：

```text
length
```

就无法正确表达：

> 数组内容发生变化。

因此 Vue 引入：

```text
ARRAY_ITERATE_KEY
```

这样的依赖语义。

这其实已经从：

> “监听属性”

进化到了：

> **“监听数据结构操作语义”。**

---

# 三十三、这就是 Vue 源码越来越有意思的地方

初级理解：

```text
reactive
=
Proxy
```

中级理解：

```text
Proxy
+
track
+
trigger
```

高级理解：

```text
Proxy
+
Reflect
+
prototype
+
receiver
+
array instrumentation
+
collection instrumentation
+
raw/proxy identity
+
dependency graph
+
batching
+
scheduler
```

真正读 Vue 3 reactivity 源码时，就是在理解这一整套系统。

---

# 三十四、Map / Set 又是另一种世界

例如：

```js
const state = reactive(
  new Map()
)

state.set('name', 'Alice')
```

普通：

```js
Proxy
```

并不能简单按照普通对象：

```js
target[key]
```

来处理。

因为：

```js
map.get(key)
map.set(key, value)
map.has(key)
map.delete(key)
map.keys()
map.values()
map.entries()
```

这些方法背后依赖 Map 的内部槽和特殊语义。

所以 Vue 对：

```text
Map
Set
WeakMap
WeakSet
```

采用专门的 collection handlers。

Vue 源码的 target type mapping 也明确将普通 Object/Array 与这些集合类型区分开。([GitHub][4])

---

# 三十五、所以 Vue 的 handler 大体可以理解为

```text
                    reactive(target)
                           │
             ┌─────────────┴─────────────┐
             ↓                           ↓
         Object / Array            Map / Set / WeakMap...
             │                           │
             ↓                           ↓
       baseHandlers             collectionHandlers
             │                           │
             ↓                           ↓
       get / set ...              get / set / has ...
             │                           │
             └─────────────┬─────────────┘
                           ↓
                     track / trigger
```

这就是为什么 Vue 源码不会只有一个简单的：

```js
new Proxy(target, handler)
```

而是一整套 handler 系统。

---

# 三十六、回头看“原型链”，你会发现它在 Vue 中扮演的角色其实很微妙

它不是：

```text
Vue Reactive
=
Prototype Chain
```

而是：

```text
Vue Reactive
=
Proxy
+
JavaScript Object Model
```

其中 JavaScript Object Model 又包括：

```text
[[Prototype]]
property lookup
getter/setter
receiver
Array.prototype
Object.prototype
Function.prototype
```

因此：

> **原型链是 Vue 响应式系统所依赖的 JavaScript 对象语义之一，而不是 Vue 响应式的核心实现机制本身。**

这个区分非常重要。

---

# 三十七、如果你以后读 Vue 源码，建议按照这个顺序

不要一上来就啃 500 行源码。

按照：

```text
第一阶段
reactive()
 ↓
Proxy
 ↓
get / set
```

先搞明白。

然后：

```text
第二阶段
track()
 ↓
trigger()
 ↓
effect()
```

然后：

```text
第三阶段
arrayInstrumentations
```

重点研究：

```text
push
pop
splice
includes
indexOf
map
filter
forEach
```

然后：

```text
第四阶段
collectionHandlers
```

重点：

```text
Map
Set
WeakMap
WeakSet
```

最后：

```text
第五阶段
computed
watch
scheduler
batch
component rendering
```

这样会非常顺。

---

# 三十八、我特别建议你自己做一个“Mini Vue Reactive”

因为你现在已经把原型链搞清楚了，这时候自己实现一个极简版本非常合适。

第一版只需要：

```js
const bucket = new WeakMap()

let activeEffect

function effect(fn) {
  activeEffect = fn
  fn()
  activeEffect = null
}

function track(target, key) {
  // 建立依赖
}

function trigger(target, key) {
  // 执行依赖
}

function reactive(target) {
  return new Proxy(target, {
    get(target, key, receiver) {
      track(target, key)

      return Reflect.get(
        target,
        key,
        receiver
      )
    },

    set(target, key, value, receiver) {
      const result = Reflect.set(
        target,
        key,
        value,
        receiver
      )

      trigger(target, key)

      return result
    }
  })
}
```

然后：

```js
const state = reactive({
  count: 0
})

effect(() => {
  console.log(state.count)
})

state.count++
```

你会真正看到：

```text
state.count
   ↓
Proxy.get
   ↓
Reflect.get
   ↓
track
   ↓
effect
```

然后：

```text
state.count++
   ↓
Proxy.set
   ↓
Reflect.set
   ↓
trigger
   ↓
effect()
```

这比单纯读 Vue 文档更容易真正理解。

---

# 三十九、第二版再加入数组

然后实现：

```js
const arrayInstrumentations = {
  push() {},
  pop() {},
  includes() {},
  indexOf() {}
}
```

然后：

```js
get(target, key, receiver) {

    if (
        Array.isArray(target) &&
        arrayInstrumentations[key]
    ) {
        return arrayInstrumentations[key]
    }

    return Reflect.get(
        target,
        key,
        receiver
    )
}
```

此时你就能亲眼看到：

```text
reactiveArray.push
       ↓
Proxy.get
       ↓
arrayInstrumentations.push
```

而不是：

```text
Array.prototype.push
```

直接执行。

这一步会让你对 Vue 的源码理解提升非常明显。

---

# 四十、最终把整个知识体系串起来

我们这一轮讲完以后，可以把：

**JavaScript 原型链 → Vue 3 Reactive**

串成这样：

```text
                    JavaScript Object Model
                             │
             ┌───────────────┼────────────────┐
             ↓               ↓                ↓
        [[Prototype]]     getter/setter     receiver
             │               │                │
             └───────────────┼────────────────┘
                             ↓
                         Proxy / Reflect
                             │
                    ┌────────┴────────┐
                    ↓                 ↓
                  Object            Array
                    │                 │
                    │          Array.prototype
                    │                 │
                    │          Vue instrumentation
                    │                 │
                    └────────┬────────┘
                             ↓
                          reactive()
                             │
                             ↓
                           track
                             │
                             ↓
                         dependency
                             │
                             ↓
                          trigger
                             │
                             ↓
                           effect
                             │
                             ↓
                         scheduler
                             │
                             ↓
                      Vue component update
                             │
                             ↓
                            DOM
```

这里面最值得记住的其实不是某一行源码，而是一个设计思想：

> **Vue 3 没有重新发明 JavaScript 对象系统，而是利用 Proxy/Reflect 在原生对象语义的关键入口处插入“依赖追踪”和“依赖触发”。**

而数组又进一步告诉我们：

> **真正成熟的响应式系统，不能只拦截 `get/set`，还必须理解 JavaScript 内建对象、原型方法、迭代、身份比较、length、Map/Set 等复杂语义。**

这也是为什么 Vue 3 的 reactivity 源码看起来比“`new Proxy()`”复杂得多。当前 Vue 源码的 `baseHandlers.ts`、`arrayInstrumentations.ts`、`reactive.ts` 正好把这套设计完整体现出来。([GitHub][1])
