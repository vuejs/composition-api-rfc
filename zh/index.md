---
sidebar: auto
---

# 组合式 API 征求意见稿

- 开始日期: 2019-07-10
- 目标主版本: 2.x / 3.x
- 相关 Issues 链接: [#42](https://github.com/vuejs/rfcs/pull/42)
- 相关实现的 Pull Request: (leave this empty)

## 概述

在此我们将为您介绍 **组合式 API**: 一组低侵入式的、函数式的 API，使得我们能够更灵活地「**组合**」组件的逻辑。

<iframe src="https://player.vimeo.com/video/365349055" width="640" height="360" frameborder="0" allow="autoplay; fullscreen" allowfullscreen></iframe>

观看 Vue Mastery 上的 [Vue 3 基础课程](https://www.vuemastery.com/courses/vue-3-essentials/why-the-composition-api/). 点击此链接下载 [Vue 3 手册](https://www.vuemastery.com/vue-3-cheat-sheet/).

## 基本范例

```html
<template>
  <button @click="increment">
    Count is: {{ state.count }}, double is: {{ state.double }}
  </button>
</template>

<script>
  import { reactive, computed } from 'vue'

  export default {
    setup() {
      const state = reactive({
        count: 0,
        double: computed(() => state.count * 2),
      })

      function increment() {
        state.count++
      }

      return {
        state,
        increment,
      }
    },
  }
</script>
```

## 动机与目的

### 更好的逻辑复用与代码组织

我们都因 Vue 简单易学而爱不释手，它让构建中小型应用程序变得轻而易举。但是随着 Vue 的影响力日益扩大，许多用户也开始使用 Vue 构建更大型的项目。这些项目通常是由多个开发人员组成团队，在很长一段时间内不断迭代和维护的。多年来，我们目睹了其中一些项目遇到了 Vue 当前 API 所带来的编程模型的限制。这些问题可归纳为两类:

1. 随着功能的增长，复杂组件的代码变得越来越难以阅读和理解。这种情况在开发人员阅读他人编写的代码时尤为常见。根本原因是 Vue 现有的 API 迫使我们通过选项组织代码，但是有的时候通过逻辑关系组织代码更有意义。

2. 目前缺少一种简洁且低成本的机制来提取和重用多个组件之间的逻辑。 (详见 [逻辑抽象、提取与复用](#逻辑抽象、提取与复用))

RFC 中提出的 API 为组件代码的组织提供了更大的灵活性。现在我们不需要总是通过选项来组织代码，而是可以将代码组织为处理特定功能的函数。这些 API 还使得在组件之间甚至组件之外逻辑的提取和重用变得更加简单。我们会在[设计细节](#设计细节)这一节展示达成的效果。

### 更好的类型推导

另一个来自大型项目开发者的常见需求是更好的 TypeScript 支持。Vue 当前的 API 在集成 TypeScript 时遇到了不小的麻烦，其主要原因是 Vue 依靠一个简单的 `this` 上下文来暴露 property，我们现在使用 `this` 的方式是比较微妙的。（比如 `methods` 选项下的函数的 `this` 是指向组件实例的，而不是这个 `methods` 对象）。

换句话说，Vue 现有的 API 在设计之初没有照顾到类型推导，这使适配 TypeScript 变得复杂。

当前，大部分使用 TypeScript 的 Vue 开发者都在通过 `vue-class-component` 这个库将组件撰写为 TypeScript class (借助 decorator)。我们在设计 3.0 时曾有[一个已废弃的 RFC](https://github.com/vuejs/rfcs/pull/17)，希望提供一个内建的 Class API 来更好的解决类型问题。然而当讨论并迭代其具体设计时，我们注意到，想通过 Class API 来解决类型问题，就必须依赖 decorator——一个在实现细节上存在许多未知数的非常不稳定的 stage 2 提案。基于它是有极大风险的。([关于 Class API 的类型相关问题请移步这里](#class-api-的类型问题))

相比较过后，本 RFC 中提出的方案更多地利用了天然对类型友好的普通变量与函数。用该提案中的 API 撰写的代码会完美享用类型推导，并且也不用做太多额外的类型标注。

这也同样意味着你写出的 JavaScript 代码几乎就是 TypeScript 的代码。即使是非 TypeScript 开发者也会因此得到更好的 IDE 类型支持而获益。

## 设计细节

### API 介绍

为了不引入全新的概念，该提案中的 API 更像是暴露 Vue 的核心功能——比如用独立的函数来创建和监听响应式的状态等。

在这里我们会介绍一些最基本的 API，及其如何取代 2.x 的选项表述组件内逻辑。

请注意，本节主要会介绍这些 API 的基本思路，所以不会展开至其完整的细节。完整的 API 规范请移步 [API 参考](./api)章节。

#### 响应式状态与副作用

让我们从一个简单的任务开始：创建一个响应式的状态

```js
import { reactive } from 'vue'

// state 现在是一个响应式的状态
const state = reactive({
  count: 0,
})
```

`reactive` 几乎等价于 2.x 中现有的 `Vue.observable()` API，且为了避免与 RxJS 中的 observable 混淆而做了重命名。这里返回的 `state` 是一个所有 Vue 用户都应该熟悉的响应式对象。

在 Vue 中，响应式状态的基本用例就是在渲染时使用它。因为有了依赖追踪，视图会在响应式状态发生改变时自动更新。在 DOM 当中渲染内容会被视为一种“副作用”：程序会在外部修改其本身 (也就是这个 DOM) 的状态。我们可以使用 `watchEffect` API 应用基于响应式状态的副作用，并*自动*进行重应用。

```js
import { reactive, watchEffect } from 'vue'

const state = reactive({
  count: 0,
})

watchEffect(() => {
  document.body.innerHTML = `count is ${state.count}`
})
```

`watchEffect` 应该接收一个应用预期副作用 (这里即设置 `innerHTML`) 的函数。它会立即执行该函数，并将该执行过程中用到的所有响应式状态的 property 作为依赖进行追踪。

这里的 `state.count` 会在首次执行后作为依赖被追踪。当 `state.count` 未来发生变更时，里面这个函数又会被重新执行。

这正是 Vue 响应式系统的精髓所在了！当你在组件中从 `data()` 返回一个对象，内部实质上通过调用 `reactive()` 使其变为响应式。而模板会被编译为渲染函数 (可被视为一种更高效的 `innerHTML`)，因而可以使用这些响应式的 property。

> `watchEffect` 和 2.x 中的 `watch` 选项类似，但是它不需要把被依赖的数据源和副作用回调分开。组合式 API 同样提供了一个 `watch` 函数，其行为和 2.x 的选项完全一致。

继续我们上面的例子，下面我们将展示如何处理用户输入:

```js
function increment() {
  state.count++
}

document.body.addEventListener('click', increment)
```

但是在 Vue 的模板系统当中，我们不需要纠结用 `innerHTML` 还是手动挂载事件监听器。让我们将例子简化为一个假设的 `renderTemplate` 方法，以专注在响应性这方面：

```js
import { reactive, watchEffect } from 'vue'

const state = reactive({
  count: 0,
})

function increment() {
  state.count++
}

const renderContext = {
  state,
  increment,
}

watchEffect(() => {
  // 假设的方法，并不是真实的 API
  renderTemplate(
    `<button @click="increment">{{ state.count }}</button>`,
    renderContext
  )
})
```

#### 计算状态 与 Ref

有时候，我们会需要一个依赖于其他状态的状态，在 Vue 中这是通过*计算属性*来处理的。我们可以使用 `computed` API 直接创建一个计算值：

```js
import { reactive, computed } from 'vue'

const state = reactive({
  count: 0,
})

const double = computed(() => state.count * 2)
```

这个 `computed` 返回了什么? 如果猜一下 `computed` 内部是如何实现的，我们可能会想出下面这样的方案:

```js
// 简化的伪代码
function computed(getter) {
  let value
  watchEffect(() => {
    value = getter()
  })
  return value
}
```

但我们知道它不会正常工作：如果 `value` 是一个例如 `number` 的基础类型，那么当被返回时，它与这个 `computed` 内部逻辑之间的关系就丢失了！这是由于 JavaScript 中基础类型是值传递而非引用传递。

![值传递 vs 引用传递](https://www.mathwarehouse.com/programming/images/pass-by-reference-vs-pass-by-value-animation.gif)

在把值作为 property 赋值给某个对象时也会出现同样的问题。一个响应式的值一旦作为 property 被赋值或从一个函数返回，而失去了响应性之后，也就失去了用途。为了确保始终可以读取到最新的计算结果，我们需要将这个值上包裹到一个对象中再返回。

```js
// 简化的伪代码
function computed(getter) {
  const ref = {
    value: null,
  }
  watchEffect(() => {
    ref.value = getter()
  })
  return ref
}
```

另外我们同样需要劫持对这个对象 `.value` property 的读/写操作，来实现依赖收集与更新通知 (为了简化我们忽略这里的代码实现)。

现在我们可以通过引用来传递计算值，也不需要担心其响应式特性会丢失了。当然代价就是：为了获取最新的值，我们每次都需要写 `.value`。

```js
const double = computed(() => state.count * 2)

watchEffect(() => {
  console.log(double.value)
}) // -> 0

state.count++ // -> 2
```

**在这里 `double` 是一个对象，我们管它叫“ref”, 用来作为一个响应性引用保留内部的值。**

> 你可能会担心 Vue 本身已经有 "ref" 的概念了。
>
> 但只是为了在模板中获取 DOM 元素或组件实例 (“模板引用”)。可以到[这里](./api.html#template-refs)查看新的 ref 系统如何同时用于逻辑状态和模板引用。

除了计算值的 ref，我们还可以使用 `ref` API 直接创建一个可变更的普通的 ref：

```js
const count = ref(0)
console.log(count.value) // 0

count.value++
console.log(count.value) // 1
```

#### 解开 Ref

::: v-pre
我们可以将一个 ref 值暴露给渲染上下文，在渲染过程中，Vue 会直接使用其内部的值，也就是说在模板中你可以把 `{{ count.value }}` 直接写为 `{{ count }}` 。
:::

这是计数器示例的另一个版本, 使用的是 `ref` 而不是 `reactive`:

```js
import { ref, watchEffect } from 'vue'

const count = ref(0)

function increment() {
  count.value++
}

const renderContext = {
  count,
  increment,
}

watchEffect(() => {
  renderTemplate(
    `<button @click="increment">{{ count }}</button>`,
    renderContext
  )
})
```

除此之外，当一个 ref 值嵌套于响应式对象之中时，访问时会自动解开:

```js
const state = reactive({
  count: 0,
  double: computed(() => state.count * 2),
})

// 无需再使用 `state.double.value`
console.log(state.double)
```

#### 组件中的使用方式

到目前为止，我们的代码已经提供了一个可以根据用户输入进行更新的 UI ，但是代码只运行了一次，无法重用。如果我们想重用其逻辑，那么不如重构成一个函数:

```js
import { reactive, computed, watchEffect } from 'vue'

function setup() {
  const state = reactive({
    count: 0,
    double: computed(() => state.count * 2),
  })

  function increment() {
    state.count++
  }

  return {
    state,
    increment,
  }
}

const renderContext = setup()

watchEffect(() => {
  renderTemplate(
    `<button @click="increment">
      Count is: {{ state.count }}, double is: {{ state.double }}
    </button>`,
    renderContext
  )
})
```

> 请注意，上面的代码并不依赖于组件实例而存在。实际上，到目前为止介绍的所有 API 都可以在组件上下文之外使用，这使我们能够在更广泛的场景中利用 Vue 的响应性系统。

现在如果我们把调用 `setup()`、创建侦听器和渲染模板的逻辑组合在一起交给框架，我们就可以仅通过 `setup()` 函数和模板定义一个组件:

```html
<template>
  <button @click="increment">
    Count is: {{ state.count }}, double is: {{ state.double }}
  </button>
</template>

<script>
  import { reactive, computed } from 'vue'

  export default {
    setup() {
      const state = reactive({
        count: 0,
        double: computed(() => state.count * 2),
      })

      function increment() {
        state.count++
      }

      return {
        state,
        increment,
      }
    },
  }
</script>
```

这就是我们熟悉的单文件组件格式，只是逻辑的部分 (`<script>`) 格式表现稍有不同。模板语法与原先保持一致。`<style>` 在这里被忽略了，因为它与原先并无二致。

#### 生命周期钩子函数

到此我们已经覆盖了组件的纯状态层面：响应式状态、计算状态和用户输入时的状态变更。但是一个组件可能还会产生其它副作用——例如在控制台打印信息、发送 AJAX 请求或在全局 `window` 对象上设置事件监听器。这些副作用大多会发生在如下时间节点上：

- 状态变化时;
- 组件挂载完成、内容更新或者解除挂载时 (这就对应了生命周期钩子)

我们知道在状态变化时可以使用 `watchEffect` 和 `watch` API 应用副作用。而为了在生命周期钩子中产生副作用，我们可以使用形如 `onXXX` 的 API (对应现有的生命周期选项)。

```js
import { onMounted } from 'vue'

export default {
  setup() {
    onMounted(() => {
      console.log('component is mounted!')
    })
  },
}
```

这些生命周期注册方法只能用在 `setup` 钩子中。它会通过内部的全局状态自动找到调用此 `setup` 钩子的实例。有意如此设计是为了减少将逻辑提取到外部函数时的冲突。

> 关于这些 API 的更多细节可以在 [API 参考](./api)页面中找到。但是在深入挖掘设计细节之前，我们建议你先将接下来的章节看完。

### 代码组织

此时我们已经将组件的 API 复制成了一些被导入的函数，但是这么做的目的是什么？用选项来定义组件看上去比用一个混入所有东西的大函数更具组织性！

这种第一印象可以理解。但正如动机与目的章节提到的，我们相信组合式的 API 实际上能够为你的代码带来*更*好组织结构，尤其是在复杂的组件中。下面我们将解释为什么。

#### 什么是“有组织的代码”

有组织的代码的最终目标应该是让代码更可读、更容易被理解。那么怎么才叫“理解”代码呢？我们可以仅仅因为知道了它所包含的选项就算了解了一个组件么？你是否有接手过他人的庞大组件 (比如[这个](https://github.com/vuejs/vue-cli/blob/a09407dd5b9f18ace7501ddb603b95e31d6d93c0/packages/@vue/cli-ui/src/components/folder/FolderExplorer.vue#L198-L404) 并觉得看得头晕？

想想看我们该如何跟别的开发者同步理解上面链接里那样的一个大型组件。你大概会从“这个组件在处理 X、Y 与 Z”开始，而不是“这个组件有这些数据 property、这些计算属性和这些方法”。

当要去理解一个组件时，我们更加关心的是“这个组件是要干什么” (即代码背后的意图) 而不是“这个组件用到了什么选项”。基于选项的 API 撰写出来的代码自然采用了后者的表述方式，然而对前者的表述并不好。

#### 逻辑关注点 vs. 选项类型

我们不妨将组件处理的“X、Y 和 Z”定义为**逻辑关注点**。可读性的问题基本不会存在于小的、单一用途的组件中，因为整个组件都在处理同一个逻辑关注点。然而这个问题在复杂的用例中会变得突出。以 [Vue CLI UI 文件浏览器](https://github.com/vuejs/vue-cli/blob/a09407dd5b9f18ace7501ddb603b95e31d6d93c0/packages/@vue/cli-ui/src/components/folder/FolderExplorer.vue#L198-L404)为例，这个组件有非常多的逻辑关注点：

- 追踪监听当前文件夹的状态并展示其中的内容
- 处理文件夹的操作（打开、关闭、刷新...）
- 处理新建文件夹的创建
- 是否只展示收藏文件夹
- 是否只展示隐藏文件夹
- 处理当前工作目录的变化

你能通过阅读基于选项的代码直接梳理出各个逻辑关注点么？显然是十分困难的。你会发现到与各个逻辑关注点相关的代码是分散在各处的。

例如“创建新文件夹”的功能使用到了[两个数据 property](https://github.com/vuejs/vue-cli/blob/a09407dd5b9f18ace7501ddb603b95e31d6d93c0/packages/@vue/cli-ui/src/components/folder/FolderExplorer.vue#L221-L222)、[一个计算属性](https://github.com/vuejs/vue-cli/blob/a09407dd5b9f18ace7501ddb603b95e31d6d93c0/packages/@vue/cli-ui/src/components/folder/FolderExplorer.vue#L240)和[一个方法](https://github.com/vuejs/vue-cli/blob/a09407dd5b9f18ace7501ddb603b95e31d6d93c0/packages/@vue/cli-ui/src/components/folder/FolderExplorer.vue#L387)，而方法的定义在距离数据 property 约一百多行的位置。

如果我们对这些逻辑关注点进行染色，我们会注意到它们在用组件选项表示时是多么分散:

<p align="center">
  <img src="https://user-images.githubusercontent.com/499550/62783021-7ce24400-ba89-11e9-9dd3-36f4f6b1fae2.png" alt="file explorer (before)" width="131">
</p>

正是这种碎片化使得理解和维护一个复杂的组件变得非常困难。选项的强行分离为展示背后的逻辑关注点设置了障碍。此外，在处理单个逻辑关注点时，我们必须不断地在选项代码块之间“跳转”，以找到与该关注点相关的部分。

> **注意**：原始代码可能在某些地方有些改动，但是我们展示了在撰写本文时的最新提交，没有进行任何修改，目的就是为了提供了一个我们自己编写生产环境代码的真实示例。

如果我们能够将相同逻辑关注点的代码并列在一起，那就再好不过了。这正是组合式 API 所能做到的，“创建新文件夹”功能可以这样写：

```js
function useCreateFolder(openFolder) {
  // 原来的数据 property
  const showNewFolder = ref(false)
  const newFolderName = ref('')

  // 原来的计算属性
  const newFolderValid = computed(() => isValidMultiName(newFolderName.value))

  // 原来的一个方法
  async function createFolder() {
    if (!newFolderValid.value) return
    const result = await mutate({
      mutation: FOLDER_CREATE,
      variables: {
        name: newFolderName.value,
      },
    })
    openFolder(result.data.folderCreate.path)
    newFolderName.value = ''
    showNewFolder.value = false
  }

  return {
    showNewFolder,
    newFolderName,
    newFolderValid,
    createFolder,
  }
}
```

请注意，与创建新文件夹功能相关的所有逻辑现在都被合并并封装在了一个函数中。由于具有描述性的名称，该函数也是某种自文档。我们建议使用 `use` 作为函数名的开头，以表示它是一个组合函数。

此模式可用于该组件的所有其它逻辑关注点，最终成为一些良好解耦的函数:

<p align="center">
  <img src="https://user-images.githubusercontent.com/499550/62783026-810e6180-ba89-11e9-8774-e7771c8095d6.png" alt="file explorer (comparison)" width="600">
</p>

> 这个比较不包括 import 语句和 `setup()` 函数。使用复合 API 重新实现的完整组件可以在[这里](https://gist.github.com/yyx990803/8854f8f6a97631576c14b63c8acd8f2e)找到。

每个逻辑关注点的代码现在都被组合进了一个组合函数。这大大减少了在处理大型组件时不断“跳转”的需要。同时组合函数也可以在编辑器中折叠起来，使组件更容易浏览:

```js
export default {
  setup() {
    // ...
  },
}

function useCurrentFolderData(networkState) {
  // ...
}

function useFolderNavigation({ networkState, currentFolderData }) {
  // ...
}

function useFavoriteFolder(currentFolderData) {
  // ...
}

function useHiddenFolders() {
  // ...
}

function useCreateFolder(openFolder) {
  // ...
}
```

`setup()` 函数现在只是简单地作为调用所有组合函数的入口：

```js
export default {
  setup() {
    // 网络状态
    const { networkState } = useNetworkState()

    // 文件夹状态
    const { folders, currentFolderData } = useCurrentFolderData(networkState)
    const folderNavigation = useFolderNavigation({
      networkState,
      currentFolderData,
    })
    const { favoriteFolders, toggleFavorite } = useFavoriteFolders(
      currentFolderData
    )
    const { showHiddenFolders } = useHiddenFolders()
    const createFolder = useCreateFolder(folderNavigation.openFolder)

    // 当前工作目录
    resetCwdOnLeave()
    const { updateOnCwdChanged } = useCwdUtils()

    // 实用工具
    const { slicePath } = usePathUtils()

    return {
      networkState,
      folders,
      currentFolderData,
      folderNavigation,
      favoriteFolders,
      toggleFavorite,
      showHiddenFolders,
      createFolder,
      updateOnCwdChanged,
      slicePath,
    }
  },
}
```

当然，使用选项 API 时我们不会这么写代码。但是请注意 `setup` 函数读起来几乎像是在口述这个组件尝试做什么——这是基于选项的版本完全丢掉的信息。你还可以根据传递的参数清楚地看到组合函数之间的依赖关系。最后的 return 语句作为单一出口确认暴露给模板的内容。

同样的功能、两套组件定义呈现出对内在逻辑的不同的表达方式。基于选项的 API 促使我们通过 _选项类型_ 组织代码，而组合式 API 让我们可以基于*逻辑关注点*组织代码。

### 逻辑提取与复用

当我们在组件间提取并复用逻辑时，组合式 API 是十分灵活的。一个组合函数仅依赖它的参数和 Vue 全局导出的 API，而不是依赖其微妙的 `this` 上下文。你可以将组件内的任何一段逻辑导出为函数以复用它。你甚至可以通过导出整个 `setup` 函数达到和 `extends` 等价的效果。

让我们来看看这个追踪鼠标位置的例子：

```js
import { ref, onMounted, onUnmounted } from 'vue'

export function useMousePosition() {
  const x = ref(0)
  const y = ref(0)

  function update(e) {
    x.value = e.pageX
    y.value = e.pageY
  }

  onMounted(() => {
    window.addEventListener('mousemove', update)
  })

  onUnmounted(() => {
    window.removeEventListener('mousemove', update)
  })

  return { x, y }
}
```

以下是一个组件如何利用该函数的展示:

```js
import { useMousePosition } from './mouse'

export default {
  setup() {
    const { x, y } = useMousePosition()
    // 其他逻辑...
    return { x, y }
  },
}
```

在组合式 API 版本的文件浏览器组件示例中，我们已经提取了很多工具代码 (例如 `usePathUtils` 和 `useCwdUtils`) 到外部文件中，因为我们发现它们也可以被用在其它组件中。

类似的逻辑复用也可以通过诸如 `mixins`、高阶组件或是 (通过作用域插槽实现的) 无渲染组件的模式达成。网上已经有很多解释这些模式的信息了所以我们不再赘述。更高层面的想法是，相比于组合函数，这些模式都有各自的弊端：

- 渲染上下文中暴露的 property 来源不清晰。例如在阅读一个运用了多个 mixin 的模板时，很难看出某个 property 是从哪一个 mixin 中注入的。

- 命名空间冲突。Mixin 之间的 property 和方法可能有冲突，同时高阶组件也可能和预期的 prop 有命名冲突。

- 性能方面，高阶组件和无渲染组件需要额外的有状态的组件实例，从而使得性能有所损耗。

相比而言，组合式 API：

- 暴露给模板的 property 来源十分清晰，因为它们都是被组合逻辑函数返回的值。

- 不存在命名空间冲突，可以通过解构任意命名

- 不再需要仅为逻辑复用而创建的组件实例。

### 与现有的 API 配合

组合式 API 完全可以和现有的基于选项的 API 配合使用。

- 组合式 API 会在 2.x 的选项 (`data`、`computed` 和 `methods`) 之前解析，并且不能提前访问这些选项中定义的 property。

- `setup()` 函数返回的 property 将会被暴露给 `this`。它们在 2.x 的选项中可以访问到。

### 插件开发

当下许多 Vue 的插件都向 `this` 注入 property。例如 Vue Router 注入 `this.$route` 和 `this.$router`，而 Vuex 注入 `this.$store`。这使得类型推导变得很有技巧性，因为每个插件都要求用户为注入的 property 向 Vue 增加类型定义。

当使用组合式 API 时，我们不再使用 `this`，取而代之的是，插件将在内部利用 [`provide` 和 `inject`](./api.html#依赖注入) 并暴露一个组合函数。以下是一个插件的假设代码：

```js
const StoreSymbol = Symbol()

export function provideStore(store) {
  provide(StoreSymbol, store)
}

export function useStore() {
  const store = inject(StoreSymbol)
  if (!store) {
    // 抛出错误，不提供 store
  }
  return store
}
```

消费者的代码：

```js
// 在根组件中提供 store
//
const App = {
  setup() {
    provideStore(store)
  },
}

const Child = {
  setup() {
    const store = useStore()
    // 使用 store
  },
}
```

注意这个 `store` 也可以通过[全局 API 更改提案](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0009-global-api-change.md#provide--inject)中 App 级别的 `provide` 来提供，但是消费它的组件中的 `useStore` 风格的 API 还是相同的。

## 弊端

### 引入 Ref 的心智负担

Ref 可以说是本提案中唯一的“新”概念。引入它是为了以变量形式传递响应式的值而不再依赖访问 `this`。其弊端如下:

1. 当使用组合式 API 时, 我们需要一直区别「响应式值引用」与普通的基本类型值与对象，这无疑增加了使用本套 API 的心智负担.

   这一层心智负担可以通过名称规范来大大降低 (例如为所有的 ref 名加类似 `xxxRef` 的后缀)，亦或者是使用类型系统。另外，由于代码组织方面的灵活性增加了，组件逻辑会更多地分解成一些短小精悍的函数，它们的上下文都比较简单，这些 ref 的上限也易于控制。

2. 读写 ref 的操作比普通值的更冗余，因为需要访问 `.value`。

   有人建议使用编译时的语法糖 (类似 Svelte 3) 来解决这个问题。这虽然在技术上可行，但我们并不认为 Vue 需要默认支持 (正如在[与 Svelte 比较](#与-svelte-比较)中所讨论的那样)。换而言之，可以在用户端开发一个 Babel 插件来处理。

我们已经讨论过了是否可以完全避免引入 ref 的概念而只使用响应性对象，但是：

- 计算值的 getter 可以返回基础类型值，所以一个类似 ref 的容器是不可避免的。

- 一些组合函数只接收或返回基础类型值，它们也需要被包裹成为一个对象才能够保持其响应性。如果框架不提供标准实现，那么用户最后会发明它们各自的 ref 模式 (从而造成生态的碎片化)。

### Ref vs. Reactive

可以理解的是，用户会纠结用 `ref` 还是 `reactive`。而首先你要知道的是，这两者你都必须要了解，才能够高效地使用组合式 API。只用其中一个很可能会使你的工作无谓地复杂化，或反复地造轮子。

使用 `ref` 和 `reactive` 的区别，可以通过如何撰写标准的 JavaScript 逻辑来比较：

```js
// 风格 1: 将变量分离
let x = 0
let y = 0

function updatePosition(e) {
  x = e.pageX
  y = e.pageY
}

// --- 与下面的相比较 ---

// 风格 2: 单个对象
const pos = {
  x: 0,
  y: 0,
}

function updatePosition(e) {
  pos.x = e.pageX
  pos.y = e.pageY
}
```

- 如果使用 `ref`，我们实际上就是将风格 (1) 转换为使用 ref (为了让基础类型值具有响应性) 的更细致的写法。

- 使用 `reactive` 和风格 (2) 一致。我们只需要通过 `reactive` 创建这个对象。

而只使用 `reactive` 的问题是，使用组合函数时必须始终保持对这个所返回对象的引用以保持响应性。这个对象不能被解构或展开：

```js
// 组合函数：
function useMousePosition() {
  const pos = reactive({
    x: 0,
    y: 0,
  })

  // ...
  return pos
}

// 消费者组件
export default {
  setup() {
    // 这里会丢失响应性!
    const { x, y } = useMousePosition()
    return {
      x,
      y,
    }

    // 这里会丢失响应性!
    return {
      ...useMousePosition(),
    }

    // 这是保持响应性的唯一办法！
    // 你必须返回 `pos` 本身，并按 `pos.x` 和 `pos.y` 的方式在模板中引用 x 和 y。
    return {
      pos: useMousePosition(),
    }
  },
}
```

[`toRefs`](./api.html#torefs) API 用来提供解决此约束的办法——它将响应式对象的每个 property 都转成了相应的 ref。

```js
function useMousePosition() {
  const pos = reactive({
    x: 0,
    y: 0,
  })

  // ...
  return toRefs(pos)
}

// x & y 现在是 ref 形式了!
const { x, y } = useMousePosition()
```

总结一下，一共有两种变量风格：

1. 就像你在普通 JavaScript 中区别声明基础类型变量与对象变量时一样区别使用 `ref` 和 `reactive`。我们推荐你在此风格下结合 IDE 使用类型系统。

2. 所有的地方都用 `reactive`，然后记得在组合函数返回响应式对象时使用 `toRefs`。这降低了一些关于 ref 的心智负担，但并不意味着你不需要熟悉这个概念。

在这个阶段，我们认为现在就强制决定 `ref` vs. `reactive` 的最佳实践还为时过早。我们建议你对以上两种方式都进行尝试，选择与你的心智模型更加配合的风格。我们将持续收集周边生态中的用户反馈，并最终在这个问题上提供更明确、更统一的实践指导建议。

### 返回语句冗长

一些用户会顾虑 `setup()` 的返回语句变得冗长，像是重复劳动。

我们相信明确的返回语句对可维护性是有益的。它使我们能够显式地控制暴露给模板的内容，并作为起点，追踪模板中某个 property 在组件哪里被定义的。

有些建议希望自动暴露 `setup()` 中声明的变量，使 `return` 语句变为可选的。再次重申，我们不认为这应该是默认的行为，因为它违背了标准 JavaScript 的直觉。不过还是有一些方法可以让用户侧的工作变少：

- 开发 IDE 插件自动将 `setup()` 中定义的变量插入到返回值语句中

- 开发 Babel 插件来隐式地生成并插入 `return` 语句。

### 更多的灵活性来自更多的自我克制

许多用户指出，虽然组合式 API 在代码组织方面提供了更多的灵活性，但它也需要开发人员更多地自我克制来 “正确地完成它”。也有些人担心 API 会让没有经验的人编写出面条代码。换句话说，虽然组合式 API 提高了代码质量的上限，但它也降低了下限。

我们在一定程度上同意这一点。但是，我们认为：

1. 提升上界的收益远远大于降低下界的损失。

2. 通过适当的文档和社区指导，我们可以有效地解决代码组织问题。

一些用户用 Angular 1 的控制器作为例子，来说明这种设计是如何导致糟糕代码的。组合式 API 和 Angular 1 控制器之间最大的不同是它不依赖于一个共享的作用域上下文。这使得将逻辑划分为单独的函数变得非常容易，这正是 JavaScript 代码组织的核心机制。

任何 JavaScript 程序都是从一个入口文件开始的 (将它想象为程序的 `setup()`)。我们根据逻辑关注点将程序分解成函数和模块来组织它。**组合式 API 使我们能够对 Vue 组件代码做同样的事情**。换句话说，编写有组织的 JavaScript 代码的技能直接转化为了编写有组织的 Vue 代码的技能。

## 接纳策略

组合式 API 完全是可被添加的，且不会影响/废弃任何现有的 2.x API。通过 [`@vue/composition` 这个库](https://github.com/vuejs/composition-api/)，它已经以插件的形式在 2.x 中生效了。这个库的基本宗旨是提供一个机会让大家体验新的 API 并收集反馈。当前的实现已经与本提案内的内容保持同步，但是由于插件的技术限制可能会包含一些微观的不一致。随着提案的更新，它也可能会做一些不兼容的变更，所以我们不建议这个阶段在生产环境中使用它。

我们会将这一套 API 内建在 Vue 3.0 中，它将与现有的 2.x 选项同时可用。

对于想要在应用程序中仅使用组合式 API 的用户，我们可能会提供一个编译时标记来删除那些仅用于 2.x 选项的代码来减小整个库的大小，这是完全可选的。

这个 API 将被定位为高级特性，因为它旨在解决的问题主要出现在大型应用程序中。我们不打算彻底修改文档来把它用作默认方案。相反，它将在文档中有自己的专用部分。

## 附录

### Class API 的类型问题

引入 Class API 的主要目的是提供另一种 API 以更好地支持 TypeScript 类型推导。但是，Vue 组件需要将声明自多个来源的 property 合并到一个 `this` 上下文中，这给基于 Class 的 API 带来了一些挑战。

Prop 的类型是其中一个例子。为了把 prop 合并到 `this` 中，我们不得不为组件的 Class 使用一个泛型参数，或使用一个 decorator。

这里是一些使用泛型参数的示例：

```ts
interface Props {
  message: string
}

class App extends Component<Props> {
  static props = {
    message: String,
  }
}
```

由于传递给泛型参数的 interface 仅仅是类型标注，所以用户仍然需要在 `this` 上为 prop 的代理行为提供其运行时声明。这种二次声明是多余且笨拙的。

我们考虑过用 decorator 作为替代方案：

```ts
class App extends Component<Props> {
  @prop message: string
}
```

使用 decorator 导致它依赖一个带有很大不确定性的 stage-2 提案，特别是现在 TypeScript 的实现方案已经完全不是与 TC39 提案相同步的。除此之外没有什么太好的办法能够将定义在 `this.$props` 上的 prop 类型声明暴露出来， 这使得难以支持 TSX。从而破坏 TSX 支持。另外用户也可能会认为他们可以通过 `@prop message: string = 'foo'` 为某些 prop 声明默认值，但技术上这其实并不可用。

加之目前还没有很好的办法利用上下文的类型标注来推导 class 方法的参数——这意味着传递给 class 的 `render` 函数的参数，不能基于 class 的其它 property 来推导参数类型。

### 与 React Hooks 相比

基于函数的组合式 API 提供了与 React Hooks 同等级别的逻辑组合能力，但是与它还是有很大不同：组合式 API 的 `setup()` 函数只会被调用一次，这意味着使用 Vue 组合式 API 的代码会是：

- 一般来说更符合惯用的 JavaScript 代码的直觉；
- 不需要顾虑调用顺序，也可以用在条件语句中；
- 不会在每次渲染时重复执行，以降低垃圾回收的压力；
- 不存在内联处理函数导致子组件永远更新的问题，也不需要 `useCallback`；
- 不存在忘记记录依赖的问题，也不需要“useEffect”和“useMemo”并传入依赖数组以捕获过时的变量。Vue 的自动依赖跟踪可以确保侦听器和计算值总是准确无误。

我们感谢 React Hooks 的创造性，它也是本提案的主要灵感来源，然而上面提到的一些问题存在于其设计之中，且我们发现 Vue 的响应式模型恰好为解决这些问题提供了一种思路。

### 与 Svelte 比较

虽然采取了非常不同的方法，但是组合式 API 和 Svelte 3 基于编译思想的方法实际上在概念上有很多相同之处。这里有一个对比示例:

#### Vue

```html
<script>
  import { ref, watchEffect, onMounted } from 'vue'

  export default {
    setup() {
      const count = ref(0)

      function increment() {
        count.value++
      }

      watchEffect(() => console.log(count.value))

      onMounted(() => console.log('mounted!'))

      return {
        count,
        increment,
      }
    },
  }
</script>
```

#### Svelte

```html
<script>
  import { onMount } from 'svelte'

  let count = 0

  function increment() {
    count++
  }

  $: console.log(count)

  onMount(() => console.log('mounted!'))
</script>
```

Svelte 的代码看起来更简洁，因为它在编译时做了以下工作:

- 隐式地将整个 `<script>` 块 (import 语句除外) 包装到一个函数中，该函数会被每个组件实例调用 (而不是只执行一次)
- 隐式地为变量的变更注册响应性
- 隐式地向渲染上下文暴露所有作用域内的变量
- 将 `$` 语句编译为重复执行的代码

从技术上讲，我们在 Vue 中也可以通过可能的 Babel 插件做到这些，但我们不想这样做的原因是：**想要与标准 JavaScript 保持一致**。如果要在 Vue 文件中提取 `<script>` 块的代码，我们希望它与标准的 ES 模块完全相同。Svelte 中 `<script>` 块的代码在技术上已经不再是标准的 JavaScript 了。我们觉得这种基于编译器的方法存在一些问题：

1. 在有或没有编译时，代码工作的方式不同。作为一个渐进的框架，许多 Vue 用户可能希望/需要/不得不在没有构建环节的情况下使用，因此编译的版本不会作为默认方案。另一方面，Svelte 将自己定位为一种编译器且*只*用于构建环节。这是两个框架都在做的有意识的权衡。

2. 代码在组件内部和外部的工作方式不同。当试图从一个 Svelte 的组件中提取逻辑并将其放入标准的 JavaScript 文件时，我们将失去那些神奇简洁的语法糖，而不得不退回到[更冗长的底层 API](https://svelte.dev/docs#svelte_store)。

3. Svelte 的响应性编译只适用于顶层变量——它不触及函数内声明的变量，所以我们[无法将响应式状态封装在一个函数中并声明在一个组件内](https://svelte.dev/repl/4b000d682c0548e79697ddffaeb757a3?version=3.6.2)。这在通过函数来组织代码时带来了不小的约束，正如我们在这个提案中所演示的，这对于保持大型组件的可维护性非常重要。

4. [非标准语义使其与 TypeScript 的集成存在困难](https://github.com/sveltejs/svelte/issues/1639).

这并不是说 Svelte 3 不好——事实上，它是一个非常创新的想法，并且我们高度尊重 Rich 的工作。但是基于 Vue 在设计上的克制及其目标，我们不得不做出不同的权衡。
