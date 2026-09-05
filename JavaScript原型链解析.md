JavaScript 的**原型链（Prototype Chain）**是 JS 最核心、也最容易被“背 API”搞混的一套机制。

很多人知道：

```js
obj.__proto__
Object.prototype
Constructor.prototype
```

但一旦问：

* `new` 到底做了什么？
* `obj.foo` 到底去哪里找？
* `obj.__proto__ === Constructor.prototype` 为什么成立？
* `Object.create()` 和 `new` 有什么区别？
* `class extends` 和原型链是什么关系？
* `instanceof` 到底判断什么？
* 为什么 `Object.prototype` 的原型是 `null`？
* `constructor` 是不是对象真正的“类型”？
* 原型链和作用域链是不是一回事？

就很容易混乱。

下面从**对象模型 → prototype → `[[Prototype]]` → 属性查找 → new → class → instanceof → 继承 → 性能 → 常见陷阱**完整拆开。

---

# 一、先建立最重要的认知：JS 世界里有两条不同的“链”

学习原型链之前，首先要把两个概念分开：

> **原型链 ≠ 作用域链**

例如：

```js
const name = "Tom";

function foo() {
    console.log(name);
}
```

这里寻找 `name`：

```text
foo
 ↓
词法环境
 ↓
外层词法环境
 ↓
全局环境
```

这是**作用域/词法环境链**。

而：

```js
const obj = {
    name: "Tom"
};

console.log(obj.toString);
```

`obj` 自己没有 `toString`，但是还能找到：

```text
obj
 ↓
Object.prototype
 ↓
null
```

这是**原型链**。

所以：

| 机制     | 解决什么问题        |
| ------ | ------------- |
| 作用域链   | 变量在哪里找        |
| 原型链    | 对象属性/方法在哪里找   |
| `this` | 当前调用的对象是谁     |
| 闭包     | 函数如何保留外部变量    |
| 原型继承   | 对象之间如何共享属性/方法 |

这几个机制经常同时出现，但它们不是一回事。

---

# 二、JavaScript 的对象到底是什么？

先看：

```js
const user = {
    name: "Alice",
    age: 20
};
```

我们通常理解成：

```text
user
├── name
└── age
```

但从 JS 内部对象模型看，它还有一个非常重要的内部槽：

```text
user
├── own properties
│   ├── name
│   └── age
│
└── [[Prototype]]
```

`[[Prototype]]` 是一个**内部槽（internal slot）**。

它指向另一个对象。

例如：

```js
Object.getPrototypeOf(user)
```

结果：

```js
Object.prototype
```

所以可以理解成：

```text
user
  │
  │ [[Prototype]]
  ↓
Object.prototype
  │
  │ [[Prototype]]
  ↓
null
```

这就是最基本的原型链。

---

# 三、`__proto__` 到底是什么？

这是学习原型链最容易踩的坑。

很多教程直接说：

> `__proto__` 就是原型。

这种说法**不够严谨**。

真正的内部机制是：

```text
[[Prototype]]
```

而：

```js
obj.__proto__
```

是访问这个内部原型的一种历史遗留接口。

例如：

```js
const obj = {};

obj.__proto__ === Object.prototype;
```

通常是：

```text
true
```

更推荐：

```js
Object.getPrototypeOf(obj)
```

例如：

```js
Object.getPrototypeOf(obj) === Object.prototype
// true
```

设置原型则可以：

```js
Object.setPrototypeOf(obj, somePrototype);
```

不过实际开发中**不推荐频繁使用 `Object.setPrototypeOf()`**，后面会讲性能原因。

---

# 四、最容易混淆的三个东西

一定要区分：

```text
prototype
[[Prototype]]
constructor
```

这是原型链学习的核心。

---

## 1. `[[Prototype]]`

它是：

> **某个对象指向另一个对象的内部链接。**

例如：

```js
const obj = {};
```

可以理解成：

```text
obj
 ↓ [[Prototype]]
Object.prototype
```

---

## 2. `prototype`

`prototype` 通常出现在：

> **函数对象上**

例如：

```js
function Person() {}

console.log(Person.prototype);
```

得到：

```text
{
    constructor: Person
}
```

所以：

```js
Person.prototype
```

本身也是一个对象。

---

## 3. `constructor`

默认情况下：

```js
Person.prototype.constructor === Person
```

成立。

因此：

```text
Person
   ↑
   │ constructor
   │
Person.prototype
```

形成了一个反向引用。

---

# 五、最关键的一张图

假设：

```js
function Person(name) {
    this.name = name;
}

const p = new Person("Alice");
```

此时关系是：

```text
                 ┌──────────────────────┐
                 │       Person         │
                 │      function       │
                 └──────────┬───────────┘
                            │
                            │ prototype
                            ↓
                 ┌──────────────────────┐
                 │   Person.prototype   │
                 │                      │
                 │ constructor ─────────┼──→ Person
                 │                      │
                 │ sayHello()           │
                 └──────────┬───────────┘
                            ▲
                            │ [[Prototype]]
                            │
                 ┌──────────┴───────────┐
                 │          p           │
                 │                      │
                 │ name: "Alice"        │
                 └──────────┬───────────┘
                            │
                            │ [[Prototype]]
                            ↓
                 Person.prototype
                            │
                            ↓
                   Object.prototype
                            │
                            ↓
                           null
```

所以：

```js
p.__proto__ === Person.prototype
```

以及：

```js
Object.getPrototypeOf(p) === Person.prototype
```

都成立。

而：

```js
Person.prototype.__proto__ === Object.prototype
```

也成立。

最终：

```text
p
↓
Person.prototype
↓
Object.prototype
↓
null
```

这就是完整原型链。

---

# 六、为什么 `new Person()` 会建立这个关系？

这是理解原型链最重要的地方。

代码：

```js
const p = new Person("Alice");
```

可以近似理解成几个步骤。

---

## 第一步：创建一个新对象

类似：

```js
const p = {};
```

但是还不完全一样。

---

## 第二步：设置新对象的原型

将：

```js
p.[[Prototype]]
```

设置为：

```js
Person.prototype
```

于是：

```text
p
 ↓
Person.prototype
```

---

## 第三步：调用构造函数

执行：

```js
Person.call(p, "Alice");
```

于是：

```js
this === p
```

最终：

```js
p.name = "Alice";
```

---

## 第四步：返回对象

所以：

```js
const p = new Person("Alice");
```

可以粗略模拟成：

```js
const p = Object.create(Person.prototype);

Person.call(p, "Alice");

return p;
```

这也是为什么很多人理解 `new` 时，最应该记住：

```js
Object.create(Constructor.prototype)
```

这一层。

---

# 七、`new` 的更准确理解

如果我们手写一个简化版：

```js
function myNew(Constructor, ...args) {
    const obj = Object.create(Constructor.prototype);

    const result = Constructor.apply(obj, args);

    if (result !== null &&
        (typeof result === "object" ||
         typeof result === "function")) {
        return result;
    }

    return obj;
}
```

于是：

```js
function Person(name) {
    this.name = name;
}

const p = myNew(Person, "Alice");
```

效果类似：

```js
const p = new Person("Alice");
```

当然，真正 ECMAScript 规范中的 `new` 涉及：

* `[[Construct]]`
* `new.target`
* constructor result
* prototype 获取
* exotic objects
* derived constructor

比这个复杂。

但是对于理解原型链：

> `Object.create(Constructor.prototype)` 是非常关键的模型。

---

# 八、为什么方法应该放到 prototype？

看两个写法。

### 写法 A

```js
function Person(name) {
    this.name = name;

    this.sayHello = function () {
        console.log("hello");
    };
}
```

每创建一个对象：

```js
const a = new Person("A");
const b = new Person("B");
```

实际上：

```text
a
└── sayHello → function A

b
└── sayHello → function B
```

两个函数不是同一个函数：

```js
a.sayHello === b.sayHello
```

结果：

```text
false
```

---

## 写法 B：放到 prototype

```js
function Person(name) {
    this.name = name;
}

Person.prototype.sayHello = function () {
    console.log("hello");
};
```

现在：

```text
a
├── name
│
└── [[Prototype]]
       ↓
   Person.prototype
       │
       └── sayHello
```

以及：

```text
b
├── name
│
└── [[Prototype]]
       ↓
   Person.prototype
       │
       └── sayHello
```

所以：

```js
a.sayHello === b.sayHello
```

结果：

```text
true
```

这就是原型的核心价值：

> **共享行为。**

---

# 九、属性查找到底是怎么进行的？

这是原型链最核心的运行机制。

执行：

```js
p.name
```

JavaScript 并不是直接问：

> “Person 有没有 name？”

而是进行类似这样的属性查找：

```text
p 自己有没有 name？
   │
   ├── 有 → 返回
   │
   └── 没有
        ↓
Person.prototype 有没有 name？
        │
        ├── 有 → 返回
        │
        └── 没有
             ↓
Object.prototype 有没有 name？
             │
             ├── 有 → 返回
             │
             └── 没有
                  ↓
                 null
                  ↓
               undefined
```

例如：

```js
const p = new Person("Alice");

console.log(p.toString);
```

`p` 没有：

```text
toString
```

继续：

```text
Person.prototype
```

没有。

继续：

```text
Object.prototype
```

找到了：

```js
Object.prototype.toString
```

所以：

```js
p.toString
```

能够正常访问。

---

# 十、这就是为什么所有普通对象都有那么多方法

例如：

```js
const obj = {};
```

你可以：

```js
obj.toString();
obj.hasOwnProperty("x");
obj.valueOf();
```

明明：

```js
obj
```

里面并没有这些方法。

原因就是：

```text
obj
 ↓
Object.prototype
 ↓
null
```

而：

```js
Object.prototype
```

上面有：

```js
toString
hasOwnProperty
valueOf
isPrototypeOf
propertyIsEnumerable
```

等等。

所以所谓：

> “对象继承了 Object 的方法”

从原型机制角度来说，更准确的是：

> **属性访问沿着 `[[Prototype]]` 链查找到了 `Object.prototype` 上的属性。**

---

# 十一、一个非常重要的区别：继承 vs 属性查找

例如：

```js
function Person() {}

Person.prototype.sayHello = function () {
    console.log("hello");
};

const p = new Person();
```

执行：

```js
p.sayHello();
```

并不是说：

```text
sayHello 被复制到了 p
```

实际上：

```js
p.hasOwnProperty("sayHello")
```

结果：

```text
false
```

但是：

```js
"sayHello" in p
```

结果：

```text
true
```

因为：

```text
p
 ↓
Person.prototype
```

存在 `sayHello`。

所以：

| 操作                        | 是否搜索原型链 |
| ------------------------- | ------- |
| `obj.x`                   | 是       |
| `obj["x"]`                | 是       |
| `"x" in obj`              | 是       |
| `obj.hasOwnProperty("x")` | 否       |
| `Object.hasOwn(obj, "x")` | 否       |

这是实际开发非常重要的区别。

---

# 十二、`hasOwnProperty()` 为什么要特别小心？

很多人写：

```js
obj.hasOwnProperty("name")
```

通常没问题。

但理论上：

```js
const obj = {
    hasOwnProperty: "hello"
};
```

此时：

```js
obj.hasOwnProperty("name");
```

直接炸掉，因为：

```js
obj.hasOwnProperty
```

已经不是函数。

现代 JS 更推荐：

```js
Object.hasOwn(obj, "name");
```

例如：

```js
Object.hasOwn(obj, "name");
```

这直接判断：

> `name` 是否是 `obj` 自己的属性。

---

# 十三、原型链不仅用于“方法继承”

它本质上是：

> **对象之间的属性委托关系。**

例如：

```js
const animal = {
    eat() {
        console.log("eating");
    }
};

const dog = Object.create(animal);

dog.bark = function () {
    console.log("bark");
};
```

结构：

```text
dog
├── bark
│
└── [[Prototype]]
       ↓
     animal
       └── eat
```

调用：

```js
dog.eat();
```

实际上：

```text
dog
 ↓
animal
 ↓
eat
```

这是一种：

> **delegation（委托）**

而不是传统意义上的：

> “复制父类成员”。

这也是 JavaScript 原型模型和典型 Java/C++ 类模型的重要思想差异。

---

# 十四、`Object.create()` 是理解原型链的神器

例如：

```js
const animal = {
    eat() {
        console.log("eat");
    }
};

const dog = Object.create(animal);
```

此时：

```js
Object.getPrototypeOf(dog) === animal
```

成立。

而：

```js
dog.eat();
```

能够调用。

但：

```js
Object.hasOwn(dog, "eat")
```

是：

```text
false
```

所以：

```js
Object.create(proto)
```

可以理解为：

> **创建一个新对象，并指定它的 `[[Prototype]]`。**

这比通过 `new` 理解原型链更加直接。

---

# 十五、`Object.create(null)` 是一个非常特殊的对象

看：

```js
const dict = Object.create(null);
```

它的原型：

```js
Object.getPrototypeOf(dict)
```

结果：

```text
null
```

结构：

```text
dict
 ↓
null
```

所以：

```js
dict.toString
```

得到：

```text
undefined
```

因为根本没有：

```text
Object.prototype
```

这在某些纯字典场景非常有用：

```js
const dict = Object.create(null);

dict.apple = 10;
dict.banana = 20;
```

它没有：

```js
toString
constructor
hasOwnProperty
```

等继承属性。

不过现代 JS 里，如果你需要真正的键值集合，通常应该认真考虑：

```js
Map
```

而不是默认使用：

```js
Object.create(null)
```

---

# 十六、`Object.prototype` 为什么最终指向 null？

原型链需要一个终点。

普通对象：

```text
obj
 ↓
Object.prototype
 ↓
null
```

如果：

```js
Object.prototype.__proto__
```

还是一个对象：

```text
A
 ↓
B
 ↓
C
 ↓
D
 ↓
...
```

就没有天然终点。

因此：

```js
Object.getPrototypeOf(Object.prototype)
```

结果：

```text
null
```

所以可以把：

```text
null
```

理解成：

> 原型链的终点。

---

# 十七、`prototype` 和 `__proto__` 的经典关系

假设：

```js
function Person() {}
```

那么：

```js
Person.prototype
```

是一个对象。

而：

```js
const p = new Person();
```

有：

```js
Object.getPrototypeOf(p) === Person.prototype
```

所以最值得背下来的关系是：

```js
p.__proto__ === Person.prototype
```

但：

```js
Person.__proto__ === Person.prototype
```

**通常是错的。**

这是大量初学者最容易搞错的地方。

---

# 十八、为什么 `Person.__proto__` 不是 `Person.prototype`？

因为：

```js
Person
```

本身也是一个对象。

准确来说：

> 函数也是对象。

所以：

```text
Person
 ↓ [[Prototype]]
Function.prototype
 ↓ [[Prototype]]
Object.prototype
 ↓
null
```

而：

```text
Person.prototype
 ↓ [[Prototype]]
Object.prototype
 ↓
null
```

注意：

```text
Person
```

和：

```text
Person.prototype
```

完全是两个不同对象。

---

# 十九、把这几个关系一次性理清

```js
function Person() {}

const p = new Person();
```

关系：

```text
                    Function.prototype
                           ↑
                           │ [[Prototype]]
                           │
Person ────────────────────┘
  │
  │ .prototype
  ↓
Person.prototype
  ↑
  │ [[Prototype]]
  │
  p
```

同时：

```js
Object.getPrototypeOf(Person) === Function.prototype
```

而：

```js
Object.getPrototypeOf(p) === Person.prototype
```

这是两个完全不同方向。

---

# 二十、一个超级重要的事实：函数也是对象

例如：

```js
function foo() {}
```

`foo` 同时具备：

```text
函数能力
+
对象能力
```

所以：

```js
foo();
```

可以调用。

同时：

```js
foo.name
foo.length
foo.prototype
foo.__proto__
```

也可以访问。

因为：

> 函数本身也是对象，只不过它具有可调用能力。

这也是为什么：

```js
Function.prototype
Object.prototype
```

这些关系看起来很绕。

---

# 二十一、`Function` 自己也非常特殊

可以观察：

```js
Object.getPrototypeOf(Function) === Function.prototype
```

同时：

```js
Function.prototype.__proto__ === Object.prototype
```

所以：

```text
Function
 ↓
Function.prototype
 ↓
Object.prototype
 ↓
null
```

甚至：

```js
Object.getPrototypeOf(Object) === Function.prototype
```

因为：

> `Object` 本身也是一个函数对象。

所以：

```text
Object
 ↓
Function.prototype
 ↓
Object.prototype
 ↓
null
```

这就是 JS 对象模型中非常漂亮但也很容易绕进去的一部分。

---

# 二十二、`constructor` 到底是什么？

看：

```js
function Person() {}

console.log(Person.prototype.constructor === Person);
```

通常：

```text
true
```

但是一定不要把：

```js
constructor
```

理解成：

> “这个对象真正的类型”。

因为：

```js
const p = new Person();

p.constructor === Person
```

通常成立。

但这是因为：

```text
p
 ↓
Person.prototype
 ↓
constructor → Person
```

属性查找找到的。

`constructor` 本身只是一个普通属性。

例如：

```js
const obj = {};

obj.constructor
```

得到：

```js
Object
```

但这不意味着：

```text
constructor 是不可改变的真实类型标签
```

---

# 二十三、为什么修改 prototype 时经常把 constructor 搞丢？

例如：

```js
function Person() {}

Person.prototype = {
    sayHello() {
        console.log("hello");
    }
};
```

现在：

```js
const p = new Person();

p.constructor
```

可能得到：

```text
Object
```

为什么？

因为你把原来的：

```js
Person.prototype
```

整个替换掉了。

原来的对象类似：

```js
{
    constructor: Person
}
```

现在变成：

```js
{
    sayHello: ...
}
```

没有：

```js
constructor: Person
```

于是继续沿着原型链查找：

```text
p
 ↓
新的 Person.prototype
 ↓
Object.prototype
 ↓
constructor → Object
```

所以：

```js
p.constructor === Object
```

可能为：

```text
true
```

---

# 二十四、正确修复 constructor

可以：

```js
Person.prototype = {
    constructor: Person,

    sayHello() {
        console.log("hello");
    }
};
```

但现代代码通常更少需要手动这么写，因为：

```js
class
```

会帮你处理这些结构。

---

# 二十五、ES6 `class` 到底是不是另一套继承机制？

不是。

这是非常重要的一点：

> **`class` 是建立在原型机制之上的语法。**

例如：

```js
class Person {
    constructor(name) {
        this.name = name;
    }

    sayHello() {
        console.log("hello");
    }
}
```

大体可以理解成：

```js
function Person(name) {
    this.name = name;
}

Person.prototype.sayHello = function () {
    console.log("hello");
};
```

所以：

```js
const p = new Person("Alice");
```

依然存在：

```text
p
 ↓
Person.prototype
 ↓
Object.prototype
 ↓
null
```

---

# 二十六、为什么说 `class` 是语法糖，但又不能简单理解成语法糖？

说：

> `class` 就是 prototype 的语法糖

有一定道理，但并不完整。

因为 ES6 `class` 还引入/规范化了很多语义：

* class body
* strict mode
* private fields
* static fields
* static methods
* `extends`
* `super`
* derived constructor
* class fields
* 私有方法等

所以：

> **class 的底层继承模型仍然基于 prototype，但它不是简单的文本替换。**

这是更准确的说法。

---

# 二十七、`class extends` 建立了两条非常重要的原型链

看：

```js
class Animal {
    eat() {
        console.log("eat");
    }
}

class Dog extends Animal {
    bark() {
        console.log("bark");
    }
}
```

很多人只想到：

```text
Dog.prototype
 ↓
Animal.prototype
```

这是其中一条。

实际上还有一条：

```text
Dog
 ↓
Animal
```

所以有**两条链**。

---

## 第一条：实例原型链

```text
dog
 ↓
Dog.prototype
 ↓
Animal.prototype
 ↓
Object.prototype
 ↓
null
```

因此：

```js
dog.eat()
```

可以找到：

```text
Animal.prototype.eat
```

---

## 第二条：构造函数本身的原型链

```text
Dog
 ↓
Animal
 ↓
Function.prototype
 ↓
Object.prototype
 ↓
null
```

因此：

```js
Dog.__proto__ === Animal
```

成立。

这条链对于理解：

```js
static
super
```

特别重要。

---

# 二十八、为什么 `super` 能调用父类方法？

例如：

```js
class Animal {
    eat() {
        console.log("animal eat");
    }
}

class Dog extends Animal {
    eat() {
        super.eat();
        console.log("dog eat");
    }
}
```

这里的：

```js
super.eat()
```

本质上涉及：

```text
Dog.prototype
 ↓
Animal.prototype
```

也就是说：

> `super` 会沿着当前方法定义位置相关的原型关系寻找父类方法。

所以：

```js
super
```

不是简单的：

```js
this.__proto__
```

它有更严格的语言语义，尤其涉及 `[[HomeObject]]`。

---

# 二十九、`instanceof` 到底在干什么？

看：

```js
function Person() {}

const p = new Person();

p instanceof Person
```

结果：

```text
true
```

很多人解释成：

> “p 是 Person 创建出来的。”

这个作为入门解释可以，但不准确。

更本质的是：

> **检查 `Person.prototype` 是否出现在 `p` 的原型链上。**

也就是：

```text
p
 ↓
Person.prototype   ← 找到了
 ↓
Object.prototype
 ↓
null
```

因此：

```js
p instanceof Person
```

为：

```text
true
```

---

# 三十、手动模拟 `instanceof`

可以写一个简化版本：

```js
function myInstanceOf(obj, Constructor) {
    let proto = Object.getPrototypeOf(obj);
    const target = Constructor.prototype;

    while (proto !== null) {
        if (proto === target) {
            return true;
        }

        proto = Object.getPrototypeOf(proto);
    }

    return false;
}
```

例如：

```js
function Person() {}

const p = new Person();

myInstanceOf(p, Person);
```

结果：

```text
true
```

因为：

```text
Object.getPrototypeOf(p)
        ↓
Person.prototype
```

直接命中。

---

# 三十一、这解释了一个非常反直觉的现象

```js
const animal = {};

const dog = Object.create(animal);

function Dog() {}

Dog.prototype = animal;

const d = Object.create(animal);

console.log(d instanceof Dog);
```

可能：

```text
true
```

尽管：

```text
d
```

根本不是：

```js
new Dog()
```

为什么？

因为：

```text
d
 ↓
animal
```

而：

```js
Dog.prototype === animal
```

所以：

```text
Dog.prototype
```

出现在：

```text
d
```

的原型链上。

这说明：

> `instanceof` 判断的是**原型链关系**，不是“出生证明”。

---

# 三十二、因此 `instanceof` 并不是可靠的“类型判断”

例如：

```js
const obj = Object.create(Array.prototype);
```

那么：

```js
obj instanceof Array
```

可能是：

```text
true
```

但它并不是真正通过：

```js
new Array()
```

创建的数组。

甚至：

```js
const fakeArray = Object.create(Array.prototype);
```

并不具备真正 Array exotic object 的全部内部行为。

所以：

```js
instanceof Array
```

和：

```js
Array.isArray()
```

用途不同。

对于数组：

```js
Array.isArray(value)
```

通常更加可靠。

---

# 三十三、原型链可以被动态修改

这是 JS 很有意思的一点。

例如：

```js
const animal = {
    eat() {
        console.log("eat");
    }
};

const dog = {};

Object.setPrototypeOf(dog, animal);
```

现在：

```js
dog.eat();
```

可以工作。

因为：

```text
dog
 ↓
animal
```

但是：

> **动态修改对象原型通常不是推荐的性能敏感代码。**

原因涉及 JavaScript 引擎的内部优化。

---

# 三十四、为什么动态修改 prototype 可能影响性能？

现代 JS 引擎，例如 V8，会对对象做大量优化。

简单理解：

```js
const user = {
    name: "Alice",
    age: 20
};
```

引擎可以观察：

```text
这种对象通常具有相同的结构
```

从而使用类似：

* Hidden Class
* Shape
* Inline Cache
* Property Cache

这样的机制进行优化。

如果你不停：

```js
Object.setPrototypeOf(obj, ...)
```

就可能导致对象结构发生变化，使优化失效。

因此：

```js
Object.setPrototypeOf()
```

适合：

> 特殊场景。

不应该作为：

> 普通业务代码里的日常操作。

---

# 三十五、为什么 prototype 方法能带来性能和内存优势？

假设：

```js
function User(name) {
    this.name = name;

    this.sayHello = function () {
        console.log(this.name);
    };
}
```

创建：

```js
100000
```

个 User。

那么可能有：

```text
100000 个对象
+
100000 个函数对象
```

如果：

```js
User.prototype.sayHello = function () {
    console.log(this.name);
};
```

则：

```text
100000 个 User 对象
+
1 个共享函数
```

结构：

```text
user1 ─┐
user2 ─┤
user3 ─┤
...    ├──→ User.prototype.sayHello
userN ─┘
```

所以 prototype 的经典优势：

> **共享方法，避免每个实例都创建一份函数。**

不过现代 JS 引擎对函数对象、对象布局等有大量优化，所以不要把它机械理解成“只要 class 方法就一定省内存”。但原型方法的共享语义确实如此。

---

# 三十六、原型链和闭包的一个重要区别

例如：

```js
function Person(name) {
    this.getName = function () {
        return name;
    };
}
```

这里：

```js
getName
```

虽然每个实例创建自己的函数，但它拥有：

```text
closure → name
```

所以可以访问构造函数局部变量。

如果改成：

```js
Person.prototype.getName = function () {
    return this.name;
};
```

则方法共享，但不再直接闭包捕获构造函数中的：

```js
name
```

这体现了两个不同的设计思想：

```text
prototype
→ 共享行为

closure
→ 封装状态
```

实际设计时需要根据需求选择。

---

# 三十七、原型属性 vs 实例属性

这是实际开发中非常重要的设计问题。

例如：

```js
class User {
    constructor(name) {
        this.name = name;
    }

    sayHello() {
        console.log(this.name);
    }
}
```

这里：

```js
this.name
```

是：

> 实例自己的属性。

而：

```js
sayHello()
```

默认定义在：

> `User.prototype`

上。

所以：

```text
user
├── name
│
└── [[Prototype]]
       ↓
User.prototype
└── sayHello
```

这就是 class 最典型的对象布局。

---

# 三十八、如果两个实例都有同名属性怎么办？

例如：

```js
const proto = {
    name: "prototype"
};

const obj = Object.create(proto);

console.log(obj.name);
```

得到：

```text
prototype
```

因为：

```text
obj 自己没有 name
 ↓
prototype 有 name
```

现在：

```js
obj.name = "instance";
```

那么：

```js
console.log(obj.name);
```

得到：

```text
instance
```

因为：

```text
obj 自己有 name
```

原型上的同名属性被“遮蔽”了。

这叫：

> **property shadowing（属性遮蔽）**

---

# 三十九、`delete` 又会发生什么？

继续：

```js
delete obj.name;
```

再次：

```js
console.log(obj.name);
```

得到：

```text
prototype
```

因为：

```text
obj 自己的 name
```

被删除了，于是属性查找继续进入：

```text
[[Prototype]]
```

这正是原型链的动态性。

---

# 四十、写属性和读属性的行为并不完全一样

这是非常重要的高级知识。

读取：

```js
obj.x
```

会：

```text
obj
 ↓
prototype
 ↓
prototype prototype
```

一路寻找。

但是：

```js
obj.x = 10;
```

并不是简单地：

> “找到原型上的 x，然后修改它。”

通常情况下，如果没有特殊 accessor 行为，会在：

```text
obj
```

自身创建：

```js
x
```

例如：

```js
const proto = {
    x: 1
};

const obj = Object.create(proto);

obj.x = 100;
```

最终：

```text
proto.x === 1
obj.x === 100
```

所以：

```text
读属性 → 沿原型链查找

写属性 → 通常创建/修改自身属性
```

这是理解 prototype 的另一个关键。

---

# 四十一、但 Setter 会改变这个规则

例如：

```js
const proto = {
    set x(value) {
        console.log("setter:", value);
    }
};

const obj = Object.create(proto);

obj.x = 100;
```

此时可能触发：

```js
proto.x 的 setter
```

而不是简单地：

```js
obj.x = 100
```

这说明真正的属性访问机制比：

```text
“找属性”
```

更加复杂。

ECMAScript 内部涉及：

```text
[[Get]]
[[Set]]
[[GetOwnProperty]]
[[DefineOwnProperty]]
```

等内部方法。

---

# 四十二、Accessor Property 让原型链更加有意思

例如：

```js
const proto = {
    get name() {
        return "Alice";
    }
};

const obj = Object.create(proto);

console.log(obj.name);
```

虽然：

```js
obj
```

没有：

```text
name
```

但原型上的 getter 被调用。

因此：

```text
obj.name
```

并不一定意味着：

> 找到一个具体的数据值。

它也可能意味着：

> 找到一个 accessor property，然后执行 getter。

---

# 四十三、原型链和 Proxy

Proxy 可以进一步改变属性查找行为。

例如：

```js
const obj = new Proxy({}, {
    get(target, property, receiver) {
        console.log("get:", property);
        return Reflect.get(target, property, receiver);
    }
});
```

现在：

```js
obj.name
```

会触发：

```text
get trap
```

这说明现代 JS 中：

```text
obj.x
```

背后并不是一个简单的 C/C++ 指针查找。

它可能涉及：

```text
Proxy
 ↓
[[Get]]
 ↓
Own Property
 ↓
Prototype Chain
 ↓
Accessor
 ↓
Receiver
```

---

# 四十四、`Reflect.get()` 和 prototype

例如：

```js
Reflect.get(obj, "name");
```

可以明确使用语言的内部属性访问语义。

更有意思的是：

```js
Reflect.get(obj, "name", receiver);
```

第三个参数：

```text
receiver
```

对 getter / setter 中的：

```js
this
```

尤其重要。

所以高级 JS 继承代码里经常看到：

```js
Reflect.get(...)
Reflect.set(...)
```

而不是手动：

```js
obj[property]
```

---

# 四十五、原型链和 `this` 不是一回事

例如：

```js
const animal = {
    name: "animal",

    say() {
        console.log(this.name);
    }
};

const dog = Object.create(animal);

dog.name = "dog";

dog.say();
```

方法：

```js
say
```

来自：

```text
animal
```

但是调用：

```js
dog.say()
```

的时候：

```js
this === dog
```

所以输出：

```text
dog
```

这体现了一个非常重要的思想：

> **方法从哪里找到，和 `this` 是谁，是两个不同问题。**

方法：

```text
原型链找到
```

`this`：

```text
由调用方式决定
```

---

# 四十六、这也是为什么“原型方法”非常强大

例如：

```js
const animal = {
    say() {
        console.log(this.name);
    }
};

const dog1 = Object.create(animal);
dog1.name = "dog1";

const dog2 = Object.create(animal);
dog2.name = "dog2";
```

两者共享：

```js
animal.say
```

但：

```js
dog1.say();
dog2.say();
```

分别拥有不同的：

```js
this
```

所以：

```text
共享代码
+
不同状态
```

正是原型机制非常核心的价值。

---

# 四十七、原型继承 vs 经典面向对象继承

Java/C++/C# 等语言通常思考：

```text
Class
 ↓
Instance
```

JavaScript 原生模型更接近：

```text
Object
 ↓
Prototype Object
 ↓
Another Prototype Object
 ↓
...
```

所以 JS 的核心思想不是：

> “对象属于哪个类？”

而更接近：

> “这个对象的原型是谁？”

例如：

```js
const dog = Object.create(animal);
```

你甚至不需要定义：

```js
class Dog
```

就可以建立：

```text
dog
 ↓
animal
```

关系。

---

# 四十八、ES6 class 让 JS 更像传统 OOP

于是：

```js
class Animal {}

class Dog extends Animal {}
```

给开发者提供了：

```text
类
继承
构造器
super
static
private
```

等熟悉的面向对象语法。

但底层仍然是：

```text
prototype
+
[[Prototype]]
```

所以：

> JavaScript 是“基于原型的语言”，而 `class` 是建立在这一机制之上的高级抽象。

---

# 四十九、`Object.getPrototypeOf()` 是学习原型链最值得掌握的 API

不要一上来就疯狂：

```js
obj.__proto__
```

推荐：

```js
Object.getPrototypeOf(obj)
```

例如：

```js
function Person() {}

const p = new Person();

console.log(Object.getPrototypeOf(p));
console.log(Object.getPrototypeOf(Object.getPrototypeOf(p)));
```

得到：

```text
Person.prototype
Object.prototype
```

再：

```js
Object.getPrototypeOf(
    Object.getPrototypeOf(
        Object.getPrototypeOf(p)
    )
)
```

最终：

```text
null
```

---

# 五十、可以自己写一个“原型链探测器”

学习阶段非常推荐：

```js
function showPrototypeChain(obj) {
    let current = obj;

    while (current !== null) {
        console.log(current);
        current = Object.getPrototypeOf(current);
    }

    console.log("null");
}
```

例如：

```js
class Animal {}

class Dog extends Animal {}

const dog = new Dog();

showPrototypeChain(dog);
```

大致：

```text
Dog.prototype
Animal.prototype
Object.prototype
null
```

再：

```js
showPrototypeChain(Dog);
```

会看到另一条：

```text
Animal
Function.prototype
Object.prototype
null
```

这对理解 `class extends` 非常有效。

---

# 五十一、`Object.prototype` 是很多原型链的“公共祖先”

普通对象：

```text
obj
 ↓
Object.prototype
 ↓
null
```

数组：

```text
arr
 ↓
Array.prototype
 ↓
Object.prototype
 ↓
null
```

函数：

```text
fn
 ↓
Function.prototype
 ↓
Object.prototype
 ↓
null
```

日期：

```text
date
 ↓
Date.prototype
 ↓
Object.prototype
 ↓
null
```

正则：

```text
regex
 ↓
RegExp.prototype
 ↓
Object.prototype
 ↓
null
```

因此：

> `Object.prototype` 可以看成大量普通 JavaScript 对象原型链的共同上层。

---

# 五十二、数组为什么不是普通对象？

例如：

```js
const arr = [];
```

它的原型：

```js
Object.getPrototypeOf(arr) === Array.prototype
```

而：

```js
Array.prototype.__proto__ === Object.prototype
```

因此：

```text
arr
 ↓
Array.prototype
 ↓
Object.prototype
 ↓
null
```

所以：

```js
arr.map
```

来自：

```js
Array.prototype.map
```

而：

```js
arr.toString
```

最终可以从：

```js
Object.prototype
```

找到。

---

# 五十三、这解释了为什么数组既有数组方法，又有 Object 方法

例如：

```js
arr.map(...)
```

查找：

```text
arr
 ↓
Array.prototype
 ↓
map
```

而：

```js
arr.hasOwnProperty(...)
```

继续：

```text
arr
 ↓
Array.prototype
 ↓
Object.prototype
 ↓
hasOwnProperty
```

所以一个对象可以“继承”多层原型。

---

# 五十四、原型链可以形成多级继承

例如：

```js
class A {
    a() {}
}

class B extends A {
    b() {}
}

class C extends B {
    c() {}
}
```

实例：

```js
const obj = new C();
```

原型链：

```text
obj
 ↓
C.prototype
 ↓
B.prototype
 ↓
A.prototype
 ↓
Object.prototype
 ↓
null
```

调用：

```js
obj.a();
```

查找过程：

```text
obj.a
 ↓
C.prototype.a    没有
 ↓
B.prototype.a    没有
 ↓
A.prototype.a    找到
```

这就是原型链的实际工作方式。

---

# 五十五、原型链太长会不会影响性能？

理论上：

```text
obj
 ↓
P1
 ↓
P2
 ↓
P3
 ↓
P4
 ↓
...
```

属性查找可能需要不断向上。

但现代 JS 引擎不会简单地每次都：

```text
while(proto)
```

机械遍历。

它们会利用：

* Hidden Classes
* Shapes
* Inline Caches
* Monomorphic IC
* Polymorphic IC
* Megamorphic IC

等机制进行优化。

所以不能简单地说：

> “prototype 查找一定很慢。”

现代引擎已经对这种模式进行了大量优化。

---

# 五十六、真正需要警惕的是“对象形状混乱”

例如一个函数：

```js
function createUser(name) {
    const obj = {};

    obj.name = name;
    obj.age = 20;

    return obj;
}
```

这种对象结构比较稳定：

```text
name
age
```

而如果大量对象：

```js
obj.a = ...
obj.b = ...
obj.c = ...
```

然后各种对象拥有完全不同的结构，就可能让引擎优化难度增加。

所以高性能 JS 设计中经常强调：

> **保持对象结构稳定。**

这和 prototype/hidden class 优化是相关的。

---

# 五十七、不要随便修改 `Object.prototype`

例如：

```js
Object.prototype.foo = function () {
    console.log("foo");
};
```

那么：

```js
const a = {};
const b = [];
const c = function () {};

a.foo();
b.foo();
c.foo();
```

都可能找到这个属性。

因为：

```text
所有这些对象
      ↓
Object.prototype
```

这叫：

> **prototype pollution / 原型污染**

在安全领域尤其危险。

例如：

```js
Object.prototype.isAdmin = true;
```

某些代码如果错误地写：

```js
if (user.isAdmin) {
    // ...
}
```

就可能出现安全问题。

---

# 五十八、为什么原型污染是 Web 安全里的大问题？

假设：

```js
const config = {};
```

代码认为：

```js
config.isAdmin
```

只有配置自身设置了才存在。

但如果攻击者能够污染：

```js
Object.prototype.isAdmin = true;
```

那么：

```js
config.isAdmin
```

也会得到：

```text
true
```

因为：

```text
config
 ↓
Object.prototype
 ↓
isAdmin
```

因此安全代码经常需要区分：

```js
Object.hasOwn(obj, key)
```

和：

```js
key in obj
```

前者：

> 只检查自身。

后者：

> 会检查整个原型链。

---

# 五十九、`for...in` 为什么也和原型链有关？

例如：

```js
const proto = {
    inherited: 123
};

const obj = Object.create(proto);

obj.own = 456;
```

那么：

```js
for (const key in obj) {
    console.log(key);
}
```

可能遍历：

```text
own
inherited
```

因为：

> `for...in` 会涉及对象自身和原型链上的可枚举属性。

如果只想获取自身 enumerable properties：

```js
Object.keys(obj)
```

更加明确。

---

# 六十、`Object.keys()`、`for...in`、`Reflect.ownKeys()` 的区别

可以这样理解：

| API                              |    自身属性 | 原型属性 | Symbol |
| -------------------------------- | ------: | ---: | -----: |
| `Object.keys()`                  |       ✅ |    ❌ |      ❌ |
| `Object.getOwnPropertyNames()`   |       ✅ |    ❌ |      ❌ |
| `Object.getOwnPropertySymbols()` |       ✅ |    ❌ |      ✅ |
| `Reflect.ownKeys()`              |       ✅ |    ❌ |      ✅ |
| `for...in`                       | 自身 + 原型 |    ✅ |    通常❌ |

这和原型链理解直接相关。

---

# 六十一、一个很经典的面试题

```js
function Foo() {}

Foo.prototype.a = 1;

const x = new Foo();

console.log(x.a);

Foo.prototype.a = 2;

console.log(x.a);
```

结果：

```text
1
2
```

为什么？

因为：

```text
x
 ↓
Foo.prototype
```

`x` 自己没有：

```text
a
```

所以每次访问都会从：

```text
Foo.prototype
```

读取。

因此修改：

```js
Foo.prototype.a
```

之后：

```js
x.a
```

自然发生变化。

---

# 六十二、但如果直接给实例赋值呢？

```js
x.a = 100;
```

此时：

```text
x
├── a = 100
│
└── [[Prototype]]
       ↓
Foo.prototype
└── a = 2
```

于是：

```js
x.a
```

得到：

```text
100
```

因为自身属性遮蔽原型属性。

---

# 六十三、删除实例属性后又恢复了

```js
delete x.a;
```

然后：

```js
console.log(x.a);
```

重新得到：

```text
2
```

因为：

```text
x
 ↓
Foo.prototype.a
```

这就是原型链非常典型的：

```text
查找 → 遮蔽 → 删除 → 恢复
```

---

# 六十四、`Object.prototype` 为什么不应该乱加东西？

例如：

```js
Object.prototype.foo = "bar";
```

那么：

```js
const user = {
    name: "Alice"
};
```

现在：

```js
user.foo
```

居然存在。

甚至：

```js
const arr = [];
arr.foo
```

也存在。

因为：

```text
Array.prototype
 ↓
Object.prototype
 ↓
foo
```

所以：

> 修改 `Object.prototype` 等于修改整个 JavaScript 对象生态的公共祖先。

这是非常危险的全局行为。

---

# 六十五、原型链最重要的几个“不等式”

假设：

```js
function Person() {}
const p = new Person();
```

那么：

```js
p.__proto__ === Person.prototype
```

是：

```text
true
```

但：

```js
Person.__proto__ === Person.prototype
```

是：

```text
false
```

而：

```js
Person.prototype.__proto__ === Object.prototype
```

是：

```text
true
```

以及：

```js
Object.getPrototypeOf(Person) === Function.prototype
```

是：

```text
true
```

最后：

```js
Object.getPrototypeOf(Object.prototype) === null
```

是：

```text
true
```

这几条关系掌握了，原型链已经掌握一大半。

---

# 六十六、把整个 JavaScript 原型体系压缩成一张图

可以把最重要的关系理解成：

```text
                         ┌─────────────────┐
                         │     Function    │
                         └────────┬────────┘
                                  │
                                  │ [[Prototype]]
                                  ↓
                         Function.prototype
                                  │
                                  │ [[Prototype]]
                                  ↓
                         Object.prototype
                                  │
                                  ↓
                                 null


function Person() {}
        │
        │ .prototype
        ↓
Person.prototype
        │
        │ [[Prototype]]
        ↓
Object.prototype
        │
        ↓
       null


const p = new Person()
        │
        │ [[Prototype]]
        ↓
Person.prototype
        │
        ↓
Object.prototype
        │
        ↓
       null
```

再加入 `class extends`：

```text
dog
 │
 ↓
Dog.prototype
 │
 ↓
Animal.prototype
 │
 ↓
Object.prototype
 │
 ↓
null


Dog
 │
 ↓
Animal
 │
 ↓
Function.prototype
 │
 ↓
Object.prototype
 │
 ↓
null
```

这里最重要的就是：

> **实例有一条原型链；构造函数本身也有一条原型链。**

---

# 六十七、学习原型链时最容易出现的 10 个误区

## 误区 1

> `prototype` 是对象的原型。

不严谨。

应该说：

```text
对象的 [[Prototype]]
```

才是它的原型。

而：

```text
Constructor.prototype
```

是构造函数上的一个属性。

---

## 误区 2

> `__proto__` 和 `prototype` 是一回事。

完全不是。

```text
__proto__
→ 对象的 [[Prototype]] 访问方式

prototype
→ 函数对象上的一个普通属性（在构造场景中具有特殊约定）
```

---

## 误区 3

> constructor 就是真实类型。

不是。

它只是一个属性引用。

---

## 误区 4

> `instanceof` 判断对象是不是由某个 class 创建的。

不准确。

核心是：

```text
Constructor.prototype
是否存在于对象原型链
```

---

## 误区 5

> class 不使用 prototype。

错误。

class 大量依赖 prototype。

---

## 误区 6

> 原型上的方法会复制到实例。

通常不是。

实例通过：

```text
[[Prototype]]
```

访问它。

---

## 误区 7

> 修改 prototype 一定会影响所有实例。

不一定。

如果实例自身已经存在同名属性：

```js
obj.x
```

就会遮蔽 prototype 的：

```js
x
```

---

## 误区 8

> `obj.x = value` 会修改原型上的 x。

通常不是。

通常会创建/修改：

```text
obj.x
```

但 setter 等情况例外。

---

## 误区 9

> 原型链就是继承。

这是简化说法。

更准确：

> 原型链是一种属性查找/委托机制，继承是建立在这种机制上的一种对象组织方式。

---

## 误区 10

> `Object.setPrototypeOf()` 随便用没问题。

功能上可以，但性能敏感场景应谨慎。

---

# 六十八、从工程角度看，什么时候应该使用 prototype？

如果你写现代 JS：

```js
class User {
    constructor(name) {
        this.name = name;
    }

    sayHello() {
        return `Hello ${this.name}`;
    }
}
```

通常不需要手动：

```js
User.prototype.sayHello = ...
```

因为：

```js
class
```

已经提供了更清晰的语法。

但理解 prototype 仍然非常重要，因为你在：

* 调试
* 阅读第三方库
* 理解 class
* 处理继承
* 使用 Proxy
* 分析性能
* 阅读框架源码
* 处理原型污染
* 理解 `instanceof`
* 理解 `Object.create`
* 理解旧式 JS 代码

时都会遇到它。

---

# 六十九、真正高级的理解：原型链是一种“动态委托系统”

如果把 JavaScript 原型机制抽象一下：

```text
对象 A
  │
  │ 找不到属性
  ↓
对象 B
  │
  │ 找不到属性
  ↓
对象 C
  │
  │ 找不到属性
  ↓
null
```

这实际上就是：

> **属性查找失败后，把查找委托给 prototype。**

因此：

```js
dog.eat()
```

可以抽象成：

```text
dog：
“我没有 eat。”

↓ 委托

Animal.prototype：
“我有 eat。”

↓ 执行

eat.call(dog)
```

这里甚至可以把 JS 的原型继承理解成：

> **delegation-based object model（基于委托的对象模型）**

这个理解比单纯背：

```js
__proto__
prototype
constructor
```

更加接近 JavaScript 的本质。

---

# 七十、最后给你一个“原型链心智模型”

以后看到任何 JS 对象，先问四个问题：

### ① 它自己有什么属性？

```js
Object.hasOwn(obj, "x")
```

---

### ② 它的 `[[Prototype]]` 是谁？

```js
Object.getPrototypeOf(obj)
```

---

### ③ 如果自己没有这个属性，原型上有没有？

```text
obj
 ↓
prototype
 ↓
prototype.prototype
 ↓
...
```

---

### ④ 最终是不是走到了 null？

```text
null
```

这四步基本可以解释绝大多数原型链问题。

---

# 七十一、最值得掌握的“原型链四件套”

如果你准备真正掌握 JavaScript 原型，而不是停留在面试八股，我建议重点吃透这四个 API：

```js
Object.getPrototypeOf(obj)
Object.setPrototypeOf(obj, proto)
Object.create(proto)
Object.hasOwn(obj, prop)
```

以及三个关键概念：

```text
[[Prototype]]
Constructor.prototype
prototype chain
```

再加三个机制：

```text
new
instanceof
class extends
```

它们串起来之后，整个体系就通了：

```text
             class / constructor
                    │
                    │ prototype
                    ↓
              Prototype Object
                    ↑
                    │ [[Prototype]]
                    │
                 instance
                    │
                    │ 属性查找
                    ↓
              Prototype Chain
                    │
                    ↓
             Object.prototype
                    │
                    ↓
                   null
```

---

## 最后浓缩成一句话

如果让我用一句话解释 JavaScript 原型链：

> **JavaScript 对象本身没有找到某个属性时，会沿着自己的 `[[Prototype]]` 指向的对象继续查找，一直查到 `null`；而 `new`、`class extends`、`instanceof` 等大量语言特性，都是建立在这套原型关系之上的。**

真正理解到这一层，你就不会再把：

```text
__proto__
prototype
constructor
new
class
extends
super
instanceof
Object.create()
```

当成一堆互相独立的知识点，而会看到它们其实都围绕着**同一个对象模型**展开。
