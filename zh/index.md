---
sidebar: auto
---

# 组合式 API 征求意见稿

- 开始日期: 2019-07-10
- 目标主版本: 2.x / 3.x
- 相关 Issues 链接: [#42](https://github.com/vuejs/rfcs/pull/42)
- 相关实现的 Pull Request: (leave this empty)

## 概述

在此我们将为您介绍 **Composition API**: 一组低侵入式的、函数式的 API，使得我们能够更灵活地「**组合**」组件的逻辑。

<iframe src="https://player.vimeo.com/video/365349055" width="640" height="360" frameborder="0" allow="autoplay; fullscreen" allowfullscreen></iframe>

观看 Vue Mastery 上的 [Vue 3 基础课程](https://www.vuemastery.com/courses/vue-3-essentials/why-the-composition-api/). 点击此链接下载 [Vue 3 代码笔记](https://www.vuemastery.com/vue-3-cheat-sheet/).

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

1. 随着功能的增长，复杂组件的代码变得越来越难以阅读和理解。这种情况在开发人员阅读他人编写的代码时尤为常见。根本原因是 Vue 现有的 API 强制通过「**选项**」组织代码，但是在某些情况下，通过逻辑关系组织代码更有意义。

2. 目前缺少一种简洁且低成本的机制来提取和重用多个组件之间的逻辑。 (更多细节详见这里的描述： [逻辑抽象、提取与复用](#逻辑抽象、提取与复用))

RFC 中 提出的 API 为用户在组织组件代码时提供了更大的灵活性。现在我们不需要总是通过「选项」来组织代码，而是可以将代码组织为处理特定功能的函数。这些 API 还使得在组件之间甚至组件外部提取和重用逻辑变得更加简单。
我们会在这一小节中 [设计细节](#设计细节) 为你展示我们想要达成的效果。

### 更好的类型推导

另外一部分功能需求的声音来自一些大型项目的开发者，呼吁更好的 TypeScript 支持。Vue 当前的 API 在集成 TypeScript 时遇到了不小的麻烦，其主要原因是：我们实际上十分依赖一个简单的 `this` 上下文来提供各种数据、方法等属性，我们现在使用 `this` 的方式是比较玄学的。（比如说：我们访问某个组件的方法时是直接从 `this` 上取而不是在 `methods`上取得的）。

换句话说，Vue 当前的各种 API 在设计时其实并没有基于类型推导的思想，这为我们之后想要适配 TypeScript 带来了很大挑战。

当前，大部分在 Vue 项目中应用 TypeScript 的开发者，都选择了官方提供的 `vue-class-component`，这个库使得 Vue 组件的写法变为一个 TypeScript 的类。 (而且运用了 Decorators 装饰器). 当我们在设计 3.0 时, 我们原本希望提供一个内建的 Class-API 来更好的解决「类型标注（typing）」相关的问题 [详见此已废弃 RFC](https://github.com/vuejs/rfcs/pull/17). 然而，当我们讨论到一些细节和具体取实现此方案时，我们注意到 **它必须依赖于「装饰器」** - 但此 ECMAScript 提案现在严重的不稳定，有许多并不完全确定的实现细节，基于它来构建新版本的 Vue 是有极大风险的。([关于 Class-API 的类型相关问题请移步这里](#class-api-的类型问题))

相比较过后，本 RFC 中提出的方案更多地是去利用原始的变量与函数，而恰恰它们在 TypeScript 是天然「**类型友好**」的，在使用我们现在提出的这种 API 写代码时，你会享受到完美的类型推导，并且也不用做太多额外的类型标注。

这也同样意味着：**你写出的 JavaScript 代码几乎就是 TypeScript 的代码**，所以即使是不想采用 TypeScript 的开发者也会从（例如 Visual Studio Code）IDE 给出的类型提示中受益。

## 设计细节

### API 介绍

为了不引入太多全新的概念，API 中提供的更多是 Vue 的核心功能。比如用独立的函数来创建和监听响应式的状态等...

在这里我们会介绍许多最基本的 API，它们可以被用于替换 2.x「Options API」当中的一些「组件内逻辑」

**请注意：** 这一小节中我们主要是介绍这些 API 的基本思想所以我们不会太多深入展开其细节。更多的使用细节请移步 [API 参考](./api) 章节。

#### 响应式状态 与 被动响应

让我们从一个简单的任务开始：创建一个响应式的状态

```js
import { reactive } from 'vue'

// state 现在是一个响应式的状态
const state = reactive({
  count: 0,
})
```

`reactive` 几乎等价于我们现有 2.x 中的 `Vue.observable()` API，重命名成这样是为了与 RxJS 中的 `observables` 区别开来。在这里，返回的 `state` 是一个响应式的对象，关于 “响应式” 的概念相信所有的 Vue 使用者都比较熟悉了。

我们可以将这个基本的示例的使用扩展到界面内容渲染上，因为有了依赖收集的机制，当响应式的状态发生改变时，视图会自动地进行更新。在 DOM 当中渲染些什么被视为一种「**被动响应 effect**」：程序在执行过程中修改了这个变量，同时将其状态的改变所引发的副作用反映出来（_这里指的就是 DOM 内容的变更_）。

对此我们可以使用 `watchEffect` 这个 API：

```js
import { reactive, watchEffect } from 'vue'

const state = reactive({
  count: 0,
})

watchEffect(() => {
  document.body.innerHTML = `count is ${state.count}`
})
```

`watchEffect` 期望接收一个函数，这个函数定义之中，应该描述了被动响应所期望的结果。**刚开始时它将会立即执行这个函数**，因而可以对这个过程中用到的响应式状态进行「**追踪监听 track**」，将其作为这个被动响应的「**依赖 dependency**」。

也就是在示例中，`state.count` 在首次执行后，会被追踪，成为一个依赖。当 `state.count` 未来发生变化时，里面这个函数又会被重新执行。

这正是 Vue 目前整个响应式系统的精髓所在了！在 2.x 中，当你从 `data()` 函数中返回一个对象，内部实质上通过调用 `reactive()` 使其变为响应式。而 `template` 模板会被编译为「渲染函数（render function）」（你可以把它想成一种更复杂的 `innerHTML` 变更方式），故而可以将响应式状态应用其中。

> `watchEffect` 方法和 2.x 中的 `watch` 选项比较类似，但是它略有不同，参数中并不要求分开书写 _被监听的数据对象_ 和 _被动响应的回调函数_。不过 Composition API 同样提供了一个 `watch` 方法，参数及用法完全与 2.x 的选项表现一致。

继续我们上面的例子，下面我们将展示如何处理用户输入:

```js
function increment() {
  state.count++
}

document.body.addEventListener('click', increment)
```

但是在 Vue 的模板系统当中，我们当然不需要完全按照原生 JavaScript 一样，去手动的去调整 `innerHTML` 或是去挂载事件监听。我们将例子中渲染相关的逻辑简化为一个假设的 `renderTemplate` 方法即可，这样我们就可以更多关注响应式方面的内容了：

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

#### 计算属性 与 响应式值引用

有时候，我们会需要一个 _依赖于其他状态的_ 状态，在 Vue 中我们将它称为 **计算属性**，我们可以使用 `computed` 方法直接创建一个计算属性。

```js
import { reactive, computed } from 'vue'

const state = reactive({
  count: 0,
})

const double = computed(() => state.count * 2)
```

这个 `computed` 返回了什么? 我们可以猜想一下 `computed` 内部是如何实现的，你可能会想出下面这样的方案:

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

但实际上并不是这样的：如果 `value` 只是一个 JavaScript 中的基础值类型，例如 `number` 类型，当被返回时，它与这个 computed 句柄函数 `getter` 之间的关系就丢失了！
这是由于 JavaScript 中基础类型是 **值传递** 而非 **引用传递**。

![值传递 vs 引用传递](https://www.mathwarehouse.com/programming/images/pass-by-reference-vs-pass-by-value-animation.gif)

在 “将基础类型的值，赋给某个对象作为其属性” 时也会出现同样的问题。我们不想让某个响应式的值 _在其被赋值成为一个属性时_、_或在函数中被返回时_ 丢失了其响应性。为了确保我们可以一直读取到最新的计算结果，我们需要将这个值上包裹成一个对象并返回它。

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

另外你必须知道的是：我们同样需要 **劫持** 对这个对象 `.value` 属性的 读/写 操作，来实现依赖收集与被动响应的更新。（为了缩减篇幅我们没有过多展示内部实现的原理）

现在我们可以通过引用来传递计算属性值了！也不需要担心其响应式特性会丢失。当然代价就是：为了获取此值，我们每次都需要写 `.value`。

```js
const double = computed(() => state.count * 2)

watchEffect(() => {
  console.log(double.value)
}) // -> 0

state.count++ // -> 2
```

**在这里 `double` 是一个对象，我们管它叫 "ref", 它作为它内部值的一个响应式引用。**

> 你可能会担心说：“诶！Vue 本身不是已经有 "refs" 的概念了么？”
>
> 原来的 `ref` 只是为了获取 DOM 元素或组建实例的「 template refs 模板引用」，更多详见这里的 [API 参考](./api.html#template-refs)，查看新的 refs 系统是如何同时兼容 _逻辑引用值_ 与 _模板引用_ 的。

在 `computed` 之外，我们同样提供了一个 API 来直接创建一个「**响应式值引用 ref**」：

```js
const count = ref(0)
console.log(count.value) // 0

count.value++
console.log(count.value) // 1
```

#### Ref 的自动解套

::: v-pre
我们可以在 `setup` 中将 `ref` 值暴露给模板渲染，在渲染过程中，Vue 会特别地为这些响应式值引用解套，上下文会直接使用其内部的值，也就是说你在写模板用到这个 `ref` 值时，不用再特意写 `.value` 了。
:::

这是计数器示例的另一个版本, 使用的是 `ref` 而不是 `reactive`:

```js
import { ref, watch } from 'vue'

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

除此之外，当一个 `ref` 值嵌套于响应式对象之中时，访问时会自动解套:

```js
const state = reactive({
  count: 0,
  double: computed(() => state.count * 2),
})

// 无需再使用 `state.double.value` 访问其值
console.log(state.double)
```

#### 组件中的使用方式

到目前为止，我们的代码已经提供了一个可以根据用户输入进行更新的 UI ，但是代码只运行一次，无法重用。如果我们想重用其逻辑，那么不如重构成一个函数:

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

现在，如果我们把调用 `setup()`、创建 `computed` 计算属性 与模板渲染的逻辑组合在一起，其实就正是用它们来定义了一个 Vue 组件:

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

看！这不就变成我们熟悉的「**单文件组件**」了么，只是逻辑部分 (`<script>`块) 格式表现得稍有不同，模板语法与原先保持一致（另外省略了 `<style>`）。

#### 生命周期钩子函数

到此我们已经覆盖到了一个组件的状态的所有情况，响应式状态、计算属性和处理用户输入时的状态变化。但是一个组件本身可能还有其他的被动响应：例如打印信息到控制台、发送 AJAX 请求或是在全局 `window` 对象上设置事件监听器等等... 这些被动响应大多会发生在如下这些时间节点上：

- 状态变化时;
- 组件挂载完成、内容更新或者解除挂载时 (这就对应了生命周期钩子)

我们已经知道可以使用 `watchEffect` 和 `watch` API 来提供状态变化时的被动响应。为了表现出在一些生命周期时的被动响应，
我们定义来一些形似 `onXXX` 的 API，与我们 2.x 现有的生命周期选项对应。

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

生命周期钩子表现为一种注册的方法，只在 `setup` 中使用。它会通过一个内部的全局状态自动找到调用此 `setup` 钩子方法的组件，这样设计是为了减少将逻辑提取到外部函数时的冲突。

> 关于这些 API 的更多细节可以在 [API 参考](./api)页面中找到。但是在深入挖掘使用细节之前，我们建议你先将下面介绍核心思想的小节看完。

### 代码组织与管理

此时我们已经将原来 Options API 中的逻辑全部过继到了新的 API 中，但是这么做的目的是什么？用选项来定义组件难道不比把所有东西揉在一个大大的函数里更有条理、更具组织性么？

有这样的第一印象是可以理解的。但是正如在 _动机与目的_ 部分所提到的，我们相信组合式的 API 实际上能够为你的代码带来更好组织结构，尤其是在复杂的组件中，下面我们将解释为什么。

#### 到底什么才是 "组织较好的代码"

让我们先暂退一步，考虑一下这个概念的真正内涵。最后我们的目标无外乎是 “让代码更具可读性、更容易被理解”，那么怎么才叫 “理解” 了代码呢？难道光是知道了它定义了哪些选项就算了解了一个组件本身么？你是否有接手过他人开发过的一个庞大组件？比如 [你看看这个](https://github.com/vuejs/vue-cli/blob/a09407dd5b9f18ace7501ddb603b95e31d6d93c0/packages/@vue/cli-ui/src/components/folder/FolderExplorer.vue#L198-L404)，你会不会觉得这东西实在让人头大？

那么想一想，我们怎么才能让后来的开发者弄懂上面链接里这样一个大型组件的内在逻辑？你可能大致会开始分析道：“这个组件是在处理 X、Y 与 Z” 而不会说 “这个组件有这些那些个数据属性（`data`）、这些那些个计算属性（`computed`）和这些那些个方法（`methods`）”。

当要去理解一个组件时，我们更加关心的是 “这个组件是要干什么”（也就是说：代码对应的的需求、所展现的意图）而不是 “这个组件用到了什么选项”。但我们的 Options API 实际上就是后者。

#### 逻辑关注 vs. 选项类型

我们不妨将组件处理的 `X、Y 和 Z` 定义为此组件的「**逻辑关注**」，可读性的问题基本不会存在于小的、单一用途的组件中，因为那样的组件，整个都在处理单个逻辑关注点，然而，这个问题在复杂的用例中会变得突出。以 [Vue CLI UI 文件浏览器](https://github.com/vuejs/vue-cli/blob/a09407dd5b9f18ace7501ddb603b95e31d6d93c0/packages/@vue/cli-ui/src/components/folder/FolderExplorer.vue#L198-L404) 为例，这个组件有非常多的逻辑关注点：

- 追踪监听当前文件夹的状态并展示其中的内容
- 处理文件夹的操作（打开、关闭、刷新或是其他...）
- 处理「新建文件夹」的操作
- 控制「收藏文件夹」展开与否
- 控制「隐藏文件夹」展开与否
- 对「文件夹路径变化」作出响应

相对应的，你能仅仅通过基于选项的代码来梳理出各个逻辑关注点么？显然是十分困难的，因为最后我们会发现到与各个逻辑关注点相关的代码是分散在各处的。

例如：「创建新文件夹」的功能使用到了 [两个数据属性](https://github.com/vuejs/vue-cli/blob/a09407dd5b9f18ace7501ddb603b95e31d6d93c0/packages/@vue/cli-ui/src/components/folder/FolderExplorer.vue#L221-L222)、[一个计算属性](https://github.com/vuejs/vue-cli/blob/a09407dd5b9f18ace7501ddb603b95e31d6d93c0/packages/@vue/cli-ui/src/components/folder/FolderExplorer.vue#L240) 和 [一个方法](https://github.com/vuejs/vue-cli/blob/a09407dd5b9f18ace7501ddb603b95e31d6d93c0/packages/@vue/cli-ui/src/components/folder/FolderExplorer.vue#L387)，而方法的定义在距离属性约一百多行的位置。

如果我们对这些逻辑问题进行染色，我们会注意到它们在用组件选项表示时是多么分散:

<p align="center">
  <img src="https://user-images.githubusercontent.com/499550/62783021-7ce24400-ba89-11e9-9dd3-36f4f6b1fae2.png" alt="file explorer (before)" width="131">
</p>

正是这种碎片化使得理解和维护一个复杂的组件变得非常困难。强制的选项分离掩盖了潜在的逻辑问题。此外，在处理单个逻辑关注点时，我们必须不断地 “跳转” 选项的代码块，以找到与该关注点相关的部分。

> **注意:** 原始代码可能在某些地方有些改动，但是我们展示了在撰写本文时的最新提交，没有进行任何修改，目的就是为了提供了一个我们自己编写的、实际生产代码的示例。

如果我们能够将与相同逻辑关注点对应的代码并列组合，那就再好不过了。这正是组合式 API 所能做到的，“创建新文件夹”功能可以这样写:

```js
function useCreateFolder(openFolder) {
  // 原来的数据属性改成这样写：
  const showNewFolder = ref(false)
  const newFolderName = ref('')

  // 原来的计算属性改成这样写：
  const newFolderValid = computed(() => isValidMultiName(newFolderName.value))

  // 原来的一个方法改成这样写：
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

请注意，与「创建新文件夹」功能相关的所有逻辑现在都被合并并封装在一个函数中。由于其读起来具有描述性的名称，该函数的代码在某种程度上也是自文档化的、更易读的。这就是我们所称的：**组合逻辑函数**。建议使用 `use` 作为函数名的开头，以表示它是一个组合逻辑函数。

将此模式用于组件中所有的逻辑关注点，代码中各个部分间将更好地解耦和重组，来看看下图的对比吧:

<p align="center">
  <img src="https://user-images.githubusercontent.com/499550/62783026-810e6180-ba89-11e9-8774-e7771c8095d6.png" alt="file explorer (comparison)" width="600">
</p>

> 这个比较不包括 import 语句和 setup()函数。使用复合 API 重新实现的完整组件可以在 [这里](https://gist.github.com/yyx990803/8854f8f6a97631576c14b63c8acd8f2e) 找到。

每个逻辑关注点的代码现在都被配置在一个组合逻辑函数中。这大大减少了在处理大型组件时不断 “跳转” 的需要。同时这一整个功能所对应的函数也可以在编辑器中折叠，使组件更容易浏览:

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

`setup()` 函数现在只是简单地作为所有「组合逻辑函数」的调用入口。

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

    // （从）其他工具函数（解构一些方法）
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

当然，使用 Options API 时我们完全不会这么写代码。但是请注意，`setup` 函数读起来就已经几乎是对「组件正在尝试做什么」的口头描述了 —— 这是基于选项的版本中完全没有的信息。您还可以根据传递的参数清楚地看到复合函数之间的依赖关系。最后，`return` 语句用作检查模板中暴露的内容的单一出口。

同样的功能、两套组件定义呈现出对内在逻辑的不同的表达方式。基于选项的 API 促使我们通过「_选项类型_」来管理代码，而组合式 API 是我们拥有了基于「_逻辑关注点_」管理代码的新套路。

### 逻辑抽象、提取与复用

组合式 API 是十分灵活的，因此我们可以在组件间抽象和重用一些逻辑，而不是依赖于一个玄学的 `this` 上下文，一个组合逻辑函数仅仅只依赖于他的参数和一些的 Vue 本身的 API，你可以重用组件内的任何一段逻辑，只要你将它导出成为一个函数即可。你甚至可以通过导出整个 `setup` 函数，做出与 `extends`（继承）等价的效果。

👀 让我们来看看这个例子：「**监听鼠标位置**」

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

以下是一个组件如何利用该功能的展示:

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

在组合式 API 版本的文件浏览器组件示例中，我们已经抽象、提取了很多工具代码（例如：`usePathUtils` 和 `useCwdUtils`）到外部文件中，因为我们发现它们也可以被用在其他组件中。

同样的逻辑复用，在之前的版本中，我们也有过一些解决方案，比如：`mixins`、高阶组件或是通过 `scoped slots` 实现的低渲染组件。网上已经有很多解释这些实践方式的信息了所以我们不再赘述。这些想法在和组合逻辑函数相比之后，都被发现有明显的缺点。

- **暴露的数据来源并不清晰。** 在一个模板下运用到的多个 mixins，很难分析得出某个数据是从哪一个 mixins 中得来的

- **命名空间冲突。** 多个 mixins 很可能恰好占用同一个属性名称。

- **性能较差。** 在 Vue 中高阶组件可能并不如 React 当中使用的多，原本是为了替代 mixins，但其实这个设计并没有更好，多个 `props` 命名空间依旧会有冲突，`props` 的来源也并不清晰，还额外带来了许多组件级别的、无谓的性能消耗。作用域插槽虽然没有了命名空间冲突、数据来源也清晰了，但是仍然是一个组件级的性能消耗。

相比而言，Composition API 具有如下的优势:

- 暴露给模板的数据来源十分清晰，因为它们都是被组合逻辑函数返回的值。

- 不存在命名空间冲突，因为在取值时，可以通过解构的语法进行变量重新命名

- 只是单纯的逻辑复用而没有额外的组件性能消耗

### 与现有的 API 配合

组合式 API 完全可以和现有的 API 配合使用。

- 组合式 API 在 2.x 的选项 (`data`, `computed` 和 `methods`) 之前解析，并且不能提前访问这些选项中定义的属性。

- `setup()` 函数返回的属性将会被暴露给 `this` 上下文，在 2.x 的选项中也可以访问到。

### 插件开发

当下的许多 Vue 插件都将属性注入到了 `this` 上，例如, Vue Router 路由库就注入了 `this.$route` 和 `this.$router`, 而 Vuex 注入了 `this.$store`。这使得类型推断变得棘手，因为每个插件都要求用户为注入的属性和 Vue 本身的类型定义进行关联。

当使用组合式 API 时，我们不再使用 `this`，相应地, 插件将在内部利用[`provide` 和 `inject`](./api.html#依赖注入) 然后暴露一个组合逻辑函数。以下是一个状态管理插件的假设代码:

生产者部分：

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

消费者部分:

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
    // 使用状态仓库 store
  },
}
```

注意！这个状态仓库 `store` 只能在 App-层级（即根组件上）提供，`provide` 是[全局 API 更改提案](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0009-global-api-change.md#provide--inject)中提出的 API, 但是消费者组件代码部分中 `useStore` 这类的 API 还是由和原来一样由插件提供。

## 缺点

### 引入 Refs 的心智负担

Ref「**响应式值引用**」可以说是本提案中唯一的 “新” 概念. 引入它是为了不再依赖 `this` 上下文，而是可以不受太多范围限制地、像一般变量一样获取响应式的值。缺点如下:

1. 当使用组合式 API 时, 我们需要一直区别「响应式值引用」与普通的基本类型值与对象，这无疑增加了使用本套 API 的心智负担.

   这一层心智负担可以通过名称规范来大大降低（例如：可以把所有的响应式值引用命名为 `xxxRef`），亦或者是使用类型系统。另外，由于代码组织方面的灵活性增加了，组件逻辑会更多地分解成一些短小精悍的函数，它们的上下文都比较简单，也更容易管理这些 refs。

2. 读写 refs 写起来感觉更冗余，因为需要特别访问它的 `.value`。

   有些建议中提出可以提供编译时的语法糖（类似 Svelte 3）来解决。 虽然它在技术上是可行的，但我们不认为在 Vue 应该默认这么做。(在[与 Svelte 比较](#与-svelte-比较)中有更详细的讨论). 我们保留对其技术上的认可，它可以作为一个 Babel 插件提供给用户。

我们已经讨论过了 “是否可以完全避免引入 Ref 概念，只使用响应性对象”，但是:

- 计算属性时常返回的是基础类型的值, 所以一个 Ref-like 的响应式引用容器是不可避免的。

- 一些组合逻辑函数接收的参数或返回的结果为基础类型的值时也需要被包裹成为一个对象，才能够保持其响应性。而如果框架官方不提供一套标准的实现方式，那么最后会有各式各样不同的 API，造成社区生态的分裂。

### Ref vs. Reactive

我们预见到, 用户们可能会在何时使用 `ref` 和何时使用 `reactive` 的选择上产生纠结，而首先你要知道的是，**这两者你都必须要了解** 才能够高效地使用组合式 API。专门使用一个将很可能将你的工作无谓地复杂化，或导致没有意义地反复造轮子。

使用 `ref` 和 `reactive` 的区别，可以通过比较对待一般 JavaScript 逻辑的两种不同方式来体现：

```js
// 风格 1: 分开的多个变量
let x = 0
let y = 0

function updatePosition(e) {
  x = e.pageX
  y = e.pageY
}

// --- 与下面的相比较 ---

// 风格 2: 一个对象，总领各个属性
const pos = {
  x: 0,
  y: 0,
}

function updatePosition(e) {
  pos.x = e.pageX
  pos.y = e.pageY
}
```

- 如果使用 `ref`, 我们实际上就是在体现 风格 (1) 中的代码，而使用 refs 只是为了让基础类型的值具有响应性。

- 使用 `reactive` 基本与 风格 (2) 是一致的。我们只需要多调用一下 `reactive` 方法就好。

而只使用 `reactive` 会造成的问题是，组合逻辑函数使用时必须始终保持对其中的属性都进行 “引用化”，以保持响应性，否则解构或扩展两种操作都会丢失响应性:

```js
// 组合逻辑函数：
function useMousePosition() {
  const pos = reactive({
    x: 0,
    y: 0,
  })

  // ...
  return pos
}

// 假设的一个组件
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
    // 你必须返回 pos 作为返回对象的一个属性，并按 `pos.x` 和 `pos.y` 的方式来来在模板中引用 x 和 y。
    return {
      pos: useMousePosition(),
    }
  },
}
```

除此之外 [`toRefs`](./api.html#torefs) API 是另一种我们提供出来解决此约束的办法，它将对象的每个属性值都套了一层 `ref`。

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

总结一下，一共有两种办法：

1. 就像你区别声明基础类型变量与对象变量时一样，区别使用 `ref` 和 `reactive`，我们推荐你在此风格下更多地使用 IDE 的语法提示帮助。

2. 所有的地方都用 `reactive`, 然后最后记得在组合逻辑函数返回响应式对象时，使用 `toRefs` 处理一下。这降低了一些关于 refs 的心智负担，但这并不意味着你不需要熟悉这个概念。

在这个阶段，我们认为现在就强制决定 `ref` vs. `reactive` 的最佳实践还为时过早。我们建议你对以上两种方式都进行尝试，选择与你的心智模型更加配合的风格。我们将持续收集周边生态中的用户反馈，并最终在这个问题上提供更明确、更统一的实践指导建议。

### 返回语句冗长

一些用户提出 `setup()` 函数的返回语句十分重复、冗余，很多是重复劳动。

我们相信明确的返回语句对可维护性是有益的。它使我们能够显式地控制暴露给模板的内容，并在跟踪组件中定义模板属性的位置时充当起点。

建议自动公开在 `setup()` 中声明的变量，使 `return` 语句变为可选的。同样我们不认为这应该是框架官方默认的行为，因为它违背了标准 JavaScript 的直觉。然而，有以下一些可能的方法可以让它在用户中变得不那么繁琐:

- 开发 IDE 插件自动将 `setup()` 中定义的变量插入到返回值语句中

- 开发 Babel 插件来隐式地生成并插入 `return` 语句。

### 更多的灵活性同样需要更多规范

许多用户指出，虽然组合式 API 在代码组织方面提供了更多的灵活性，但它也需要开发人员遵守更多的规范来 “正确完成”。有些人担心 API 会让没有经验的人编写出复杂的代码。换句话说，虽然组合式 API 提高了代码质量的上限，但它也降低了下限。

我们在一定程度上同意这一点。但是，我们认为:

1. 上界的收益远远大于下界的损失。

2. 通过适当的文档和社区指导，我们可以有效地解决代码组织问题。

一些用户用 Angular 1 的控制器作为例子，来说明这种设计是如何可能导致糟糕代码的。组合式 API 和 Angular 1 控制器之间最大的不同是它不依赖于一个共享的范围上下文。这使得将逻辑划分为单独的函数变得非常容易，这正是 JavaScript 代码组织的核心机制。

任何 JavaScript 程序都是从一个输入文件开始的(可以将它看作是程序的 `setup()`)。我们根据逻辑关系将程序分解成函数和模块来组织它。**组合式 API 使我们能够对 Vue 组件代码做同样的事情。** 换句话说，编写组织良好的 JavaScript 代码的技能直接转化为编写组织良好的 Vue 代码的技能。

## 升级策略与建议

组合式 API 完全是可选的、可被添加到现有项目中的，也不会对现有的 2.x API 们造成任何影响. 它已经被提供为了一个 2.x 插件，详见[`@vue/composition` 这个库](https://github.com/vuejs/composition-api/). 这个库的基本宗旨是提供一个机会让大家体验新的 API 并收集用户反馈。当前的实现已经与本提案内的内容保持一致, 但是由于插件的技术限制，可能会包含一些不一致的地方。随着这个方案的更新，它也可能会有一些破坏性的大变化，所以我们不建议这个阶段在生产环境中使用它。

我们会将这一套 API 内建在 Vue 3.0 中，它将与 2.x 的选项式 API 同时存在，两者皆可用。

对于想要在应用程序中仅使用组合式 API 的用户，我们可能会提供一个编译时标记来删除那些仅适用于 2.x 的代码来减小整个库的大小，但这应当是完全可选的。

这个 API 将被定位为一个额外的高级特性，因为它旨在解决的问题主要出现在大型应用程序中。我们不打算彻底修改文档来使用它作为默认建议。相反，它将在文档中有自己的专用部分。

## 附录

### Class-API 的类型问题

引入 Class-API 的主要目的是提供一个具有更好的 TypeScript 类型推导，但是，Vue 组件需要将多个源声明的属性合并到一个上下文中，这事实上给基于类的 API 带来了很大挑战。

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

由于传递给泛型参数的接口（这里的 `Props`）是仅仅是类型标注而已，所以用户仍然需要为 `this` 上的 `props` 代理行为提供一个运行时的 `props` 声明，这种二次声明是多余且笨拙的。

我们考虑了使用 `@prop` 这样的装饰器来作为替代方案：

```ts
class App extends Component<Props> {
  @prop message: string
}
```

但由于这是依赖「装饰器」这样一个仍然处于 stage-2 的提案，所以在未来这有非常大的不确定性，特别是现在 TypeScript 的实现方案已经完全不是与 TC39 提案相同步的。除此之外，另外没有什么太好的办法能够将定义在 `this.$props` 上的属性类型暴露出来，
这使得难以支持 TSX。另外用户们可能还会为某些使用装饰器设置的 prop 设置默认值，但这其实并不可用。

所以说，目前还没有很好的办法利用上下文的类型标注来推导方法及其参数的类型，这意味着传递给类 `render` 函数的参数不能利用类的其他属性来做类型推导。

### 与 React Hooks 相比

基于函数的组合式 API 提供了与 React Hooks 同等级别的逻辑组合能力，但是与它还是有很大不同：组合式 API 的 `setup()` 函数只会被调用一次，这意味着使用 Vue 组合式 API 的代码会是：

- 一般来说，更符合习惯的 JavaScript 代码的直觉
- 对调用顺序并没有那么严格，因而可以因境况而更好地调整
- 不会每次重新渲染都执行，因而可以大大降低垃圾回收的压力
- 不存在内联回调函数导致子组件永远更新的问题
- 如果用户的依赖传递出现失误，“useEffect” 和 “useMemo” 可能会捕获过时的变量。Vue 的自动依赖跟踪确保观察者和计算值总是准确无误

我们敬佩 React Hooks 的创造性，它也是本提案的主要灵感来源，然而上面提到的一些其设计中的问题与缺陷，我们在 Vue 的响应式模型中恰好顺带地给解决了。

### 与 svelte 比较

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

Svelte 的代码看起来更简洁，因为它在编译时期做了以下工作:

- 隐式地将整个 `script` 块（ `import` 语句除外）包装到每个组件实例调用的函数中（并不是只执行一次）
- 隐式在变量更改时注入了响应性
- 隐式地向呈现上下文公开所有范围内的变量
- 将 `$` 语句编译为重新执行的代码

从技术上讲，我们在 Vue 中也可以通过可能的 Babel 插件做到这些，但我们不想这样做的原因是：**想要与标准 JavaScript 保持一致**。
如果您从 Vue 单文件组件的 `script` 块中提取代码，我们希望它与标准的 ES 模块完全相同。另一方面，Svelte 的 `script` 块中的代码在技术上已经不再是标准的 JavaScript 了，这种基于编译器的方法存在很多问题:

1. 在有或没有编译时，其代码的工作方式是不同。作为一个渐进的框架，许多 Vue 用户可能 希望/需要/必须 在没有任何构建配置的情况下使用，因此编译后的版本不能是默认的。另一方面，Svelte 将自己定位为一种编译器，只能搭配构建步骤一起使用。这也都是两个框架都在有意识地根据自己的目标进行权衡的结果。

2. 代码在组件内部和外部的工作方式不同。当试图从一个 Svelte 的组件中提取逻辑并将其放入标准的 JavaScript 文件时，我们将失去那看上去神奇、简洁的语法糖，而不得不退回到 [一个更冗长的底层 API](https://svelte.dev/docs#svelte_store)。

3. Svelte 的响应性编译只适用于顶层变量，它不涉及函数中声明的变量, 所以我们 [无法将响应应式状态封装在组件内声明的函数中](https://svelte.dev/repl/4b000d682c0548e79697ddffaeb757a3?version=3.6.2)。这给自带函数的代码组织带来了无可避免的约束，正如我们在这个提案中所演示的，这对于保持大型组件的可维护性非常重要。

4. [非标准语义使得它与 TypeScript 相集成十分困难](https://github.com/sveltejs/svelte/issues/1639).

这并不是说 Svelte 3 的方案是一个很坏的设计，事实上，它是一个非常创新的想法，我们非常尊重 Rich 的工作。但是基于 Vue 的设计限制和最终目标上的不同，我们必须做出不同的权衡。
