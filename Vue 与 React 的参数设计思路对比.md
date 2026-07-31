# Vue 与 React 的参数设计思路对比

这两个框架在"参数"设计上的哲学差异，其实反映了它们更深层的设计理念分歧：Vue 倾向"约定 + 声明式配置"，React 倾向"一切都是函数入参"。展开聊聊。

## 一、组件 Props 设计

### Vue：Props 是一份"契约声明"

Vue 要求显式声明 props 的类型、默认值、校验逻辑，这些声明本身就是一种归一化——把"父组件可能传什么"这件事，收敛成一份 schema。

```javascript
// Vue 3 (Composition API)
const props = defineProps({
  size: {
    type: String,
    default: 'medium',
    validator: (v) => ['small', 'medium', 'large'].includes(v)
  },
  modelValue: [String, Number], // 支持多类型
})
```

这里有个很典型的归一化案例：Element Plus、Ant Design Vue 这类组件库，很多 props 都设计成"既能传字符串速写，也能传对象精确配置"：

```javascript
// 简写
<el-button size="large" />

// 完整配置（部分组件支持）
<el-table :size="{ row: 'large', cell: 'small' }" />
```

组件内部会有一层归一化逻辑，把简写形式统一展开成完整对象再使用。

### React：Props 就是普通的函数参数

React 组件本质是函数，props 就是这个函数的入参对象，没有内建的类型声明机制（除非用 TypeScript 或 PropTypes）。这意味着**归一化的责任完全落在开发者手上**，社区没有统一约定，只能靠 TS 类型 + 默认值语法：

```jsx
function Button({ size = 'medium', variant = 'primary', ...rest }) {
  // 归一化逻辑要自己写
  const normalizedSize = typeof size === 'string' ? size : size.default;
  return <button className={cx(normalizedSize, variant)} {...rest} />;
}
```

**核心差异**：Vue 把"参数校验和归一化"上升为框架层面的一等公民（`props` 选项系统），React 把这件事完全交还给开发者或社区库（如 `zod`、`yup` 做运行时校验）。

## 二、组件间"双向绑定"参数的设计

这是两者思路分野最明显的地方。

### Vue：`v-model` 是语法糖，内部靠约定归一化

```javascript
// Vue 3 v-model 展开后等价于：
<Child :modelValue="val" @update:modelValue="val = $event" />

// 组件内部
const props = defineProps(['modelValue'])
const emit = defineEmits(['update:modelValue'])
```

Vue 3 甚至支持多个 v-model（`v-model:title`、`v-model:visible`），本质是把"值 + 变更事件"这一组参数用命名约定归一化成统一模式，模板编译器在背后做展开。

### React：没有双向绑定语法糖，全靠手写受控模式

```jsx
function Input({ value, onChange }) {
  return <input value={value} onChange={e => onChange(e.target.value)} />;
}

// 使用方要自己维护状态
const [val, setVal] = useState('');
<Input value={val} onChange={setVal} />
```

React 坚持"单向数据流 + 显式回调"，不提供归一化的糖衣，理由是官方认为隐式的双向绑定会让数据流向变得难以追踪。这是**设计哲学的取舍**，不是能力欠缺。

## 三、Hooks / Composition API 参数设计的对比

### React Hooks：参数顺序 + 依赖数组

React 的 hooks 设计里，最典型的"参数归一化"痛点是 `useEffect` 的依赖数组：

```javascript
useEffect(() => {
  doSomething(a, b);
}, [a, b]); // 依赖项需要开发者手动归一化列出
```

这里没有自动归一化机制，容易漏写依赖（ESLint 插件 `exhaustive-deps` 就是为了弥补这个问题而生）。React 的哲学是"显式优于隐式"，哪怕麻烦也不做魔法。

### Vue Composables：更接近普通函数，但有响应式参数的隐藏规则

```javascript
function useMouse() {
  const x = ref(0), y = ref(0)
  // ...
  return { x, y }
}

// 传参时，如果想保持响应性，需要传 ref 或 getter，而不是解构出的普通值
function useFeature(targetRef) {
  watchEffect(() => {
    console.log(targetRef.value) // 依赖 .value 访问约定
  })
}
```

Vue 的响应式系统要求参数如果想"保活"响应性，必须遵循 `ref`/`reactive` 的传递约定，这也是一种隐式的参数归一化规则——很多新手会在这里踩坑（解构导致响应性丢失）。

## 四、事件参数设计

| | Vue | React |
|---|---|---|
| 事件命名 | `emit('update:xxx')` 约定俗成 | 完全自定义，如 `onXxx` 只是社区惯例 |
| 事件参数个数 | 可以 emit 多个参数 `emit('change', val, oldVal)` | 通常包一层 SyntheticEvent 对象 |
| 归一化方式 | 框架层做事件名到 prop 名的映射 | 开发者自己在回调里处理 event 对象取值 |

React 的合成事件（SyntheticEvent）本身就是一层归一化——把不同浏览器的原生事件对象差异统一抹平，这点两个框架其实殊途同归，只是 Vue 更多把归一化做在"组件通信协议"层，React 更多做在"浏览器兼容"层。

## 五、配置类 API 的参数设计

### Vue Router / Vuex：配置对象 + 归一化选项

```javascript
const router = createRouter({
  routes: [
    { path: '/user/:id', component: User },
    { path: '/user/:id', component: User, props: true }, // props 可以是 bool / object / function
  ]
})
```

`props` 这个字段就是典型的"多态参数归一化"设计——布尔值、对象、函数三种输入，内部统一转换成一个 `(route) => props` 的函数处理。

### React Router：更倾向组件化配置，参数即 JSX props

```jsx
<Route path="/user/:id" element={<User />} loader={userLoader} />
```

React Router v6+ 把很多配置能力（loader、action）做成组件的 props，本质还是"参数即函数入参"的思路延伸，较少见到 Vue 那种"一个字段接受三种类型"的多态设计。

## 总结一下设计哲学的核心差异

- **Vue**：偏向"框架提供归一化能力"，鼓励多态输入（字符串/对象/函数三态皆可），复杂度收敛在框架内部，开发者写起来更省心，但需要学习框架约定
- **React**：偏向"归一化是开发者的责任"，参数就是普通 JS 值，没有魔法，可预测性强，但重复劳动更多，社区靠约定俗成（如 `onXxx`、受控组件模式）弥补

如果你在做组件库设计，这个对比其实很有参考价值：想要 API 好用、允许多种简写方式，就要在入口处显式写归一化层（不管是 Vue 的 `props` 校验器，还是 React 组件函数体开头的归一化代码）。

---
