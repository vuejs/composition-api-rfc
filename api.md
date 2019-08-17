---
sidebar: auto
sidebarDepth: 2
---

# API Reference

## `setup`

The `setup` function is a new component option. It serves as the entry point for using the Composition API inside components.

- **Invocation Timing**

    `setup` is called right after the initial props resolution when a component instance is created. Lifecycle-wise, it is called after the `beforeCreate` hook and before the `created` hook.

- **Usage with Templates**

    If `setup` returns an object, the properties on the object will be merged on to the render context for the component's template:

    ``` html
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

    ``` js
    import { h, ref, reactive } from 'vue'

    export default {
      setup() {
        const count = ref(0)
        const object = reactive({ foo: 'bar' })

        return () => h('div', [
          count.value,
          object.foo
        ])
      }
    }
    ```

- **Arguments**

    The function receives the resolved props as its first argument:

    ``` js
    export default {
      props: {
        name: String
      },
      setup(props) {
        console.log(props.name)
      }
    }
    ```

    Note this `props` object is reactive - i.e. it is updated when new props are passed in, and can be observed and reacted upon using the `watch` function:

    ``` js
    export default {
      props: {
        name: String
      },
      setup(props) {
        watch(() => {
          console.log(`name is: ` + props.name)
        })
      }
    }
    ```

    The `props` object is immutable for userland code during development (will emit warning if user code attempts to mutate it).

    The second argument provides a context object which exposes a selective list of properties that were previously exposed on `this` in 2.x APIs:

    ``` js
    const MyComponent = {
      setup(props, context) {
        context.attrs
        context.slots
        context.parent
        context.root
        context.emit
      }
    }
    ```

    `attrs` and `slots` are proxies to the corresponding values on the internal component instance. This ensures they always expose the latest values even after updates so that we can destructure them without worrying accessing a stale reference:

    ``` js
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

    ``` js
    setup() {
      function onClick() {
        this // not the `this` you'd expect!
      }
    }
    ```

- **Typing**

    ``` ts
    interface Data {
      [key: string]: unknown
    }

    interface SetupContext {
      attrs: Data
      slots: Slots
      parent: ComponentInstance | null
      root: ComponentInstance
      emit: ((event: string, ...args: unknown[]) => void)
    }

    function setup(
      props: Data,
      context: SetupContext
    ): Data
    ```

    ::: tip
    To get type inference for the arguments passed to `setup()`, the use of [`createComponent`](#createcomponent) is needed.
    :::

## `reactive`

Takes an object and returns a reactive proxy of the original. This is equivalent to 2.x's `Vue.observable()`.

``` js
const obj = reactive({ count: 0 })
```

- **Typing**

    ``` ts
    function reactive<T extends object>(raw: T): T
    ```

## `ref`

Takes an inner value and returns a reactive and mutable ref object. The ref object has a single property `.value` that points to the inner value.

``` js
const count = ref(0)
console.log(count.value) // 0

count.value++
console.log(count.value) // 1
```

- **Access in Templates**

    When a ref is returned as a property on the render context (the object returned from `setup()`) and accessed in the template, it automatically unwraps to the inner value. There is no need to append `.value` in the template:

    ``` html
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

    ``` js
    const count = ref(0)
    const state = reactive({
      count
    })

    console.log(state.count) // 0

    state.count = 1
    console.log(count.value) // 1
    ```

    Note that if a new ref is assigned to a property linked to an existing ref, it will replace the old ref:

    ``` js
    const otherCount = ref(2)

    state.count = otherCount
    console.log(state.count) // 2
    console.log(count.value) // 1
    ```

- **Typing**

    ``` ts
    interface Ref<T> {
      value: T
    }

    function ref<T>(value: T): Ref<T>
    ```

    Sometimes we may need to specify complex types for a ref's inner value. We can do that succinctly by passing a generics argument when calling `ref` to override the default inference:

    ``` ts
    const foo = ref<string | number>('foo') // foo's type: Ref<string | number>

    foo.value = 123 // ok!
    ```

## `isRef`

Check whether a value is a ref object. Useful when unwrapping something that may be a ref.

``` js
const unwrapped = isRef(foo) ? foo.value : foo
```

- **Typing**

    ``` ts
    function isRef(value: any): boolean
    ```

## `toRefs`

Convert a reactive object to a plain object, where each property on the resulting object is a ref pointing to the corresponding property in the original object.

``` js
const state = reactive({
  foo: 1,
  bar: 2
})

const stateAsRefs = toRefs(state)
/*
Type of stateAsRefs:

{
  foo: Ref<1>,
  bar: Ref<2>
}
*/

// The ref and the original property is "linked"
state.foo++
console.log(stateAsRefs.foo) // 2

stateAsRefs.foo.value++
console.log(state.foo) // 3
```

`toRefs` is useful when returning a reactive object from a composition function so that the consuming component can destructure / spread the returned object without losing reactivity:

``` js
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

## `computed`

Takes a getter function and returns an immutable reactive ref object for the returned value from the getter.

``` js
const count = ref(1)
const plusOne = computed(() => count.value + 1)

console.log(plusOne.value) // 2

plusOne.value++ // error
```

Alternatively, it can take an object with `get` and `set` functions to create a writable ref object.

``` js
const count = ref(1)
const plusOne = computed({
  get: () => count.value + 1,
  set: val => { count.value = val - 1 }
})

plusOne.value = 1
console.log(count.value) // 0
```

- **Typing**

    ``` ts
    // read-only
    function computed<T>(getter: () => T): Ref<T>

    // writable
    function computed<T>(options: {
      get: () => T,
      set: (value: T) => void
    }): Ref<T>
    ```

## `watch`

- **Basic Usage**

    Run a function on the next tick (see [Callback Flush Timing](#callback-flush-timing)) while reactively tracking its dependencies, and re-run it whenever the dependencies have changed.

    ``` js
    const count = ref(0)

    watch(() => console.log(count.value))
    // -> logs 0

    setTimeout(() => {
      count.value++
      // -> logs 1
    }, 100)
    ```

- **Watching a Source**

    In some cases, we may also want to:

    - Be more specific about what state should trigger the watcher to re-run;
    - Keep a copy of the previous value of the watched state.

    In these cases we can use the alternative signature of `watch`:

    ``` js
    // watching a getter
    const state = reactive({ count: 0 })
    watch(() => state.count, (count, prevCount) => { /* ... */ })

    // directly watching a ref
    const count = ref(0)
    watch(count, (count, prevCount) => { /* ... */ })
    ```

- **Callback Flush Timing**

    Vue's reactivity system buffers watcher callbacks and flush them asynchronously to avoid unnecessary duplicate invocation when there are many state mutations happening in the same "tick". Internally, a component's update function is also a watcher callback. When a user watcher callback is queued, is is always invoked after all component render functions:

    ``` html
    <template>
      <div>{{ count }}</div>
    </template>

    <script>
    export default {
      setup() {
        const count = ref(0)

        watch(() => {
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

    - The count will not be logged synchronously.
    - The callback will first be called after the component has mounted.
    - When `count` is mutated, the callback will be called after the component has updated.

    **Rule of thumb: when a watcher callback is invoked, the component state and DOM state are already in sync.**

    In cases where a watcher callbacks needs to be invoked synchronously or before component updates, we can pass an additional options object with the `flush` option (default is `'post'`):

    ``` js
    // fire synchronously
    watch(() => { /* ... */ }, {
      flush: 'sync'
    })

    // fire before component updates
    watch(() => { /* ... */ }, {
      flush: 'pre'
    })
    ```

- **Lazy Invocation**

    In 2.x, the default behavior of `this.$watch` and the `watch` option is lazy: it will execute the getter eagerly, but only fires the callback after the first change. This has led to the need of duplicated logic in both a watcher callback and a lifecycle hook (e.g. `mounted`). The `watch` API proposed here avoids such duplication by invoking the callback eagerly. If you wish to use the 2.x behavior, you can use the `lazy` option:

    ``` js
    watch(
      () => state.foo,
      foo => console.log('foo is ' + foo),
      { lazy: true }
    )
    ```

    Note the `lazy` option only works when using the getter + callback format, since it does not make sense for the single callback usage.

- **Debugging**

    The `onTrack` and `onTrigger` options can be used to debug a watcher's behavior.

    - `onTrack` will be called when a reactive property or ref is tracked as a dependency
    - `onTrigger` will be called when the watcher callback is triggered by the mutation of a dependency

    Both callbacks will receive a debugger event which contains information on the dependency in question. It is recommended to place a `debugger` statement in these callbacks to interactively inspect the dependency:

    ``` js
    watch(() => { /* ... */ }, {
      onTrigger(e) {
        debugger
      }
    })
    ```

    `onTrack` and `onTrigger` only works in development mode.

- **Typing**

    ``` ts
    type StopHandle = () => void

    interface DebuggerEvent {
      effect: ReactiveEffect
      target: any
      type: OperationTypes
      key: string | symbol | undefined
    }

    interface WatchOptions {
      lazy?: boolean
      flush?: 'pre' | 'post' | 'sync'
      deep?: boolean
      onTrack?: (event: DebuggerEvent) => void
      onTrigger?: (event: DebuggerEvent) => void
    }

    function watch(
      effect: () => void,
      options?: WatchOptions
    ): StopHandle

    function watch<T>(
      source: Ref<T> | () => T,
      effect: (value: T) => void,
      options?: WatchOptions
    ): StopHandle
    ```

## Lifecycle Hooks

Lifecycle hooks can be registered with directly imported `onXXX` functions:

``` js
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

    ``` js
    export default {
      onRenderTriggered(e) {
        debugger
        // inspect which dependency is causing the component to re-render
      }
    }
    ```

## `provide` & `inject`

`provide` and `inject` enables dependency injection similar to the 2.x `provide/inject` options. Both can only be called during `setup()` with a current active instance.

``` js
import { ref, provide, inject } from 'vue'

const CountSymbol = Symbol()

const Ancestor = {
  setup() {
    // providing a ref can make it reactive
    const count = ref(0)
    provide(CountSymbol, count)
  }
}

const Descendent = {
  setup() {
    const count = inject(CountSymbol, 1 /* optional default value */)
    return {
      count
    }
  }
}
```

- **Details**

    - If a dependency is found, `inject` always returns a ref regardless of whether the provided value is a ref or not.

      - If a ref is provided, the ref will be returned directly by `inject`. **Reactivity is retained in this case** (i.e. the child will update if ancestor mutates the provided value).

      - If a plain value is provided, a ref will be created with its `.value` pointing to the original value. However, reactivity is not retained and the value will remain constant.

    - `inject` accepts an optional second argument as a default value in case a provided value cannot be found. In such case:

      - If the default value is already ref, it will be returned directly.

      - Otherwise, a ref wrapping the default value will be returned.

    - If a dependency is not found and a default value is not provided, `inject` returns `undefined`.

- **Typing**

    ``` ts
    interface InjectionKey<T> extends Symbol {}

    function provide<T>(key: InjectionKey<T> | string, value: T | Ref<T>): void

    // without default value
    function inject<T>(key: InjectionKey<T> | string): Ref<T> | undefined
    // with default value
    function inject<T>(key: InjectionKey<T> | string, defaultValue: T): Ref<T>
    ```

    Vue provides a `InjectionKey` interface which is a generic type that extends `Symbol`. It can be used to sync the type of the injected value between the provider and the consumer:

    ``` ts
    import { InjectionKey, provide, inject } from 'vue'

    const key: InjectionKey<string> = Symbol()

    provide(key, 'foo') // providing non-string value will result in error

    const foo = inject(key) // type of foo: Ref<string> | undefined
    ```

    If using string keys or non-typed symbols, the type of the injected value will need to be explicitly declared:

    ``` ts
    const foo = inject<string>('foo') // Ref<string> | undefined
    ```

    You can also type cast using the `Ref` type:

    ``` js
    import { Ref } from 'vue'

    const foo = inject('foo') as Ref<string>
    ```

## Template Refs

When using the Composition API, the concept of *reactive refs* and *template refs* are unified. In order to obtain a reference to an in-template element or component instance, we can declare a ref as usual and return it from `setup()`:

``` html
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

    ``` js
    export default {
      setup() {
        const root = ref(null)

        return () => h('div', {
          ref: root
        })

        // with JSX
        return () => <div ref={root}/>
      }
    }
    ```

- **Usage inside `v-for`**

    Composition API template refs do not have special handling when used inside `v-for`. Instead, use function refs (new feature in 3.0) to perform custom handling:

    ``` html
    <div
      v-for="(item, i) in list"
      :ref="el => { divs[i] = el }">
    </div>
    ```

## `createComponent`

This function is provided solely for type inference. It is needed in order for TypeScript to know that an object should be treated as a component definition so that it can infer the types of the props passed to `setup()`. It is a no-op behavior-wise. It expects a component definition and returns the argument as-is.

``` ts
import { createComponent } from 'vue'

export default createComponent({
  props: {
    foo: String
  },
  setup(props) {
    props.foo // <- type: string
  }
})
```

When using SFCs with VSCode + Vetur, the export will be implicitly wrapped so there is no need for the users to do this themselves.
