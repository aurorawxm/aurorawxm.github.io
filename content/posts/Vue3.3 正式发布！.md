---
title: "Vue3.3 正式发布！"
date: 2023-05-13T02:17:54Z
draft: false
---
<script setup> + TypeScript 开发体验改善
支持类型导入和复杂类型
以前，在 defineProps 和 defineEmits 的类型仅支持本地类型，不能使用 import，并且仅支持类型字面量和 interface。这是因为 Vue 需要能够分析 props 接口上的属性，以生成相应的运行时选项。
这个限制现在在 3.3 中得到解决。编译器现在可以解析导入的类型，并支持有限的复杂类型：
ts<script setup lang="ts">
import type { Props } from './foo'

// imported + intersection type
defineProps<Props & { extraProp?: string }>()
</script>

请注意，对复杂类型支持是基于 AST 的，因此不是 100% 全面的。一些需要实际类型分析的复杂类型，例如条件类型，不受支持。您可以使用条件类型来定义单个 prop 的类型，但不能用于整个 props 对象。

详情： PR#8083

泛型组件
使用 <script setup> 的组件现在可以通过 generic 属性接受泛型类型参数：
HTML<script setup lang="ts" generic="T">
defineProps<{
  items: T[]
  selected: T
}>()
</script>

generic 的值与 TypeScript 中 <...> 之间的参数列表用法完全相同。例如，您可以使用多个参数、extend 约束、默认类型和引用导入的类型：
html<script setup lang="ts" generic="T extends string | number, U extends Item">
import type { Item } from './types'
defineProps<{
  id: T
  list: U[]
}>()
</script>

此功能在最新版本的 volar / vue-tsc 中默认启用。

相关讨论： RFC#436
相关 PR： generic defineComponent() - PR#7963

defineEmits
以前，defineEmits 的类型参数仅支持调用签名语法：
js// 以前
const emit = defineEmits<{
  (e: 'foo', id: number): void
  (e: 'bar', name: string, ...rest: any[]): void
}>()

该类型与 emit 的返回类型匹配，但编写起来有点冗长和笨拙。3.3 引入了一种更符合人体工程学的声明带有类型的 emits 的方式：
js// 现在
const emit = defineEmits<{
  foo: [id: number]
  bar: [name: string, ...rest: any[]]
}>()

在类型字面量中，key 是事件名称，value 是事件参数的数组类型（使用标记元组元素的方式）。
以前的函数调用签名语法仍然受支持。
使用 defineSlots 设置 slots 类型
新的 defineSlots 宏可以声明 slots 及其类型：
html<script setup lang="ts">
defineSlots<{
  default?: (props: { msg: string }) => any
  item?: (props: { id: number }) => any
}>()
</script>

defineSlots()只接受一个类型参数，没有运行时参数。类型参数应该是一个类型字面量

key 是 slot 名称
value 是 slot 函数

函数的第一个参数是 slot 期望接收的 props，它的类型将用于模板中的 slot props。



defineSlots的返回值与 useSlots 返回的 slots 对象相同。
目前存在的限制：

volar / vue-tsc 尚未实现 slots 类型检查。
slot 函数的返回类型目前是忽略的，是任何类型，但我们可能会在将来利用它进行 slot 内容检查。

除了在 <script setup> 中使用 defineSlots 定义 slots 类型，还能在 defineComponent 中的 slots 属性中定义
typescriptimport { SlotsType } from 'vue'

defineComponent({
  slots: Object as SlotsType<{
    default: { foo: string; bar: number }
    item: { data: number }
  }>,
  setup(props, { slots }) {
    expectType<undefined | ((scope: { foo: string; bar: number }) => any)>(
      slots.default
    )
    expectType<undefined | ((scope: { data: number }) => any)>(slots.item)
  }
})

这两个 API 都对运行时没有影响，纯粹作为 IDE 和 vue-tsc 的类型提示。

更多详情： PR#7982

实验性功能
reactive 解构
之前是 Reactivity Transform 提案的一部分（已被废弃），现在已被拆分为单独的功能。
该功能可以解构的 props 并保持响应性，并提供了一种更符合人体工程学的方式来声明 props 的默认值：
html<script setup>
import { watchEffect } from 'vue'

const { msg = 'hello' } = defineProps(['msg'])

watchEffect(() => {
  // 在 watch 和 computed 中使用 msg
  // 能够正常收集依赖，就好像使用 props.msg
  console.log(`msg is: ${msg}`)
})
</script>

<template>{{ msg }}</template>

此功能是实验性的，需要明确的选择加入。以 Vite 为例：
js// vite.config.js
export default {
  plugins: [
    vue({
      propsDestructure: true
    })
  ]
}


详情：RFC#502

defineModel
以前组件想要支持 v-model，需要两个步骤：

声明 props
在打算更新 props 时，emit update:propName 事件

子组件支持 v-model 的写法：
html<!-- BEFORE  -->
<script setup>
const props = defineProps(['modelValue'])
const emit = defineEmits(['update:modelValue'])
console.log(props.modelValue)

function onInput(e) {
  emit('update:modelValue', e.target.value)
}
</script>

<template>
  <input :value="modelValue" @input="onInput" />
</template>

父组件：
html<script setup>
import { ref } from 'vue'
import Comp from './Comp.vue'

const msg = ref('')
</script>
<template>
  <Comp v-model="msg">
</template>

Vue 3.3 用新的 defineModel 宏简化了用法。宏将自动注册 props 和事件 ，并返回一个 ref：
html<!-- AFTER -->
<script setup>
const modelValue = defineModel()
console.log(modelValue.value)
</script>

<template>
  <input v-model="modelValue" />
</template>

根据接受 defineModel 返回值的变量名，这里是 modelValue，会自动定义 props 名为 modelValue，emit 事件为 update:modelValue
也支持显示传入 props 名称
jsconst count = defineModel<number>('count', { default: 0 })

此功能是实验性的，需要明确的选择加入。

详情： RFC#503

其他值得注意的功能
defineOptions
defineOptions 允许直接在 <script setup> 中声明组件选项，而无需单独的 <script> 块：
html<script setup>
defineOptions({ inheritAttrs: false })
</script>


终于可以快乐地设置组件 name 了

toRef、toValue 提供 getter 支持
增强 toRef，支持将 values/getters/ref 标准化为 ref：
js// 相当于 ref(1)
toRef(1)
// 创建只读 ref，使用 .value 时执行 getter
toRef(() => props.foo)
// 返回 ref
toRef(existingRef)

调用 toRef 类似于 computed，但如果 getter 没有昂贵的计算，toRef 会更高效
toValue 则相反，将 values/getters/ref 标准化为 value：
typescripttoValue(1) //       --> 1
toValue(ref(1)) //  --> 1
toValue(() => 1) // --> 1


详情：PR#7997

JSX 导入源支持
目前，Vue 的类型自动注册全局 JSX 类型。这可能会与需要 JSX 类型推断的其他库一起使用时发生冲突，特别是 React
从3.3开始，Vue 支持通过 TypeScript 的 jsxImportSource 选项指定 JSX 命名空间。这允许用户根据其需要，选择全局或每个文件的选择加入。
为了向后兼容，3.3 仍然全局注册 JSX 命名空间。我们计划在 3.4 中删除默认的全局注册。如果您正在使用 TSX 与  Vue，请在升级到 3.3后在 tsconfig.json 中添加显式的 jsxImportSource，以避免在 3.4 中出现问题。
按计划，我们的目标是在2023年开始制作较小，更频繁的功能发行版。请继续关注！

作者：candyTong
链接：https://juejin.cn/post/7231853294409531449
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
