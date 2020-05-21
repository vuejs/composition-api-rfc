---
sidebar: auto
sidebarDepth: 2
---

# API Reference

::: tip
Download the free [Cheat Sheet](https://www.vuemastery.com/vue-3-cheat-sheet/) from Vue Mastery or watch their [Vue 3 Course](https://www.vuemastery.com/courses/vue-3-essentials/why-the-composition-api/).
:::

## `setup`

The `setup` function is a new component option. It serves as the entry point for using the Composition API inside components.

- **Invocation Timing**

  `setup` is called right after the initial props resolution when a component instance is created. Lifecycle-wise, it is called before the `beforeCreate` hook.

- **Usage with Templates**

  If `setup` returns an object, the properties on the object will be merged on to the render context for the component's template:

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
          object
        }
      }
    }
  </script>
  ```

  Note that refs returned from `setup` are automatically unwrapped when accessed in the template so there's no need for `.value` in templates.

- **Usage with Render Functions / JSX**

  `setup` can also return a render function, which can directly make use of reactive state declared in the same scope:

  ```js
  import { h, ref, reactive } from 'vue'

  export default {
    setup() {
      const count = ref(0)
      const object = reactive({ foo: 'bar' })

      return () => h('div', [count.value, object.foo])
    }
  }
  ```

- **Arguments**

  The function receives the resolved props as its first argument:

  ```js
  export default {
    props: {
      name: String
    },
    setup(props) {
      console.log(props.name)
    }
  }
  ```

  Note this `props` object is reactive - i.e. it is updated when new props are passed in, and can be observed and reacted upon using `watchEffect` or `watch`:

  ```js
  export default {
    props: {
      name: String
    },
    setup(props) {
      watchEffect(() => {
        console.log(`name is: ` + props.name)
      })
    }
  }
  ```

  However, do NOT destructure the `props` object, as it will lose reactivity:

  ```js
  export default {
    props: {
      name: String
    },
    setup({ name }) {
      watchEffect(() => {
        console.log(`name is: ` + name) // Will not be reactive!
      })
    }
  }
  ```

  The `props` object is immutable for userland code during development (will emit warning if user code attempts to mutate it).

  The second argument provides a context object which exposes a selective list of properties that were previously exposed on `this` in 2.x APIs:

  ```js
  const MyComponent = {
    setup(props, context) {
      context.attrs
      context.slots
      context.emit
    }
  }
  ```

  `attrs` and `slots` are proxies to the corresponding values on the internal component instance. This ensures they always expose the latest values even after updates so that we can destructure them without worrying accessing a stale reference:

  ```js
  const MyComponent = {
    setup(props, { attrs }) {
      // a function that may get called at a later stage
      function onClick() {
        console.log(attrs.foo) // guaranteed to be the latest reference
      }
    }
  }
  ```

  There are a number of reasons for placing `props` as a separate first argument instead of including it in the context:

  - It's much more common for a component to use `props` than the other properties, and very often a component uses only `props`.

  - Having `props` as a separate argument makes it easier to type it individually (see [TypeScript-only Props Typing](#typescript-only-props-typing) below) without messing up the types of other properties on the context. It also makes it possible to keep a consistent signature across `setup`, `render` and plain functional components with TSX support.

- **Usage of `this`**

  **`this` is not available inside `setup()`.** Since `setup()` is called before 2.x options are resolved, `this` inside `setup()` (if made available) will behave quite differently from `this` in other 2.x options. Making it available will likely cause confusions when using `setup()` along other 2.x options. Another reason for avoiding `this` in `setup()` is a very common pitfall for beginners:

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
  To get type inference for the arguments passed to `setup()`, the use of [`defineComponent`](#defineComponent) is needed.
  :::

## Reactivity APIs

### `reactive`

Takes an object and returns a reactive proxy of the original. This is equivalent to 2.x's `Vue.observable()`.

```js
const obj = reactive({ count: 0 })
```

The reactive conversion is "deep": it affects all nested properties. In the ES2015 Proxy based implementation, the returned proxy is **not** equal to the original object. It is recommended to work exclusively with the reactive proxy and avoid relying on the original object.

- **Typing**

  ```ts
  function reactive<T extends object>(raw: T): T
  ```

### `ref`

Takes an inner value and returns a reactive and mutable ref object. The ref object has a single property `.value` that points to the inner value.

```js
const count = ref(0)
console.log(count.value) // 0

count.value++
console.log(count.value) // 1
```

If an object is assigned as a ref's value, the object is made deeply reactive by the `reactive` method.

- **Access in Templates**

  When a ref is returned as a property on the render context (the object returned from `setup()`) and accessed in the template, it automatically unwraps to the inner value. There is no need to append `.value` in the template:

  ```html
  <template>
    <div>{{ count }}</div>
  </template>

  <script>
    export default {
      setup() {
        return {
          count: ref(0)
        }
      }
    }
  </script>
  ```

- **Access in Reactive Objects**

  When a ref is accessed or mutated as a property of a reactive object, it automatically unwraps to the inner value so it behaves like a normal property:

  ```js
  const count = ref(0)
  const state = reactive({
    count
  })

  console.log(state.count) // 0

  state.count = 1
  console.log(count.value) // 1
  ```

  Note that if a new ref is assigned to a property linked to an existing ref, it will replace the old ref:

  ```js
  const otherCount = ref(2)

  state.count = otherCount
  console.log(state.count) // 2
  console.log(count.value) // 1
  ```

  Note that ref unwrapping only happens when nested inside a reactive `Object`. There is no unwrapping performed when the ref is accessed from an `Array` or a native collection type like `Map`:

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

  Sometimes we may need to specify complex types for a ref's inner value. We can do that succinctly by passing a generics argument when calling `ref` to override the default inference:

  ```ts
  const foo = ref<string | number>('foo') // foo's type: Ref<string | number>

  foo.value = 123 // ok!
  ```

### `computed`

Takes a getter function and returns an immutable reactive ref object for the returned value from the getter.

```js
const count = ref(1)
const plusOne = computed(() => count.value + 1)

console.log(plusOne.value) // 2

plusOne.value++ // error
```

Alternatively, it can take an object with `get` and `set` functions to create a writable ref object.

```js
const count = ref(1)
const plusOne = computed({
  get: () => count.value + 1,
  set: val => {
    count.value = val - 1
  }
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

Takes an object (reactive or plain) or a ref and returns a readonly proxy to the original. A readonly proxy is deep: any nested property accessed will be readonly as well.

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

Run a function immediately while reactively tracking its dependencies, and re-run it whenever the dependencies have changed.

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

When `watchEffect` is called during a component's `setup()` function or lifecycle hooks, the watcher is linked to the component's lifecycle, and will be automatically stopped when the component is unmounted.

In other cases, it returns a stop handle which can be called to explicitly stop the watcher:

```js
const stop = watchEffect(() => {
  /* ... */
})

// later
stop()
```

#### Side Effect Invalidation

Sometimes the watched effect function will perform async side effects that need to be cleaned up when it is invalidated (i.e state changed before the effects can be completed). The effect function receives an `onInvalidate` function that can be used to register a invalidation callback. The invalidation callback is called when:

- the effect is about to re-run
- the watcher is stopped (i.e. when the component is unmounted if `watchEffect` is used inside `setup()` or lifecycle hooks)

```js
watchEffect(onInvalidate => {
  const token = performAsyncOperation(id.value)
  onInvalidate(() => {
    // id has changed or watcher is stopped.
    // invalidate previously pending async operation
    token.cancel()
  })
})
```

We are registering the invalidation callback via a passed-in function instead of returning it from the callback (like React `useEffect`) because the return value is important for async error handling. 

It is very common for the effect function to be an async function when performing data fetching:

```js
const data = ref(null)
watchEffect(async () => {
  data.value = await fetchData(props.id)
})
```

An async function implicitly returns a Promise, but the cleanup function needs to be registered immediately before the Promise resolves. In addition, Vue relies on the returned Promise to automatically handle potential errors in the Promise chain.

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
        count
      }
    }
  }
</script>
```

In this example:

- The count will be logged synchronously on initial run.
- When `count` is mutated, the callback will be called **after** the component has updated.

Note the first run is executed before the component is mounted. So if you wish to access the DOM (or template refs) in a watched effect, do it in the mounted hook:

```js
onMounted(() => {
  watchEffect(() => {
    // access the DOM or template refs
  })
})
```

In cases where a watcher effect needs to be re-run synchronously or before component updates, we can pass an additional options object with the `flush` option (default is `'post'`):

```js
// fire synchronously
watchEffect(
  () => {
    /* ... */
  },
  {
    flush: 'sync'
  }
)

// fire before component updates
watchEffect(
  () => {
    /* ... */
  },
  {
    flush: 'pre'
  }
)
```

#### Watcher Debugging

The `onTrack` and `onTrigger` options can be used to debug a watcher's behavior.

- `onTrack` will be called when a reactive property or ref is tracked as a dependency
- `onTrigger` will be called when the watcher callback is triggered by the mutation of a dependency

Both callbacks will receive a debugger event which contains information on the dependency in question. It is recommended to place a `debugger` statement in these callbacks to interactively inspect the dependency:

```js
watchEffect(
  () => {
    /* side effect */
  },
  {
    onTrigger(e) {
      debugger
    }
  }
)
```

`onTrack` and `onTrigger` only works in development mode.

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

The `watch` API is the exact equivalent of the 2.x `this.$watch` (and the corresponding `watch` option). `watch` requires watching a specific data source, and applies side effects in a separate callback function. It also is lazy by default - i.e. the callback is only called when the watched source has changed.

- Compared to `watchEffect`, `watch` allows us to:

  - Perform the side effect lazily;
  - Be more specific about what state should trigger the watcher to re-run;
  - Access both the previous and current value of the watched state.

- **Watching a Single Source**

  A watcher data source can either be a getter function that returns a value, or directly a ref:

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

  A watcher can also watch multiple sources at the same time using an Array:

  ```js
  watch([fooRef, barRef], ([foo, bar], [prevFoo, prevBar]) => {
    /* ... */
  })
  ```

* **Shared Behavior with `watchEffect`**

  `watch` shares behavior with `watchEffect` in terms of [manual stoppage](#stopping-the-watcher), [side effect invalidation](#side-effect-invalidation) (with `onInvalidate` passed to the callback as the 3rd argument instead), [flush timing](#effect-flush-timing) and [debugging](#watcher-debugging).

* **Typing**

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

Lifecycle hooks can be registered with directly imported `onXXX` functions:

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
  }
}
```

These lifecycle hook registration functions can only be used synchronously during `setup()`, since they rely on internal global state to locate the current active instance (the component instance whose `setup()` is being called right now). Calling them without a current active instance will result in an error.

The component instance context is also set during the synchronous execution of lifecycle hooks, so watchers and computed properties created inside synchronously inside lifecycle hooks are also automatically tore down when the component unmounts.

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

  In addition to 2.x lifecycle equivalents, the Composition API also provides the following debug hooks:

  - `onRenderTracked`
  - `onRenderTriggered`

  Both hooks receive a `DebuggerEvent` similar to the `onTrack` and `onTrigger` options for watchers:

  ```js
  export default {
    onRenderTriggered(e) {
      debugger
      // inspect which dependency is causing the component to re-render
    }
  }
  ```

## Dependency Injection

`provide` and `inject` enables dependency injection similar to the 2.x `provide/inject` options. Both can only be called during `setup()` with a current active instance.

```js
import { provide, inject } from 'vue'

const ThemeSymbol = Symbol()

const Ancestor = {
  setup() {
    provide(ThemeSymbol, 'dark')
  }
}

const Descendent = {
  setup() {
    const theme = inject(ThemeSymbol, 'light' /* optional default value */)
    return {
      theme
    }
  }
}
```

`inject` accepts an optional default value as the 2nd argument. If a default value is not provided and the property is not found on the provide context, `inject` returns `undefined`.

- **Injection Reactivity**

  To retain reactivity between provided and injected values, a ref can be used:

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

  If a reactive object is injected, it can also be reactively observed.

- **Typing**

  ```ts
  interface InjectionKey<T> extends Symbol {}

  function provide<T>(key: InjectionKey<T> | string, value: T): void

  // without default value
  function inject<T>(key: InjectionKey<T> | string): T | undefined
  // with default value
  function inject<T>(key: InjectionKey<T> | string, defaultValue: T): T
  ```

  Vue provides a `InjectionKey` interface which is a generic type that extends `Symbol`. It can be used to sync the type of the injected value between the provider and the consumer:

  ```ts
  import { InjectionKey, provide, inject } from 'vue'

  const key: InjectionKey<string> = Symbol()

  provide(key, 'foo') // providing non-string value will result in error

  const foo = inject(key) // type of foo: string | undefined
  ```

  If using string keys or non-typed symbols, the type of the injected value will need to be explicitly declared:

  ```ts
  const foo = inject<string>('foo') // string | undefined
  ```

## Template Refs

When using the Composition API, the concept of _reactive refs_ and _template refs_ are unified. In order to obtain a reference to an in-template element or component instance, we can declare a ref as usual and return it from `setup()`:

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
        root
      }
    }
  }
</script>
```

Here we are exposing `root` on the render context and binding it to the div as its ref via `ref="root"`. In the Virtual DOM patching algorithm, if a VNode's `ref` key corresponds to a ref on the render context, then the VNode's corresponding element or component instance will be assigned to the value of that ref. This is performed during the Virtual DOM mount / patch process, so template refs will only get assigned values after the initial render.

Refs used as templates refs behave just like any other refs: they are reactive and can be passed into (or returned from) composition functions.

- **Usage with Render Function / JSX**

  ```js
  export default {
    setup() {
      const root = ref(null)

      return () =>
        h('div', {
          ref: root
        })

      // with JSX
      return () => <div ref={root} />
    }
  }
  ```

- **Usage inside `v-for`**

  Composition API template refs do not have special handling when used inside `v-for`. Instead, use function refs (new feature in 3.0) to perform custom handling:

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
          divs
        }
      }
    }
  </script>
  ```

## Reactivity Utilities

### `unref`

Returns the inner value if the argument is a ref, otherwise return the argument itself. This is a sugar function for `val = isRef(val) ? val.value : val`.

```js
function useFoo(x: number | Ref<number>) {
  const unwrapped = unref(x) // unwrapped is guaranteed to be number now
}
```

### `toRef`

`toRef` can be used to create a ref for a property on a source reactive object. The ref can then be passed around and retains the reactive connection to its source property.

```js
const state = reactive({
  foo: 1,
  bar: 2
})

const fooRef = toRef(state, 'foo')

fooRef.value++
console.log(state.foo) // 2

state.foo++
console.log(fooRef.value) // 3
```

`toRef` is useful when you want to pass the ref of a prop to a composition function:

```js
export default {
  setup(props) {
    useSomeFeature(toRef(props, 'foo'))
  }
}
```

### `toRefs`

Convert a reactive object to a plain object, where each property on the resulting object is a ref pointing to the corresponding property in the original object.

```js
const state = reactive({
  foo: 1,
  bar: 2
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
    bar: 2
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
      bar
    }
  }
}
```

### `isRef`

Check if a value is a ref object.

### `isProxy`

Check if an object is a proxy created by `reactive` or `readonly`.

### `isReactive`

Check if an object is a reactive proxy created by `reactive`.

It also returns `true` if the proxy is created by `readonly`, but is wrapping another proxy created by `reactive`.

### `isReadonly`

Check if an object is a readonly proxy created by `readonly`.

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
        }
      }
    })
  }

  export default {
    setup() {
      return {
        text: useDebouncedRef('hello')
      }
    }
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
  nested: {}
})

const bar = reactive({
  // although `foo` is marked as raw, foo.nested is not.
  nested: foo.nested
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
    bar: 2
  }
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
    bar: 2
  }
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
