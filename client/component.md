# <component>

Vue([官网地址](https://cn.vuejs.org/v2/api/#component)) 里面提供了内置组件  `component`，语法如下：

```html
<!-- 动态组件由 vm 实例的属性值 `componentId` 控制 -->
<component :is="componentId"></component>

<!-- 也能够渲染注册过的组件或 prop 传入的组件 -->
<component :is="$options.components.child"></component>
```

这个内置的组件通过 `is` 来决定到底渲染哪一个组件。看下面一个场景：

```js
vm = new Vue({
  template: '<component :is="view"></component>',
  data: {
    view: 'viewA'
  },
  components: {
    'viewA': {
      name: 'jz',
      template: '<div>foo {{view}}</div>',
      data () {
        return { view: 'a' }
      }
    },
    'view-b': {
      template: '<div>bar {{view}}</div>',
      data () {
        return { view: 'b' }
      }
    }
  }
}).$mount()
document.body.append(vm.$el)
```

真实渲染之后的 DOM 元素是 `<div>foo a</div>`，因为 `is` 接收 `string | ComponentDefinition | ComponentConstructor`。

其实你会发现上面的 rootVm 的 template 编译之后变成如下的 render 函数：

```js
(function anonymous() {
    with(this) {
        return _c(view, {
            tag: "component"
        })
    }
})
```

因为处在 with 的语法当中，所以 _c(view) 就是 this.view，它的值是 `'viewA'`，这样的话就会创建一个 viewA 的组件 VNode，所以这样与你写 `<view-a></view-a>` 没啥区别。如果改变 view 属性的值，就会切换到另外一个组件，所有的一切是因为 `component` 内置组件么？

**不是！**

如果你修改为如下的代码，你会发现效果一模一样。

```js
vm = new Vue({
  template: '<div :is="view"></div>',
  data: {
    view: 'viewA'
  },
  components: {
    // ......
  }
}).$mount()
```

把 component 组件改成 div 了，生成的 render 函数是

```js
(function anonymous() {
    with(this) {
        return _c(view, {
            tag: "div"
        })
    }
})
```

VNodeData 里面的 tag 属性变了而已，这个属性一般用在提示或者报错里面，真实的渲染 `view-a` 组件还是依赖 `_c(view, /../)`，所以这一切都是在 compile 的时候依据 is 这个 attribute 来决定的，跟是不是 component 组件没有关系。

很有趣是吧？

component 还接收 inline-template，用来改变将包裹的模版转化为 render 函数。

```js
window.vm = new Vue({
  template: '<component :is="view" inline-template><em>{{view}}</em></component>',
  data: {
    view: 'viewA'
  },
  components: {
    'viewA': {
      name: 'jz',
      template: '<div>foo {{view}}</div>',
      data () {
        return { view: 'a test' }
      }
    },
    'view-b': {
      template: '<div>bar {{view}}</div>',
      data () {
        return { view: 'b' }
      }
    }
  }
}).$mount()
document.body.append(vm.$el)
```

渲染的是 `<em>a test</em>`，也就是说 em 标签都成为了子组件 `view-a` 的模版，根据这个模版得到 render 函数，并且作用域也是 `view-a` 组件，而不是根组件。