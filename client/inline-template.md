# inline-template

[Vue](https://cn.vuejs.org/v2/guide/components-edge-cases.html#%E5%86%85%E8%81%94%E6%A8%A1%E6%9D%BF) 官网有这么一句话：

> 当 `inline-template` 这个特殊的特性出现在一个子组件上时，这个组件将会使用其里面的内容作为模板，而不是将其作为被分发的内容。这使得模板的撰写工作更加灵活。

并且列举了一个例子。

```js
<my-component inline-template>
  <div>
    <p>These are compiled as the component's own template.</p>
    <p>Not parent's transclusion content.</p>
  </div>
</my-component>
```

这个例子等于白给，看不太懂，那么便从源码的角度来看吧。例如下面的场景：

```js
const vm = new Vue({
  template: '<div><test inline-template><span>{{a}}</span></test></div>',
  data: {
    a: 'parent'
  },
  components: {
    test: {
      data () {
        return { a: 'child' }
      }
    }
  }
}).$mount()
document.body.append(vm.$el)
```

最后渲染出来的 DOM 结构是 `<div><span>child</span></div>`，也就是说，`{{a}}` 取得的属性是子组件，而不是我们想当然的父组件的。那么是怎么做到的呢。

父组件 Template 编译的时候，得到的 `render` 函数是这样的：

```js
(function anonymous() {
    with(this) {
        return _c('div', [_c('test', {
            inlineTemplate: {
                render: function () {
                    with(this) {
                        return _c('span', [_v(_s(a))])
                    }
                },
                staticRenderFns: []
            }
        })], 1)
    }
})
```

test 这个组件 VNode 的 data 上有个 `inlineTemplate` 属性，在 `inlineTemplate` 上同时也有 render 和 staticRenderFns 两个方法，这样对 a 属性求值的时候，上下文就已经切换到了子 vm 上了。在父组件得到 VNode 的时候调用 `_update` 进行 VNode patch 的时候，会走到 `createComponentInstanceForVnode` 来创建 test 这个子 vm。

```js
export function createComponentInstanceForVnode (
  vnode: any, // we know it's MountedComponentVNode but flow doesn't
  parent: any, // activeInstance in lifecycle state
): Component {
  const options: InternalComponentOptions = {
    _isComponent: true,
    _parentVnode: vnode,
    parent
  }

  const inlineTemplate = vnode.data.inlineTemplate
  if (isDef(inlineTemplate)) {
    options.render = inlineTemplate.render
    options.staticRenderFns = inlineTemplate.staticRenderFns
  }
  return new vnode.componentOptions.Ctor(options)
}
```

在实例化 test vm 的时候，会拿到其 VNode.data.inlineTemplate.render 作为 test vm 的 render 函数，所以最后 `{{a}}` 求值的过程是发生在 test vm 这个上下文的环境，因此渲染的就是 `<div><span>child</span></div>`。

说白了，拥有这些能力，是在 Vue 对 Template 做 Compile 的过程中实现的，因为需要把 test 组件包裹的 DOM 元素 codegen 成 inlineTemplate 对象，并且把对 `a` 属性的求值包裹在 render 函数内部，这样才能切换 render 函数的当前 vm 上下文。就如同上面的结构。

```js
(function anonymous() {
    with(this) {
        return _c('div', [_c('test', {
            inlineTemplate: {
                render: function () {
                    with(this) {
                        return _c('span', [_v(_s(a))])
                    }
                },
                staticRenderFns: []
            }
        })], 1)
    }
})
```