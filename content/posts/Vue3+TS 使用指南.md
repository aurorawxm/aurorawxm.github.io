---
title: "Vue3+TS 使用指南"
date: 2023-05-12T03:03:06Z
draft: false
---

vue3相比vue2对ts拥有更友好的支持，当初在用vue2写ts时各种装饰器，现在只要使用<script setup lang="ts"></script>标签就可以直接使用ts。ts虽然写的时候麻烦了点，但是真的香。
如果不知道怎么搭建vue3的ts项目，可参考我的上一篇文章：Vite4.3+Typescript+Vue3+Pinia 最新搭建企业级前端项目
1. ref
传入一个泛型参数

值 ref

jsconst initCode = ref('200'); //默认推导出string类型

//或者手动定义更复杂的类型
const initCode = ref<string | number>('200');


模板 ref

js<template>
  <div ref="el"></div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
const el = ref<HTMLImageElement | null>(null);
</script>



组件 ref

js<template>
  <HelloWorld ref="helloworld" />
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue';
import HelloWorld from '@/components/HelloWorld.vue';

const helloworld = ref<InstanceType<typeof HelloWorld> | null>(null);

onMounted(() => {
  //调用子组件的handleClick方法
  helloworld.value?.handleClick();
});
</script>

如果子组件使用<script setup lang="ts">，默认是全关闭的，子组件需使用defineExpose定义父组件能访问的属性
2. reactive

定义接口

新建src/types/user.ts（在types文件夹下新建user.ts）
jsexport interface User {
  name: string;
  age: number;
}


使用

js<script setup lang="ts">
import { reactive } from 'vue';
import type { User } from '@/types/user';

//reactive 会隐式地从它的参数中推导类型
//也可以使用接口直接给变量定义类型
const user: User = reactive({
  name: 'zhangsan',
  age: 20,
});
</script>

提示：不推荐使用 reactive() 的泛型参数，因为处理了深层次 ref 解包的返回值与泛型参数的类型不同
3. computed
computed() 会自动从其计算函数的返回值上推导出类型
也可以通过泛型参数显式指定类型
jsconst age = computed<number>(() => {
  return 2;
});

4. defineProps（父传子）
定义props类型，直接使用，不需要import引入
1. 运行时声明
类型作为参数的一部分传入
1. 基本使用
运行时声明的类型共八大类：String，Number，Boolean，Array，Object，Date，Function，Symbol（注意这里和ts的类型有区别，例如ts中字符串类型为string，这里是String）
如果想为引用类型定义更具体的类型，需使用 PropType
js<template>
  <div>
    <div>{{ msg }}</div>
    <div>{{ address }}</div>
    <div>{{ user.name }}--{{ user.age }}</div>
  </div>
</template>

<script setup lang="ts">
import { PropType } from 'vue';

interface User {
  name: string;
  age: number;
}

const props = defineProps({
  msg: {
    type: String,
    default: 'hello',
  },
  address: {
    type: [String, Number], //联合类型，类似于ts的 |
    default: 2,
  },
  user: {
    type: Object as PropType<User>, //定义更具体的类型
    default: () => ({
      name: 'zhangsan',
      age: 12,
    }),
  },
});

console.log(props.msg);

console.log(props.user.name);
</script>


2. 接口User由外部传入
js<script setup lang="ts">
import type { User } from '@/types/user';

//将原先定义的接口注释掉
// interface User {
//   name: string;
//   age: number;
// }
</script>

2. 类型声明（使用泛型）
1. 基本使用
使用泛型，不需要PropType定义更具体的类型
js<template>
  <div>{{ name }}</div>
</template>

<script setup lang="ts">
interface User {
  name?: string;
  age?: number;
}

const props = defineProps<User>();

console.log(props.name);
</script>


2. 定义默认值
使用泛型，需要使用 withDefaults 定义默认值
js<script setup lang="ts">
interface User {
  name: string;
  age: number;
}

const props = withDefaults(defineProps<User>(), {
  name: 'hello',
  age: 10,
});
</script>

3. 局限性
不能使用外部传入的类型（未来版本中可能会解决这个限制）
jsimport type { User } from '@/types/user';

//将原先定义的接口注释掉
// interface User {
//   name: string;
//   age: number;
// }

报错

5. defineEmits（子传父）
定义emits类型 ，直接使用，不需要import引入
父组件
js<template>
  <div>
    <HelloWorld @change="handleChange" />
  </div>
</template>

<script setup lang="ts">
import HelloWorld from '@/components/HelloWorld.vue';
import type { User } from '@/types/user';
const handleChange = (value: User) => {
  console.log(value);
};
</script>

1. 运行时声明
js<!-- 子组件 -->
<template>
  <div>
    <button @click="handleClick">按钮</button>
  </div>
</template>

<script setup lang="ts">
const emit = defineEmits(['change']);

const handleClick = () => {
  emit('change', { name: '2', age: 21 });
};
</script>

2. 类型声明（使用泛型）
可以对所触发事件的类型进行更细粒度的控制。并且可以使用外部传入的类型
js<!-- 子组件 -->
<template>
  <div>
    <button @click="handleClick">按钮</button>
  </div>
</template>

<script setup lang="ts">
import type { User } from '@/types/user';
const emit = defineEmits<{ (e: 'change', value: User): void }>();

const handleClick = () => {
  emit('change', { name: '2', age: 21 });
};
</script>

6. defineExpose（父调用子）
定义父组件通过模板 ref 能获取到的属性
接着上面组件ref案例修改子组件HelloWorld
js<template>
  <div></div>
</template>

<script setup lang="ts">
const handleClick = () => {
  console.log('子组件方法');
};
defineExpose({ handleClick });
</script>

7. provide / inject（跨组件传值）
1. key是Symbol
新建src/constant/index.ts
jsimport type { InjectionKey } from 'vue';

export const key = Symbol() as InjectionKey<string>;

js<!-- 父组件使用provide提供值 -->
<script setup lang="ts">
import { provide } from 'vue';
import { key } from '@/constant/index';
provide(key, '123'); //提供改变响应式对象的方法
</script>

<!-- 子组件使用inject取值 -->
<script setup lang="ts">
import { inject } from 'vue';
import { key } from '@/constant/index';

const string = inject(key);
</script>

2. key是字符串
inject返回的类型是 unknown，需要通过泛型参数显式声明
js<!-- 父组件提供provide -->
<script setup lang="ts">
import { ref, provide } from 'vue';
const state = ref(0);
const handlerState = () => {
  state.value = 1;
};
provide('info', state); //提供响应式对象
provide('func', handlerState); //提供改变响应式对象的方法
</script>

<!-- 子组件使用inject取值 -->
<script setup lang="ts">
import { inject } from 'vue';

//通过泛型参数显式声明
const state = inject<number>('info');
const func = inject<() => void>('func');
</script>


3. undefined问题
由于无法保证provide会提供这个值，因此inject通过泛型参数显示声明了类型，还会多个undefined类型

提供默认值，可消除undefined

jsconst state = inject<number>('info', 20);


使用类型断言，告诉编辑器这个值一定会提供

jsconst state = inject('info') as number;

8. 事件类型
1. input change事件
js<template>
  <input type="text" @change="handleChange" />
</template>

<script setup lang="ts">
const handleChange = (evt: Event) => {
  console.log((evt.target as HTMLInputElement).value);
};
</script>

2. button Click事件
js<template>
  <button @click="handleClick">按钮</button>
</template>

<script setup lang="ts">
const handleClick = (evt: Event) => {
  //获取按钮的样式信息
  console.log((evt.target as HTMLButtonElement).style);
};
</script>

3. HTML标签映射关系
jsinterface HTMLElementTagNameMap {
  "a": HTMLAnchorElement;
  "abbr": HTMLElement;
  "address": HTMLElement;
  "applet": HTMLAppletElement;
  "area": HTMLAreaElement;
  "article": HTMLElement;
  "aside": HTMLElement;
  "audio": HTMLAudioElement;
  "b": HTMLElement;
  "base": HTMLBaseElement;
  "basefont": HTMLBaseFontElement;
  "bdi": HTMLElement;
  "bdo": HTMLElement;
  "blockquote": HTMLQuoteElement;
  "body": HTMLBodyElement;
  "br": HTMLBRElement;
  "button": HTMLButtonElement;
  "canvas": HTMLCanvasElement;
  "caption": HTMLTableCaptionElement;
  "cite": HTMLElement;
  "code": HTMLElement;
  "col": HTMLTableColElement;
  "colgroup": HTMLTableColElement;
  "data": HTMLDataElement;
  "datalist": HTMLDataListElement;
  "dd": HTMLElement;
  "del": HTMLModElement;
  "details": HTMLDetailsElement;
  "dfn": HTMLElement;
  "dialog": HTMLDialogElement;
  "dir": HTMLDirectoryElement;
  "div": HTMLDivElement;
  "dl": HTMLDListElement;
  "dt": HTMLElement;
  "em": HTMLElement;
  "embed": HTMLEmbedElement;
  "fieldset": HTMLFieldSetElement;
  "figcaption": HTMLElement;
  "figure": HTMLElement;
  "font": HTMLFontElement;
  "footer": HTMLElement;
  "form": HTMLFormElement;
  "frame": HTMLFrameElement;
  "frameset": HTMLFrameSetElement;
  "h1": HTMLHeadingElement;
  "h2": HTMLHeadingElement;
  "h3": HTMLHeadingElement;
  "h4": HTMLHeadingElement;
  "h5": HTMLHeadingElement;
  "h6": HTMLHeadingElement;
  "head": HTMLHeadElement;
  "header": HTMLElement;
  "hgroup": HTMLElement;
  "hr": HTMLHRElement;
  "html": HTMLHtmlElement;
  "i": HTMLElement;
  "iframe": HTMLIFrameElement;
  "img": HTMLImageElement;
  "input": HTMLInputElement;
  "ins": HTMLModElement;
  "kbd": HTMLElement;
  "label": HTMLLabelElement;
  "legend": HTMLLegendElement;
  "li": HTMLLIElement;
  "link": HTMLLinkElement;
  "main": HTMLElement;
  "map": HTMLMapElement;
  "mark": HTMLElement;
  "marquee": HTMLMarqueeElement;
  "menu": HTMLMenuElement;
  "meta": HTMLMetaElement;
  "meter": HTMLMeterElement;
  "nav": HTMLElement;
  "noscript": HTMLElement;
  "object": HTMLObjectElement;
  "ol": HTMLOListElement;
  "optgroup": HTMLOptGroupElement;
  "option": HTMLOptionElement;
  "output": HTMLOutputElement;
  "p": HTMLParagraphElement;
  "param": HTMLParamElement;
  "picture": HTMLPictureElement;
  "pre": HTMLPreElement;
  "progress": HTMLProgressElement;
  "q": HTMLQuoteElement;
  "rp": HTMLElement;
  "rt": HTMLElement;
  "ruby": HTMLElement;
  "s": HTMLElement;
  "samp": HTMLElement;
  "script": HTMLScriptElement;
  "section": HTMLElement;
  "select": HTMLSelectElement;
  "slot": HTMLSlotElement;
  "small": HTMLElement;
  "source": HTMLSourceElement;
  "span": HTMLSpanElement;
  "strong": HTMLElement;
  "style": HTMLStyleElement;
  "sub": HTMLElement;
  "summary": HTMLElement;
  "sup": HTMLElement;
  "table": HTMLTableElement;
  "tbody": HTMLTableSectionElement;
  "td": HTMLTableDataCellElement;
  "template": HTMLTemplateElement;
  "textarea": HTMLTextAreaElement;
  "tfoot": HTMLTableSectionElement;
  "th": HTMLTableHeaderCellElement;
  "thead": HTMLTableSectionElement;
  "time": HTMLTimeElement;
  "title": HTMLTitleElement;
  "tr": HTMLTableRowElement;
  "track": HTMLTrackElement;
  "u": HTMLElement;
  "ul": HTMLUListElement;
  "var": HTMLElement;
  "video": HTMLVideoElement;
  "wbr": HTMLElement;
}

9. 结尾
vue的ts分为运行时声明和类型声明，运行时声明是基于vue2延续并改进的，类型声明是基于ts的泛型实现的。个人更喜欢类型声明，更符合ts的书写规范，希望尤大早日将defineProps类型声明不能使用外部传入的类型问题早日解决
最后：本篇整理的内容基于vue3.2.47，vue3.3.0已经走的了alpha.8阶段。点赞多的话后续随着vue的升级继续更新本篇文章

作者：敲代码的彭于晏
链接：https://juejin.cn/post/7231200657902207013
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

