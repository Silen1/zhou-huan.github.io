---
layout: post
title: "Vue 自定义指令 (custom directive)"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - Vue.js学习
---

尽管 Vue 推崇数据驱动，但有时仍然需要对 DOM 进行底层操作，这时就能用到自定义指令，自定义指令是可复用的。

## 自动聚焦的例子

    <template>
        <div>
            <input type="text" placeholder="姓名">
            <br>
            <input type="text" placeholder="电话" v-focus>
        </div>
    </template>

    <script>
        export default {
            directives: {
                focus: {
                    inserted: function (el) {
                        el.focus()
                    }
                }
            }
        }
    </script>

![auto-focus](/img/article/auto-focus.png)

使用了 `v-focus` 属性的 `input` 元素在插入父节点时会自动聚焦。

`directives` 选项中的 `focus` 是指令的名字， `focus` 的值是指令的定义。

`inserted` 是自定义指令的一个钩子函数，该钩子函数在绑定指令的元素插入父节点时调用 (仅保证父节点存在，但不一定已被插入文档中)。自定义指令还有其它钩子函数：

- `bind`：只调用一次，指令第一次绑定到元素时调用。在这里可以进行一次性的初始化设置

- `update`：所在组件的 VNode（Vue 编译生成的虚拟节点） 更新时调用，但是可能发生在其子 VNode 更新之前。指令的值可能发生了改变，也可能没有。但是你可以通过比较更新前后的值来忽略不必要的模板更新

- `componentUpdated`：指令所在组件的 VNode 及其子 VNode 全部更新后调用

- `unbind`：只调用一次，指令与元素解绑时调用

`el` 是钩子函数的参数，表示指令所绑定的元素，可以用来直接操作 DOM 。

本例在 `directives` 选项中注册了一个局部自定义指令，也可以像下面这样注册全局指令：

    Vue.directive('focus', {
        inserted: function (el) {
            el.focus()
        }
    })

注册之后就可以在任何地方的模板里给元素添加 `v-focus` 属性。

## 图片加载的例子

    <template>
        <div>
            <img v-load="url" alt="">
        </div>
    </template>

    <script>
        export default {
            data () {
                return {
                    url: 'http://i3.stycdn.net/images/2013/12/52/article/so11/so11z01001/jordan-air-jordan-3-retro-schuhe-weiss-rot-1150-zoom-0.jpg'
                }
            },
            directives: {
                load: {
                    inserted: function (el, binding) {
                        el.src = '/static/img/smallpotato.jpg'
                        var image = new Image()
                        image.src = binding.value
                        image.onload = _ => {
                            el.src = binding.value
                        }
                    }
                }
            }
        }
    </script>

    <style scoped lang="scss">
        div {
            display: flex;
            justify-content: space-around;
        }
        img {
            width: 300px;
            height: 260px;
        }
    </style>

![load-img](/img/article/load-img.gif)

添加了 `v-load` 属性的 `img` 元素在插入父节点时首先加载了 `smallpotato.jpg`，等传入的 `src` 加载好时又会换成了传入的图片。

`binding` 是自定义指令钩子函数的一个参数（**只读**），该参数是一个对象，`value` 又是该对象的一个属性，表示指令的绑定值，比如 `v-load="url"` 中绑定值为 `url`。该对象还有其他属性：

- `name`：指令名，不包括 v- 前缀

- `oldValue`：指令绑定的前一个值，仅在 `update` 和 `componentUpdated` 钩子中可用，无论值是否改变都可用

- `expression`：字符串形式的指令表达式。例如 `v-my-directive="1 + 1"` 中，表达式为 `"1 + 1"`

- `arg`：传给指令的参数，可选。例如 `v-my-directive:foo` 中，参数为 `"foo"`

- `modifiers`：一个包含修饰符的对象。例如：`v-my-directive.foo.bar` 中，修饰符对象为 `{ foo: true, bar: true }`

`smallpotato.jpg` 是预置的一张小图，加载较快，传入的图片较大时可以这样填补加载图片时的空白期。也可以做成百度图片加载时的效果，元素换成 div, li 之类的容器，然后取一个随机色填充为背景色，等图片完全加载时再将图片设置为背景即可。