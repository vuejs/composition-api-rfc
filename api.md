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

    // TODO ensure context consistency with `this`

    The second argument provides a context object which exposes a number of properties that were previously exposed on `this` in 2.x APIs:

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

TODO

## `toRefs`

TODO

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
    function computed<T>(getter: () => T) => Ref<T>
    // writable
    function computed<T>(options: {
      get: () => T,
      set: (value: T) => void
    }) => Ref<T>
    ```

## `watch`

- **Basic Usage**

    Run a function while reactively tracking its dependencies, and re-run it whenever the dependencies have changed.

    ``` js
    const count = ref(0)

    watch(() => console.log(count.value))
    // -> logs 0

    count.value++
    // -> logs 1
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

- **Invocation Timing When Used in Component**

    TODO

- **Debugging**

    TODO

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

## Lifecycle

TODO

## `provide` & `inject`

TODO

- **Typing**

    ``` ts
    function provide<T>(key: Key<T> | string, value: T | Ref<T>): void
    function inject<T>(key: Key<T> | string): Ref<T> | undefined
    ```

    Vue provides a `Key` generic type that can be used to sync the type of the injected value between the provider and the consumer:

    ``` ts
    import { Key, provide, inject } from 'vue'

    const key: Key<string> = Symbol()

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

TODO

## `getCurrentInstance`

TODO
