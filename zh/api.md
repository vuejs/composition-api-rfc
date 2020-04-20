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

- **Invocation Timing**

  在组件实例创建时，`setup` 会在初始化 props 后立即调用。从生命周期来看，它会在 `beforeCreate` 前被调用。

- **Usage with Templates**

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

- **Usage with Render Functions / JSX**

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

- **Arguments**

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
      // a function that may get called at a later stage
      function onClick() {
        console.log(attrs.foo) // guaranteed to be the latest reference
      }
    },
  }
  ```

  出于一些原因将 `props` 作为第一个参数，而不是包含在上下文中：

  - 组件使用 `props` 的场景更多，有时候甚至只使用 `props`

  - Having `props` as a separate argument makes it easier to type it individually (see [TypeScript-only Props Typing](#typescript-only-props-typing) below) without messing up the types of other properties on the context. It also makes it possible to keep a consistent signature across `setup`, `render` and plain functional components with TSX support.

- **Usage of `this`**

  **`this` 在 `setup()` 中不可用。** 由于 `setup()` 在解析 2.x 选项前被调用，`setup()` 中的 `this` 将与 2.X 选项中的 `this` 完全不同。同时在 `setup()` 和 2.x 选项中使用 `this` 时将造成混乱。在 `setup()` 中避免这种情况的另一个原因是这对于初学者来说非常常见的陷阱：

  ```js
  setup() {
    function onClick() {
      this // not the `this` you'd expect!
    }
  }
  ```

- **Typing**

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

## Reactivity APIs

### `reactive`

传入一个对象并返回原始对象的响应代理。等同于 2.x 的 `Vue.observable()`

```js
const obj = reactive({ count: 0 })
```

响应式转换是"深度的"：它影响所有嵌套的属性。基于 ES2015 代理实现，返回的代理对象 **不等于** 原始对象。建议仅使用代理对象而避免依赖原始对象。

- **Typing**

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

如果传入 ref 一个对象，将调用 `reactive` 方法进行深度响应转换。

- **Access in Templates**

  当 ref 作为渲染上下文返回（`setup()` 返回的对象）并在模板中使用时，它会自动解构，无需在模板内添加 `.value`：

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

- **Access in Reactive Objects**

  当 ref 作为 reactive 对象的属性被访问或修改时，将自动解构 value 值，其行为类似普通属性：

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

  注意当嵌套在 reactive `Object` 中时，ref 才会解构。从 `Array` 或诸如 `Map` 的 collection 中访问 ref 时，不会自动解构：

  ```js
  const arr = reactive([ref(0)])
  // need .value here
  console.log(arr[0].value)

  const map = reactive(new Map([['foo', ref(0)]]))
  // need .value here
  console.log(map.get('foo').value)
  ```

- **Typing**

  ```ts
  interface Ref<T> {
    value: T
  }

  function ref<T>(value: T): Ref<T>
  ```

  有时我们可能需要为 ref 指定复杂类型的值。我们可以通过在调用 `ref` 来覆盖默认推导时传递一个范型参数来实现：

  ```ts
  const foo = ref<string | number>('foo') // foo's type: Ref<string | number>

  foo.value = 123 // ok!
  ```

### `computed`

使用 getter 函数并返回一个不可改变的 ref 对象。

```js
const count = ref(1)
const plusOne = computed(() => count.value + 1)

console.log(plusOne.value) // 2

plusOne.value++ // error
```

或者使用 `get` 和 `set` 函数创建一个可修改的 ref 对象。

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

- **Typing**

  ```ts
  // read-only
  function computed<T>(getter: () => T): Readonly<Ref<Readonly<T>>>

  // writable
  function computed<T>(options: {
    get: () => T
    set: (value: T) => void
  }): Ref<T>
  ```

### `readonly`

传入一个对象 （响应或普通）或 ref 并返回一个 readonly 的代理到原始对象。对于一个 readonly 的代理对象，访问任何嵌套属性都是只读。

```js
const original = reactive({ count: 0 })

const copy = readonly(original)

watchEffect(() => {
  // works for reactivity tracking
  console.log(copy.count)
})

// mutating original will trigger watchers relying on the copy
original.count++

// mutating the copy will fail and result in a warning
copy.count++ // warning!
```

### `watchEffect`

侦听响应依赖开始之后立即调用，并在依赖改变时触发侦听。

```js
const count = ref(0)

watchEffect(() => console.log(count.value))
// -> logs 0

setTimeout(() => {
  count.value++
  // -> logs 1
}, 100)
```

#### Stopping the Watcher

当 `watchEffect` 在组件的 `setup()` 函数或生命周期钩子被调用时， watcher 会被链接到该组件的生命周期，并在组件卸载时自动停止。

在一些情况下，也可以显示的调用返回的 stop 以停止侦听：

```js
const stop = watchEffect(() => {
  /* ... */
})

// later
stop()
```

#### Side Effect Invalidation

有时当无效时需要被清理时（即在完成效果之前更改 state），被侦听的函数会执行异步副作用。副作用函数接受一个 `onInvalidate` 函数用于注册一个无效回调。该回调函数会在以下情况时调用：

- 副作用函数重新执行

- 侦听函数停止后（即当组件卸载时，如果在 `setup()` 或生命周期钩子中使用 `watchEffect`）

```js
watchEffect((onInvalidate) => {
  const token = performAsyncOperation(id.value)
  onInvalidate(() => {
    // id has changed or watcher is stopped.
    // invalidate previously pending async operation
    token.cancel()
  })
})
```

我们之所以通过传递的函数注册无效回调，而不是从回调（如 React `useEffect`）返回它，是因为返回值对于异步错误处理很重要。在执行数据请求时，副作用函数通常是一个异步函数：

```js
const data = ref(null)
watchEffect(async () => {
  data.value = await fetchData(props.id)
})
```

一个异步函数隐式的返回 Promise，但是在 Promise resolves 之前，必须立即注册 cleanup 函数。此外，Vue 依靠返回的 Promise 来自动处理 Promise 链中的潜在错误。

#### Effect Flush Timing

Vue's reactivity system buffers invalidated effects and flush them asynchronously to avoid unnecessary duplicate invocation when there are many state mutations happening in the same "tick". Internally, a component's update function is also a watched effect. When a user effect is queued, it is always invoked after all component update effects:

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

- 该计数将在初始运行时同步记录。
- 更改 count 时，将在组件**更新后**调用回调。

请注意，第一次运行是在组件 mounted 之前执行的。因此，如果你希望在侦听的副作用中访问 DOM（或模板 refs），请在 mounted 钩子中进行：

```js
onMounted(() => {
  watchEffect(() => {
    // access the DOM or template refs
  })
})
```

如果侦听的副作用需要同步或在组件更新之前重新运行，我们可以传递一个拥有 `flush` 属性的对象（默认为 `'post'`）：

```js
// fire synchronously
watchEffect(
  () => {
    /* ... */
  },
  {
    flush: 'sync',
  }
)

// fire before component updates
watchEffect(
  () => {
    /* ... */
  },
  {
    flush: 'pre',
  }
)
```

#### Watcher Debugging

`onTrack` 和 `onTrigger` 选项可用于调试一个侦听的行为。

- 当一个 reactive 属性或 ref 作为依赖被追踪时，将调用 `onTrack`
- 当侦听回调因依赖项变化被触发时，将调用 `onTrigger`

这两个回调都将接收到一个包含有关所依赖项信息的 debugger 事件。建议在以下回调中编写 `debugger` 语句来检查依赖关系：

```js
watchEffect(
  () => {
    /* side effect */
  },
  {
    onTrigger(e) {
      debugger
    },
  }
)
```

`onTrack` 和 `onTrigger` 仅在开发模式下生效。

- **Typing**

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

`watch` API 完全等效于 2.x `this.$watch` （以及 `watch` 中相应的选项）。`watch` 需要侦听特定的数据源，并在回调函数中执行副作用。默认情况是惰性，也就是说仅在侦听的源更改时才执行回调。

- 对比 `watchEffect`，`watch` 允许我们：

  - 惰性的执行副作用；
  - 更具体地说明应触发观察程序重新运行的状态；
  - 访问监视状态变化前后的值。

- **Watching a Single Source**

  侦听者的数据源头可以是一个拥有返回值的 getter 函数，也可以是 ref：

  ```js
  // watching a getter
  const state = reactive({ count: 0 })
  watch(
    () => state.count,
    (count, prevCount) => {
      /* ... */
    }
  )

  // directly watching a ref
  const count = ref(0)
  watch(count, (count, prevCount) => {
    /* ... */
  })
  ```

- **Watching Multiple Sources**

  watcher 也可以使用数组同时侦听多个源：

  ```js
  watch([fooRef, barRef], ([foo, bar], [prevFoo, prevBar]) => {
    /* ... */
  })
  ```

- **Shared Behavior with `watchEffect`**

  `watch` shares behavior with `watchEffect` in terms of [manual stoppage](#stopping-the-watcher), [side effect invalidation](#side-effect-invalidation) (with `onInvalidate` passed to the callback as the 3rd argument instead), [flush timing](#effect-flush-timing) and [debugging](#watcher-debugging).

- **Typing**

  ```ts
  // wacthing single source
  function watch<T>(
    source: WatcherSource<T>,
    callback: (
      value: T,
      oldValue: T,
      onInvalidate: InvalidateCbRegistrator
    ) => void,
    options?: WatchOptions
  ): StopHandle

  // watching multiple sources
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

  // see `watchEffect` typing for shared options
  interface WatchOptions extends WatchEffectOptions {
    immediate?: boolean // default: false
    deep?: boolean
  }
  ```

## Lifecycle Hooks

可以通过导入 `onXXX` 函数来注册生命周期钩子：

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

这些生命周期钩子注册函数只能在 `setup()` 期间同步使用，因为它们依赖于内部全局状态来定位当前的活动实例（当前正在调用 `setup()` 的组件实例）。在没有当前活动实例的情况下调用它们将导致错误。

组件实例上下文也是在生命周期钩子同步执行期间设置的，因此，在卸载组件时，在生命周期钩子内部同步创建的 watchers 和 computed 属性也将自动删除。

- **Mapping between 2.x Lifecycle Options and Composition API**

  - ~~`beforeCreate`~~ -> use `setup()`
  - ~~`created`~~ -> use `setup()`
  - `beforeMount` -> `onBeforeMount`
  - `mounted` -> `onMounted`
  - `beforeUpdate` -> `onBeforeUpdate`
  - `updated` -> `onUpdated`
  - `beforeDestroy` -> `onBeforeUnmount`
  - `destroyed` -> `onUnmounted`
  - `errorCaptured` -> `onErrorCaptured`

- **New hooks**

  除了和 2.x 生命周期等效项之外，Composition API 还提供了以下调试 hooks 函数：

  - `onRenderTracked`
  - `onRenderTriggered`

  两个 hooks 函数都接收一个 `DebuggerEvent`，类似于 watchers 的 `onTrack` 和 `onTrigger` 选项：

  ```js
  export default {
    onRenderTriggered(e) {
      debugger
      // inspect which dependency is causing the component to re-render
    },
  }
  ```

## Dependency Injection

`provide` 和 `inject` 功能类似 2.x 的 `provide/injext` ，允许依赖注入。两者都只能在当前活动实例的 `setup()` 中调用。

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

`inject` 接受一个可选的默认值作为第二个参数。如果未提供默认值，并且在 provide 上下文中未找到该属性，则 `inject` 返回 `undefined`。

- **Injection Reactivity**

  可以使用 ref 来保证 provided 和 injected 之间值的响应：

  ```js
  // in provider
  const themeRef = ref('dark')
  provide(ThemeSymbol, themeRef)

  // in consumer
  const theme = inject(ThemeSymbol, ref('light'))
  watchEffect(() => {
    console.log(`theme set to: ${theme.value}`)
  })
  ```

  如果注入一个响应对象，则它也可以被响应观察到。

- **Typing**

  ```ts
  interface InjectionKey<T> extends Symbol {}

  function provide<T>(key: InjectionKey<T> | string, value: T): void

  // without default value
  function inject<T>(key: InjectionKey<T> | string): T | undefined
  // with default value
  function inject<T>(key: InjectionKey<T> | string, defaultValue: T): T
  ```

  Vue 提供了一个继承 `Symbol` 的 `InjectionKey` 接口。它可用于在 provider 和 consumer 之间同步注入值的类型：

  ```ts
  import { InjectionKey, provide, inject } from 'vue'

  const key: InjectionKey<string> = Symbol()

  provide(key, 'foo') // providing non-string value will result in error

  const foo = inject(key) // type of foo: string | undefined
  ```

  如果使用字符串作为键或 non-typed symbols，则需要显式声明注入值的类型：

  ```ts
  const foo = inject<string>('foo') // string | undefined
  ```

## Template Refs

当使用 Composition API 时，_reactive refs_ 和 _template refs_ 的概念是统一的。为了获得对模板内元素或组件实例的引用，我们可以像往常一样在 `setup()` 中声明一个 ref 并返回它：

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
        // the DOM element will be assigned to the ref after initial render
        console.log(root.value) // <div/>
      })

      return {
        root,
      }
    },
  }
</script>
```

这里我们将 `root` 暴露在渲染上下文中，并通过 `ref ="root"` 绑定到 div 作为其 ref。 在 Virtual DOM patching 算法中，如果一个 VNode 的 `ref` 键对应于一个渲染上下文中的 ref，则该 VNode 的对应元素或组件实例将被分配给对应 ref 的值。 这是在 Virtual DOM 的 mount / patch 过程中执行的，因此模板 refs 仅在初始渲染后才能被访问。

Refs 被用作模板引用时的行为和其他 refs 一样：它们都是响应式的，并可以传递进 composition 函数（或返回）。

- **Usage with Render Function / JSX**

  ```js
  export default {
    setup() {
      const root = ref(null)

      return () =>
        h('div', {
          ref: root,
        })

      // with JSX
      return () => <div ref={root} />
    },
  }
  ```

- **Usage inside `v-for`**

  当在 `v-for` 中使用时，Composition API 模板 refs 没有特殊处理。而是使用 refs 函数（3.0 中的新功能）执行自定义的处理：

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

## Reactivity Utilities

### `unref`

如果参数是一个 ref 则返回它的 value，否则返回参数本身。它是 `val = isRef(val) ? val.value : val` 的语法糖。

```js
function useFoo(x: number | Ref<number>) {
  const unwrapped = unref(x) // unwrapped is guaranteed to be number now
}
```

### `toRef`

`toRef` 可以用来为一个 reactive 对象的属性创建一个 ref。这个 ref 可以被传递并且保留与其源引用的响应。

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

当您要将一个 prop 的 ref 传递给 composition 函数时，`toRef` 很有用：

```js
export default {
  setup(props) {
    useSomeFeature(toRef(props, 'foo'))
  },
}
```

### `toRefs`

将一个 reactive 对象转换成一个普通对象，其中返回结果中的每个属性都是一个指向原始对象对应属性的 ref。

```js
const state = reactive({
  foo: 1,
  bar: 2,
})

const stateAsRefs = toRefs(state)
/*
Type of stateAsRefs:

{
  foo: Ref<number>,
  bar: Ref<number>
}
*/

// The ref and the original property is "linked"
state.foo++
console.log(stateAsRefs.foo) // 2

stateAsRefs.foo.value++
console.log(state.foo) // 3
```

`toRefs` is useful when returning a reactive object from a composition function so that the consuming component can destructure / spread the returned object without losing reactivity:

```js
function useFeatureX() {
  const state = reactive({
    foo: 1,
    bar: 2,
  })

  // logic operating on state

  // convert to refs when returning
  return toRefs(state)
}

export default {
  setup() {
    // can destructure without losing reactivity
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

检查一个 proxy 对象是由 `reactive` 还是 `readonly` 创建的。

### `isReactive`

检查一个对象是否是由 `reactive` 创建的 reactive proxy。

如果 proxy 是由 `readonly` 创建的，但是被包装成由 `reactive` 创建的另一个 proxy，则返回 true。

### `isReadonly`

检查一个对象是否是由 `readonly` 创建的只读代理。

## Advanced Reactivity APIs

### `customRef`

Create a customized ref with explicit control over its dependency tracking and update triggering. It expects a factory function. The factory function receives `track` and `trigger` functions as arguments and should return an object with `get` and `set`.

- Example using a custom ref to implement debounce with `v-model`:

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

- **Typing**

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

Mark an object so that it will never be converted to a proxy. Returns the object itself.

```js
const foo = markRaw({})
console.log(isReactive(reactive(foo))) // false

// also works when nested inside other reactive objects
const bar = reactive({ foo })
console.log(isReactive(bar.foo)) // false
```

::: warning
`markRaw` and the shallowXXX APIs below allow you to selectively opt-out of the default deep reactive / readonly conversion and embed raw, non-proxied objects in your state graph. They can be used for various reasons:

- Some values simply should not be made reactive, for example a complex 3rd party class instance, or a Vue component object.

- Skipping proxy conversion can provide performance improvements when rendering large lists with immutable data sources.

They are considered advanced because the raw opt-out is only at the root level, so if you set a nested, non-marked raw object into a reactive object and then access it again, you get the proxied version back. This can lead to **identity hazards** - i.e. performing an operation that relies on object identity but using both the raw and the proxied version of the same object:

```js
const foo = markRaw({
  nested: {},
})

const bar = reactive({
  // although `foo` is marked as raw, foo.nested is not.
  nested: foo.nested,
})

console.log(foo.nested === bar.nested) // false
```

Identity hazards are in general rare. But to properly utilize these APIs while safely avoiding identity hazards requires a solid understanding of how the reactivity system works.
:::

### `shallowReactive`

Create a reactive proxy that tracks reactivity of its own properties, but does not perform deep reactive conversion of nested objects (exposes raw values).

```js
const state = shallowReactive({
  foo: 1,
  nested: {
    bar: 2,
  },
})

// mutating state's own properties is reactive
state.foo++
// ...but does not convert nested objects
isReactive(state.nested) // false
state.nested.bar++ // non-reactive
```

### `shallowReadonly`

Create a proxy that makes its own properties readonly, but does not perform deep readonly conversion of nested objects (exposes raw values).

```js
const state = shallowReadonly({
  foo: 1,
  nested: {
    bar: 2,
  },
})

// mutating state's own properties will fail
state.foo++
// ...but works on nested objects
isReadonly(state.nested) // false
state.nested.bar++ // works
```

### `shallowRef`

Create a ref that tracks its own `.value` mutation but doesn't make its value reactive.

```js
const foo = shallowRef({})
// mutating the ref's value is reactive
foo.value = {}
// but the value will not be converted.
isReactive(foo.value) // false
```

### `toRaw`

Return the raw, original object of a `reactive` or `readonly` proxy. This is an escape hatch that can be used to temporarily read without incurring proxy access / tracking overhead or write without triggering changes. It is **not** recommended to hold a persistent reference to the original object. Use with caution.

```js
const foo = {}
const reactiveFoo = reactive(foo)

console.log(toRaw(reactiveFoo) === foo) // true
```
