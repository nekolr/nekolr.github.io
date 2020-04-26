---
title: Vue 之在子组件中改变父组件的状态
date: 2020/4/26 11:51:0
tags: [Vue.js]
categories: [Vue.js]
---

一个 Vue 应用是由一个通过 new Vue 创建的根实例和可选的嵌套的、可复用的组件树组成。比如，一个应用可能是这样的：

<!--more-->

```
根实例
└─ TodoList
   ├─ TodoItem
   │  ├─ DeleteTodoButton
   │  └─ EditTodoButton
   └─ TodoListFooter
      ├─ ClearTodosButton
      └─ TodoListStatistics
```

在这里，组件树节点之间形成了上下级的关系，也就是存在父组件与子组件这样的关系。如果我们想将父组件中的某些状态（数据）传递到子组件中，那么可以使用 `v-bind` 指令（或者 `:` 缩写）将父组件中的值绑定到子组件上，子组件使用 props 属性来接收这些传过来的值。比如：

```vue
<template>
  <Child :foo="foo" />
</template>

<script>
import Child from './Child'

export default {
  name: 'Parent',
  components: {
    Child
  },
  data() {
    return {
      foo: 'hello'
    }
  }
}
</script>
```

```vue
<template>
  <p>I am a child component.{{ foo }}</p>
</template>

<script>
export default {
  name: 'Child',
  props: ['foo'],
  data() {
    return {}
  }
}
</script>
```

需要注意的是，父组件与子组件之间的数据是**单向下行绑定的**，即父组件数据的更新会自动向下流动到子组件中，但是反过来则不行。这样可以防止子组件意外变更父组件的状态，从而导致应用的数据流向难以理解。同时，每次父组件发生变更时，子组件中所有的 prop 都会刷新为最新的值，这意味着我们不应该在一个子组件中直接修改 prop，如果直接修改，Vue 也会在浏览器的控制台中发出警告。

```
[Vue warn]: Avoid mutating a prop directly since the value will be overwritten whenever the parent component re-renders. 
Instead, use a data or computed property based on the prop's value ...
```

在 Vue 官网中给出了两种常见的试图变更一个 prop 的情形。其中一种情况是，porp 传递的是一个初始值，这个子组件在接下来希望将其作为一个本地的 prop 数据来使用，这时最好是定义一个本地的 data 属性并将这个 prop 用作其初始值，比如：

```js
props: ['initialCounter'],
data: function () {
  return {
    counter: this.initialCounter
  }
}
```

另外一种情况是，prop 以一种原始的值传入且需要进行转换，这时最好使用这个 prop 的值来定义一个计算属性，比如：

```js
props: ['size'],
computed: {
  normalizedSize: function () {
    return this.size.trim().toLowerCase()
  }
}
```

同时，官网也给出了一个警告信息，这个信息很重要，笔者之前就在这上面狠狠摔了一跤。官网写道：

> 注意在 JavaScript 中对象和数组是通过引用传入的，所以对于一个数组或对象类型的 prop 来说，在子组件中变更这个对象或数组本身将会影响到父组件的状态。

这句话的意思是说，如果父组件传递给子组件的值是一个对象或数组，那么如果在子组件中直接修改这个对象的属性或者数组的元素，父组件中的状态也会跟着变化。比如：

```vue
<template>
  <div>
    我是父组件的值：{{ foo }}
    <Child :foo="foo" />
  </div>
</template>

<script>
import Child from './Child'

export default {
  name: 'Parent',
  components: {
    Child
  },
  data() {
    return {
      foo: {
        bar: 'hello'
      }
    }
  }
}
</script>
```

```vue
<template>
  <div>
    <p>我是子组件中的值：{{ foo }}</p>
    <input v-model="foo.bar">
  </div>
</template>

<script>
export default {
  name: 'Child',
  props: ['foo'],
  data() {
    return {}
  }
}
</script>
```

当然这种直接修改父组件值的方式也是不推荐使用的，因为这种方式直接破坏了我们单向数据流的设定，容易造成数据流向难以理解的问题。一种更清晰的方式是：在子组件想要修改父组件的状态时，通过事件通知的方式告诉父组件我要修改哪个值，以及新值是多少，在父组件中需要预先定义好事件触发时对应触发的修改值的方法。比如：

```vue
<template>
  <div>
    我是父组件的值：{{ foo }}
    <Child :foo="foo" v-on:update:bar="updateBar" />
  </div>
</template>

<script>
import Child from './Child'

export default {
  name: 'Parent',
  components: {
    Child
  },
  data() {
    return {
      foo: {
        bar: 'hello'
      }
    }
  },
  methods: {
    // 父组件提供的修改 bar 值的方法
    updateBar(val) {
      this.foo.bar = val
    }
  }
}
</script>
```

```vue
<template>
  <div>
    <p>我是子组件中的值：{{ foo }}</p>
    <input v-model="obj.bar">
  </div>
</template>

<script>
export default {
  name: 'Child',
  props: ['foo'],
  data() {
    return {
      obj: this.foo
    }
  },
  watch: {
    'obj.bar': function(val) {
      // 监听 bar 值的变动，发生变动时触发当前实例上的 update:bar 事件
      // 将新值传给事件监听器回调
      this.$emit('update:bar', val)
    }
  }
}
</script>
```

当然这里还有更简便的写法，就是直接将更新的逻辑写到绑定事件上，同时使用 `v-on` 的语法糖简化写法。

```vue
<template>
  <div>
    我是父组件的值：{{ foo }}
    <Child :foo="foo" @update:bar="foo.bar = $event" />
  </div>
</template>

<script>
import Child from './Child'

export default {
  name: 'Parent',
  components: {
    Child
  },
  data() {
    return {
      foo: {
        bar: 'hello'
      }
    }
  }
}
</script>
```

这种父子组件进行“双向绑定”的模式比较固定，因此在 vue 2.3.0 就专门给这种模式新增了一个语法糖，即 `.sync` 修饰符。

```vue
<template>
  <div>
    我是父组件的值：{{ foo }}
    <Child :foo.sync="foo" />
  </div>
</template>

<script>
import Child from './Child'

export default {
  name: 'Parent',
  components: {
    Child
  },
  data() {
    return {
      foo: {
        bar: 'hello'
      }
    }
  }
}
</script>
```

这种写法可以在我们用一个对象给子组件同时设置多个 prop 的时候，把对象中的每一个属性（比如这里的 bar 属性）都作为独立的 prop 传进去，然后各自添加用于更新的 `v-on` 监听器。
