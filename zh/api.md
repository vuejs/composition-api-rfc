---
sidebar: auto
sidebarDepth: 2
---

# API Reference

::: tip
从 Vue Mastery 免费下载 [Cheat Sheet](https://www.vuemastery.com/vue-3-cheat-sheet/)，或观看 [Vue3 免费课程](https://www.vuemastery.com/courses/vue-3-essentials/why-the-composition-api/)
:::

## `setup`

`setup` 函数是一个新的组件选项。作为在组件内使用 Composition API 的入口点。

- **何时调用？**

  在组件实例创建时，`setup` 会在初始化 props 后立即调用。从生命周期来看，它会在 `beforeCreate` 前被调用。

- **与模板一起使用时的注意事项**

  如果 `setup` 返回一个对象，则对象的属性将会被合并到组件模板的渲染上下文：

  ```html
  <template>
    <div>{{ count }} {{ object.foo }}</div>
  </template>

  <script>
    import { ref, reactive } from 'vue'

    export default {
      setup() {
        const count = ref(0)
        const object = reactive({ foo: 'bar' })

        // expose to template
        return {
          count,
          object,
        }
      },
    }
  </script>
  ```

  注意 `setup` 返回的 refs 在模板中访问时会自动解构，因此在模板中使用时不需要 `.value`。

- **配合 render 函数 / JSX 的用法**

  `setup` 也可以返回一个能使用同作用域下 reactive 状态的渲染函数：

  ```js
  import { h, ref, reactive } from 'vue'

  export default {
    setup() {
      const count = ref(0)
      const object = reactive({ foo: 'bar' })

      return () => h('div', [count.value, object.foo])
    },
  }
  ```

- **参数**

  该函数接收 props 作为其第一个参数：

  ```js
  export default {
    props: {
      name: String,
    },
    setup(props) {
      console.log(props.name)
    },
  }
  ```

  注意 `props` 对象是响应式的，`watchEffect` 或 `watch` 会观察和响应 props 的更新：

  ```js
  export default {
    props: {
      name: String,
    },
    setup(props) {
      watchEffect(() => {
        console.log(`name is: ` + props.name)
      })
    },
  }
  ```

  然而不要解构 `props` 对象，那样会使其失去响应：

  ```js
  export default {
    props: {
      name: String,
    },
    setup({ name }) {
      watchEffect(() => {
        console.log(`name is: ` + name) // Will not be reactive!
      })
    },
  }
  ```

  在开发过程中，组件内尝试修改 `props` 时将会报出警告。

  第二个参数提供了一个包含 2.X APIs `this` 属性的上下文：

  ```js
  const MyComponent = {
    setup(props, context) {
      context.attrs
      context.slots
      context.emit
    },
  }
  ```

  `attrs` 和 `slots` 都是组件实例上对应的代理对象，可以确保在更新变化后显示最新值：

  ```js
  const MyComponent = {
    setup(props, { attrs }) {
      // 一个可能之后回调用的签名
      function onClick() {
        console.log(attrs.foo) // 一定是最新的引用，没有丢失响应性
      }
    },
  }
  ```

  出于一些原因将 `props` 作为第一个参数，而不是包含在上下文中：

  - 组件使用 `props` 的场景更多，有时候甚至只使用 `props`

  - 将 `props` 独立出来作为第一个参数，是为了让 TypeScript 对 props 有更好对类型推导，不会和上下文中的其他属性相混淆。这也使得与 `setup` 、 `render` 和其他使用了 TSX 的函数式组件的参数签名保持一致。

- **Usage of `this`**

  **`this` 在 `setup()` 中不可用。** 由于 `setup()` 在解析 2.x 选项前被调用，`setup()` 中的 `this` 将与 2.X 选项中的 `this` 完全不同。同时在 `setup()` 和 2.x 选项中使用 `this` 时将造成混乱。在 `setup()` 中避免这种情况的另一个原因是：这对于初学者来说可能并没有完全弄清楚 `this` 当时所代表的含义从而导致出错：

  ```js
  setup() {
    function onClick() {
      this // 这里 `this` 不是你想要的意思！
    }
    /*
      JavaScript this 关键词指的是它所属的对象。

      它拥有不同的值，具体取决于它的使用位置：

      - 在方法中，this 指的是所有者对象。
      - 单独的情况下，this 指的是全局对象。
      - 在函数中，this 指的是全局对象。
      - 在函数中，严格模式下，this 是 undefined。
      - 在事件中，this 指的是接收事件的元素。

      像 call() 和 apply() 这样的方法可以将 this 引用到任何对象。
    */
  }
  ```

- **类型定义**

  ```ts
  interface Data {
    [key: string]: unknown
  }

  interface SetupContext {
    attrs: Data
    slots: Slots
    emit: (event: string, ...args: unknown[]) => void
  }

  function setup(props: Data, context: SetupContext): Data
  ```

  ::: tip
  为了获得传递给 `setup()` 参数的类型推断，需要使用 [`defineComponent`](#defineComponent)。
  :::

## 响应式系统相关 API

### `reactive`

传入一个对象并返回原始对象的响应代理。等同于 2.x 的 `Vue.observable()`

```js
const obj = reactive({ count: 0 })
```

响应式转换是 “深层递归”：它影响所有内部的属性。基于 ES2015 的 Proxy 代理实现，返回的代理对象 **不等于** 原始对象。我们建议 **仅使用代理对象** 而避免依赖原始对象。

- **类型定义**

  ```ts
  function reactive<T extends object>(raw: T): T
  ```

### `ref`

接受一个参数值并返回一个响应且可改变的 ref 对象。ref 对象拥有一个指向内部值的单一属性 `.value`。

```js
const count = ref(0)
console.log(count.value) // 0

count.value++
console.log(count.value) // 1
```

如果传入 ref 的是一个对象，将调用 `reactive` 方法进行深度响应转换。

- **在模板中访问**

  当 ref 作为渲染上下文返回（`setup()` 返回的对象）并在模板中使用时，它会自动解套，无需在模板内额外书写 `.value`：

  ```html
  <template>
    <div>{{ count }}</div>
  </template>

  <script>
    export default {
      setup() {
        return {
          count: ref(0),
        }
      },
    }
  </script>
  ```

- **作为响应式对象的属性访问**

  当 ref 作为 reactive 对象的属性被访问或修改时，也将自动解套 value 值，其行为类似普通属性：

  ```js
  const count = ref(0)
  const state = reactive({
    count,
  })

  console.log(state.count) // 0

  state.count = 1
  console.log(count.value) // 1
  ```

  注意如果将一个新的 ref 分配给现有的 ref， 将替换旧的 ref：

  ```js
  const otherCount = ref(2)

  state.count = otherCount
  console.log(state.count) // 2
  console.log(count.value) // 1
  ```

  注意当嵌套在响应式 `Object` 中时，ref 才会解套。从 `Array` 或诸如 `Map` 等原生集合类中访问 ref 时，不会自动解套：

  ```js
  const arr = reactive([ref(0)])
  // 这里需要写 .value
  console.log(arr[0].value)

  const map = reactive(new Map([['foo', ref(0)]]))
  // 这里需要写 .value
  console.log(map.get('foo').value)
  ```

- **类型定义**

  ```ts
  interface Ref<T> {
    value: T
  }

  function ref<T>(value: T): Ref<T>
  ```

  有时我们可能需要为 ref 做一个较为复杂的类型标注。我们可以通过在调用 `ref` 时传递泛型参数来覆盖根据所赋的值作出的默认推导：

  ```ts
  const foo = ref<string | number>('foo') // foo 的类型: Ref<string | number>

  foo.value = 123 // 能够通过！
  ```

### `computed`

传入一个句柄函数，并返回一个默认不可手动修改的 ref 对象。

```js
const count = ref(1)
const plusOne = computed(() => count.value + 1)

console.log(plusOne.value) // 2

plusOne.value++ // 错误！
```

或者使用 `get` 和 `set` 函数创建一个可手动修改的计算属性。

```js
const count = ref(1)
const plusOne = computed({
  get: () => count.value + 1,
  set: (val) => {
    count.value = val - 1
  },
})

plusOne.value = 1
console.log(count.value) // 0
```

- **类型定义**

  ```ts
  // 只读的
  function computed<T>(getter: () => T): Readonly<Ref<Readonly<T>>>

  // 可更改的
  function computed<T>(options: {
    get: () => T
    set: (value: T) => void
  }): Ref<T>
  ```

### `readonly`

传入一个对象（响应式或普通皆可）或 响应式值引用 ref 并返回一个 **只读** 的代理到原始对象。对于一个只读的代理对象，访问任何内部更深层的属性也都是只读的。

```js
const original = reactive({ count: 0 })

const copy = readonly(original)

watchEffect(() => {
  // 在这里会完成依赖追踪
  console.log(copy.count)
})

// original 上的修改同样会触动依赖于它的 copy 相关的响应
original.count++

// 无法修改 copy 并且会被警告提示
copy.count++ // warning!
```

### `watchEffect`

监听响应依赖开始之后，其句柄函数立即调用，并在其依赖改变时触发响应。

```js
const count = ref(0)

watchEffect(() => console.log(count.value))
// -> 打印出 0

setTimeout(() => {
  count.value++
  // -> 打印出 1
}, 100)
```

#### 手动停止监听

当 `watchEffect` 在组件的 `setup()` 函数或生命周期钩子被调用时， watcher 监听器会被链接到该组件的生命周期，并在组件卸载时自动停止。

在一些情况下，也可以显式的调用返回的 stop 以停止监听：

```js
const stop = watchEffect(() => {
  /* ... */
})

// later
stop()
```

#### 清除被动响应

有时被动响应的句柄函数会执行一些异步的被动响应, 这些响应需要在其失效时清除（即完成之前状态已改变了）。所以监听者的回调函数可以接收一个 `onInvalidate` 函数入参, 用来注册清理该响应的回调函数。当以下情况发生时，这个 **清理回调** 会被触发:

- 监听回调即将重新调用时

- 停止监听时 (如果在 `setup()` 或 生命周期钩子函数中使用了 `watchEffect`, 则在卸载组件时)

```js
watchEffect((onInvalidate) => {
  const token = performAsyncOperation(id.value)
  onInvalidate(() => {
    // id 改变时 或 停止监听时
    // 取消之前的异步操作
    token.cancel()
  })
})
```

我们之所以是通过传递函数去注册 “清理回调”，而不是从回调返回它（如 React `useEffect` 中的方式），是因为返回值对于异步错误处理很重要。在执行数据请求时，响应的句柄函数往往是一个异步函数：

```js
const data = ref(null)
watchEffect(async () => {
  data.value = await fetchData(props.id)
})
```

我们知道 async function 都会隐性地返回一个 Promise，这样的情况下, 我们是无法返回一个需要被立刻注册的 “清理回调” 的。除此之外，回调返回的 Promise 还会被 Vue 用于内部的异步错误处理。

#### 响应句柄刷新时间

Vue 的响应式系统会缓存被动响应的句柄函数，并异步地刷新它们, 这样可以避免同一个 "tick" 中多个状态改变导致的不必要的重复调用。在核心的具体实现中, 组件的更新函数也是一个响应的句柄函数. 当一个句柄函数进入队列时, 会在所有的组件的 render 函数之后执行:

```html
<template>
  <div>{{ count }}</div>
</template>

<script>
  export default {
    setup() {
      const count = ref(0)

      watchEffect(() => {
        console.log(count.value)
      })

      return {
        count,
      }
    },
  }
</script>
```

在这个例子中：

- `count` 会在初始运行时同步打印出来
- 更改 `count` 时，将在组件 **更新后** 执行响应句柄。

请注意，初始化运行是在组件 `mounted` 之前执行的。因此，如果你希望在编写被动响应的句柄时访问 DOM（或模板中的 ref），请在 `onMounted` 钩子中进行：

```js
onMounted(() => {
  watchEffect(() => {
    // 在这里可以访问到 DOM 或者 template refs
  })
})
```

如果被动响应需要同步或在组件更新之前重新运行，我们可以传递一个拥有 `flush` 属性的对象作为选项（默认为 `'post'`）：

```js
// 同步运行
watchEffect(
  () => {
    /* ... */
  },
  {
    flush: 'sync',
  }
)

// 组件更新前执行
watchEffect(
  () => {
    /* ... */
  },
  {
    flush: 'pre',
  }
)
```

#### 监听器调试

`onTrack` 和 `onTrigger` 选项可用于调试一个监听的行为。

- 当一个 reactive 响应式对象属性或 ref 响应式值引用作为依赖被追踪时，将调用 `onTrack`

- 当监听回调因依赖项变化被触发时，将调用 `onTrigger`

这两个回调都将接收到一个包含有关所依赖项信息的调试器事件。建议在以下回调中编写 `debugger` 语句来检查依赖关系：

```js
watchEffect(
  () => {
    /* 被动响应的内容 */
  },
  {
    onTrigger(e) {
      debugger
    },
  }
)
```

**`onTrack` 和 `onTrigger` 仅在开发模式下生效。**

- **类型定义**

  ```ts
  function watchEffect(
    effect: (onInvalidate: InvalidateCbRegistrator) => void,
    options?: WatchEffectOptions
  ): StopHandle

  interface WatchEffectOptions {
    flush?: 'pre' | 'post' | 'sync'
    onTrack?: (event: DebuggerEvent) => void
    onTrigger?: (event: DebuggerEvent) => void
  }

  interface DebuggerEvent {
    effect: ReactiveEffect
    target: any
    type: OperationTypes
    key: string | symbol | undefined
  }

  type InvalidateCbRegistrator = (invalidate: () => void) => void

  type StopHandle = () => void
  ```

### `watch`

`watch` API 完全等效于 2.x `this.$watch` （以及 `watch` 中相应的选项）。`watch` 需要监听特定的数据源，并在回调函数中执行副作用。默认情况是懒执行的，也就是说仅在监听的源更改时才执行回调。

- 对比 `watchEffect`，`watch` 允许我们：

  - 懒执行的被动响应；
  - 更具体地说明应触发观察程序重新运行的状态；
  - 访问监视状态变化前后的值。

- **监听单个数据源**

  监听者的数据源头可以是一个拥有返回值的 getter 函数，也可以是 ref：

  ```js
  // 监听一个 getter
  const state = reactive({ count: 0 })
  watch(
    () => state.count,
    (count, prevCount) => {
      /* ... */
    }
  )

  // 直接监听一个 ref
  const count = ref(0)
  watch(count, (count, prevCount) => {
    /* ... */
  })
  ```

- **监听多个数据源**

  `watcher` 也可以使用数组来同时监听多个源：

  ```js
  watch([fooRef, barRef], ([foo, bar], [prevFoo, prevBar]) => {
    /* ... */
  })
  ```

- **与 `watchEffect` 共享的行为**

  watch 和 watchEffect 在 [停止监听](#手动停止监听), [清除被动响应](#清除被动响应) (`onInvalidate` 回调入参会作为第三个参数传入)，[响应句柄刷新时间](#响应句柄刷新时间) 和 [监听器调试](#监听器调试) 等方面行为一致.

- **类型定义**

  ```ts
  // 监听单数据源
  function watch<T>(
    source: WatcherSource<T>,
    callback: (
      value: T,
      oldValue: T,
      onInvalidate: InvalidateCbRegistrator
    ) => void,
    options?: WatchOptions
  ): StopHandle

  // 监听多数据源
  function watch<T extends WatcherSource<unknown>[]>(
    sources: T
    callback: (
      values: MapSources<T>,
      oldValues: MapSources<T>,
      onInvalidate: InvalidateCbRegistrator
    ) => void,
    options? : WatchOptions
  ): StopHandle

  type WatcherSource<T> = Ref<T> | (() => T)

  type MapSources<T> = {
    [K in keyof T]: T[K] extends WatcherSource<infer V> ? V : never
  }

  // 共有的属性 请查看 `watchEffect` 的类型定义
  interface WatchOptions extends WatchEffectOptions {
    immediate?: boolean // default: false
    deep?: boolean
  }
  ```

## 生命周期钩子函数

可以通过导入 `onXXX` 一族的函数来注册生命周期钩子：

```js
import { onMounted, onUpdated, onUnmounted } from 'vue'

const MyComponent = {
  setup() {
    onMounted(() => {
      console.log('mounted!')
    })
    onUpdated(() => {
      console.log('updated!')
    })
    onUnmounted(() => {
      console.log('unmounted!')
    })
  },
}
```

这些生命周期钩子注册函数只能在 `setup()` 期间同步使用， 因为它们依赖于内部的全局状态来定位当前组件实例（正在调用 `setup()` 的组件实例）, 不在当前组件下调用这些函数会抛出一个错误。

组件实例上下文也是在生命周期钩子同步执行期间设置的，因此，在卸载组件时，在生命周期钩子内部同步创建的 watchers 和 computed 计算属性也将自动删除。

- **与 2.x 版本生命周期相对应的 Composition API**

  - ~~`beforeCreate`~~ -> use `setup()`
  - ~~`created`~~ -> use `setup()`
  - `beforeMount` -> `onBeforeMount`
  - `mounted` -> `onMounted`
  - `beforeUpdate` -> `onBeforeUpdate`
  - `updated` -> `onUpdated`
  - `beforeDestroy` -> `onBeforeUnmount`
  - `destroyed` -> `onUnmounted`
  - `errorCaptured` -> `onErrorCaptured`

- **新增的钩子函数**

  除了和 2.x 生命周期等效项之外，Composition API 还提供了以下调试钩子函数：

  - `onRenderTracked`
  - `onRenderTriggered`

  两个钩子函数都接收一个 `DebuggerEvent`，这个参数跟 `watchEffect` 第二个参数 options 中的 onTrack 和 onTrigger 一样:：

  ```js
  export default {
    onRenderTriggered(e) {
      debugger
      // 检查哪个依赖性导致组件重新渲染
    },
  }
  ```

## 依赖注入

`provide` 和 `inject` 功能类似 2.x 的 `provide/inject` ，提供依赖注入。两者都只能在当前活动组件实例的 `setup()` 中调用。

```js
import { provide, inject } from 'vue'

const ThemeSymbol = Symbol()

const Ancestor = {
  setup() {
    provide(ThemeSymbol, 'dark')
  },
}

const Descendent = {
  setup() {
    const theme = inject(ThemeSymbol, 'light' /* optional default value */)
    return {
      theme,
    }
  },
}
```

`inject` 接受一个可缺省的默认值作为第二个参数。如果未提供默认值，并且在 provide 上下文中未找到该属性，则 `inject` 返回 `undefined`。

- **Injection Reactivity**

  可以使用 ref 来保证 provided 和 injected 之间值的响应：

  ```js
  // 提供者：
  const themeRef = ref('dark')
  provide(ThemeSymbol, themeRef)

  // 使用者：
  const theme = inject(ThemeSymbol, ref('light'))
  watchEffect(() => {
    console.log(`theme set to: ${theme.value}`)
  })
  ```

  如果注入一个响应式对象，则它的状态变化也可以被监听。

- **类型定义**

  ```ts
  interface InjectionKey<T> extends Symbol {}

  function provide<T>(key: InjectionKey<T> | string, value: T): void

  // 未传，使用缺省值
  function inject<T>(key: InjectionKey<T> | string): T | undefined
  // 传入了默认值
  function inject<T>(key: InjectionKey<T> | string, defaultValue: T): T
  ```

  Vue 提供了一个继承 `Symbol` 的 `InjectionKey` 接口。它可用于在 provider 和 consumer 之间同步注入值的类型：

  ```ts
  import { InjectionKey, provide, inject } from 'vue'

  const key: InjectionKey<string> = Symbol()

  provide(key, 'foo') // 类型不是 string 则会报错

  const foo = inject(key) // foo 的类型: string | undefined
  ```

  如果使用字符串作为键或没有定义类型的符号，则需要显式声明注入值的类型：

  ```ts
  const foo = inject<string>('foo') // string | undefined
  ```

## 模板中的 Ref

当使用组合式 API 时，_reactive refs_ 和 _template refs_ 的概念已经是统一的。为了获得对模板内元素或组件实例的引用，我们可以像往常一样在 `setup()` 中声明一个 ref 并返回它：

```html
<template>
  <div ref="root"></div>
</template>

<script>
  import { ref, onMounted } from 'vue'

  export default {
    setup() {
      const root = ref(null)

      onMounted(() => {
        // 在渲染完成后, 这个 div DOM 会被赋值给 root ref 对象
        console.log(root.value) // <div/>
      })

      return {
        root,
      }
    },
  }
</script>
```

这里我们将 `root` 暴露在渲染上下文中，并通过 `ref ="root"` 绑定到 `div` 作为其 `ref`。 在 Virtual DOM 对比匹配算法中，如果一个 VNode 的 `ref` 键对应于一个渲染上下文中的 ref，则该 VNode 的对应元素或组件实例将被分配给对应 ref 的值。 这是在 Virtual DOM 的 mount / patch 过程中执行的，因此模板 refs 仅在初始渲染后才能被访问。

ref 被用作模板引用时的行为和其他 ref 一样：它们都是响应式的，并可以传递进组合逻辑函数（或返回）。

- **配合 render 函数 / JSX 的用法**

  ```js
  export default {
    setup() {
      const root = ref(null)

      return () =>
        h('div', {
          ref: root,
        })

      // 使用 JSX
      return () => <div ref={root} />
    },
  }
  ```

- **在 `v-for` 中使用**

  `template refs` 在 `v-for` 中，需要使用 **函数型的 `ref`**（3.0 提供的新功能）来自定义处理方式：

  ```html
  <template>
    <div v-for="(item, i) in list" :ref="el => { divs[i] = el }">
      {{ item }}
    </div>
  </template>

  <script>
    import { ref, reactive, onBeforeUpdate } from 'vue'

    export default {
      setup() {
        const list = reactive([1, 2, 3])
        const divs = ref([])

        // make sure to reset the refs before each update
        onBeforeUpdate(() => {
          divs.value = []
        })

        return {
          list,
          divs,
        }
      },
    }
  </script>
  ```

## 响应式系统常用方法

### `unref`

如果参数是一个 ref 则返回它的 value，否则返回参数本身。它是 `val = isRef(val) ? val.value : val` 的语法糖。

```js
function useFoo(x: number | Ref<number>) {
  const unwrapped = unref(x) // unwrapped 一定是 number 类型
}
```

### `toRef`

`toRef` 可以用来为一个 reactive 对象的属性创建一个 ref。这个 ref 可以被传递并且能够保持响应性。

```js
const state = reactive({
  foo: 1,
  bar: 2,
})

const fooRef = toRef(state, 'foo')

fooRef.value++
console.log(state.foo) // 2

state.foo++
console.log(fooRef.value) // 3
```

当您要将一个 prop 中的属性作为 ref 传给组合逻辑函数时，`toRef` 就派上了用场：

```js
export default {
  setup(props) {
    useSomeFeature(toRef(props, 'foo'))
  },
}
```

### `toRefs`

传入一个响应式对象, 返回所有属性被转换成 ref 对象的普通对象:

```js
const state = reactive({
  foo: 1,
  bar: 2,
})

const stateAsRefs = toRefs(state)
/*
stateAsRefs 的类型如下:

{
  foo: Ref<number>,
  bar: Ref<number>
}
*/

// ref 对象 与 原属性的引用是 "链接" 上的
state.foo++
console.log(stateAsRefs.foo) // 2

stateAsRefs.foo.value++
console.log(state.foo) // 3
```

当想要从一个组合逻辑函数中返回响应式对象时，用 `toRefs` 是很有效的，因此例子中假设的这个组件可以 解构 / 扩展（使用 `...` 操作符）返回的对象，并不会丢失响应性：

```js
function useFeatureX() {
  const state = reactive({
    foo: 1,
    bar: 2,
  })

  // 对 state 的逻辑操作

  // 返回时将属性都转为 ref
  return toRefs(state)
}

export default {
  setup() {
    // 可以结构，不会丢失响应性
    const { foo, bar } = useFeatureX()

    return {
      foo,
      bar,
    }
  },
}
```

### `isRef`

检查一个值是否为一个 ref 对象。

### `isProxy`

检查一个 proxy 对象是否是由 `reactive` 还是 `readonly` 方法创建的。

### `isReactive`

检查一个对象是否是由 `reactive` 创建的响应式代理对象。

如果这个代理是由 `readonly` 创建的，但是又被 `reactive` 创建的另一个代理包裹了一层，那么同样也会返回 `true`。

### `isReadonly`

检查一个对象是否是由 `readonly` 创建的只读代理。

## 更高级的响应式系统 API

### `customRef`

`customRef` 用于自定义一个 `ref`，需要显式地指出如何控制依赖追踪（`track()`）和如何触发响应（`trigger`），接受一个工厂函数，两个参数分别是用于追踪的 `track` 与用于触发响应的 `trigger`，并返回一个一个带有 `get` 和 `set` 属性的对象

- 这里给出了一个如何用来做 `v-model` 防抖的例子：

  ```html
  <input v-model="text" />
  ```

  ```js
  function useDebouncedRef(value, delay = 200) {
    let timeout
    return customRef((track, trigger) => {
      return {
        get() {
          track()
          return value
        },
        set(newValue) {
          clearTimeout(timeout)
          timeout = setTimeout(() => {
            value = newValue
            trigger()
          }, delay)
        },
      }
    })
  }

  export default {
    setup() {
      return {
        text: useDebouncedRef('hello'),
      }
    },
  }
  ```

- **类型定义**

  ```ts
  function customRef<T>(factory: CustomRefFactory<T>): Ref<T>

  type CustomRefFactory<T> = (
    track: () => void,
    trigger: () => void
  ) => {
    get: () => T
    set: (value: T) => void
  }
  ```

### `markRaw`

显式标记一个对象为 “永远不会转为响应式代理”，函数返回这个对象本身。

```js
const foo = markRaw({})
console.log(isReactive(reactive(foo))) // false

// 如果被 markRaw 标记了，即使在响应式对象中作属性，也依然不是响应式的
const bar = reactive({ foo })
console.log(isReactive(bar.foo)) // false
```

::: warning
`markRaw` 和下面的 shallowXXX 一族的 API 使得你可以只是浅层地读取一个响应式对象，即只读取目标的属性值而不关心其更深层次的响应性。需要这种「浅层读取」有以下几种原因：

- 一些值的实际上的用法非常简单，并没有必要转为响应式，比如某个复杂的第三方类库的实例，或者 Vue 组件对象

- 当渲染一个元素数量庞大，但又几乎不怎么更改的列表时，跳过 Proxy 的转换可以带来很大的性能提升。

目前这方面我们做得还不够，仍在改进中。现在这种输出的选择还仅仅停留在根级别，所以如果你将一个深层次的，没有标记为 “raw” 的对象设置为某 reactive 响应式对象的属性，在重新访问他们时，你又会得到一个 Proxy 的版本，在使用中最终会导致「**依赖混淆**」的严重问题：即 被动响应的执行 同时依赖于某个对象的原始版本和代理版本。

```js
const foo = markRaw({
  nested: {},
})

const bar = reactive({
  // 尽管 `foo` 己经被标记为 raw 了, 但 foo.nested 并没有
  nested: foo.nested,
})

console.log(foo.nested === bar.nested) // false
```

「依赖混淆」 在一般使用当中应该是非常罕见的，但是要想完全避免这样的问题，必须要对整个响应式系统对工作原理有一个相当清晰对认知。
:::

### `shallowReactive`

只为某个对象的自有（第一层）属性创建浅层的响应式代理，不会对 “属性的属性” 做深层次、递归地响应式代理，而只是保留原样。

```js
const state = shallowReactive({
  foo: 1,
  nested: {
    bar: 2,
  },
})

// 更改 state 的自有属性是响应式的
state.foo++
// ...但不会深层代理
isReactive(state.nested) // false
state.nested.bar++ // 非响应式
```

### `shallowReadonly`

只为某个对象的自有（第一层）属性创建浅层的**只读**响应式代理，同样也不会做深层次、递归地代理，深层次的属性并不是只读的。

```js
const state = shallowReadonly({
  foo: 1,
  nested: {
    bar: 2,
  },
})

// 尝试修改 state 的自有属性但是失败报错了
state.foo++
// ...但对于深层次属性来说是无作用的
isReadonly(state.nested) // false
state.nested.bar++ // 深层次属性依然可修改
```

### `shallowRef`

创建一个 ref 引用，将会追踪它的 `.value` 更改操作，但是并不会对更新后对 `.value` 做响应式代理转换（即不会执行一次 `reactive`）

[具体原理详见 reactivity/ref.ts](https://github.com/vuejs/vue-next/blob/master/packages/reactivity/src/ref.ts#L57)

```js
const foo = shallowRef({})
// 更改对操作会触发响应
foo.value = {}
// 但上面新赋的这个对象并不会变为响应式对象
isReactive(foo.value) // false
```

### `toRaw`

返回由 `reactive` 或 `readonly` 方法转换成响应式代理的原始值。算是一个软脱离的方法。

可以用于临时读取但又不想造成代理访问/跟踪的开销，或写入而不想触发响应的场景。

```js
const foo = {}
const reactiveFoo = reactive(foo)

console.log(toRaw(reactiveFoo) === foo) // true
```
