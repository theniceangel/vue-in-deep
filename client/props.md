# props

props 用的很多，但是一直对它的实现细节知之甚少。[官网](https://cn.vuejs.org/v2/guide/components-props.html#Prop-%E7%B1%BB%E5%9E%8B)给了很详细的使用介绍，但是我想知道 props 是怎么做到的，以及为啥子组件不能修改父组件的传下来的 props，父组件更新的同时，是怎么更新传下去的子组件 props 的。

## 如何实现 props 功能

首先在 Vue 的组件系统的架构里，每一个子组件的实例化，都是依赖父组件里面引用了子组件，在父组件 `_render` 的时候会生成子组件 VNode，在其 patch 的过程中，发现如果是子组件 VNode，那么会走到子组件的实例化，在子组件 VNode 的 data 字段里面会有父组件传下去的 props 对象，举个例子：

```html
<child gender="man"></child>
```

那么 child 这个组件 VNode 的 data 就是这样的：

```js
with(this) {
    return _c('child', {
        attrs: {
            "gender": "man"
        }
    })
}
```

在生成 child 这个组件 VNode 的时候（`src/core/vdom/create-component.js`），会先拿到 child 组件选项，调用 Vue.extend，将选项中的 props 进行 normalize，例如 child 组件 props 的配置如下：

```js
{
  props: ['gender']
}
```

normalize 之后就变成:

```js
{
  props: {
    gender: {
      type: null
    }
  }
}
```

接着调用 `propsData = extractPropsFromVNodeData(data, Ctor, tag)`，将 VNode 的 data 种的 attrs 与 normalize 后的选项 props 对比之后摘出来，然后将 `propsData`(**称之为外部注入的数据**) 放在组件 VNode 的 `componentOptions` 属性上。接着实例化 child 组件的时候，在其 `_init` 函数里面会走到 `initInternalComponent`。

```js
export function initInternalComponent (vm: Component, options: InternalComponentOptions) {
  const opts = vm.$options = Object.create(vm.constructor.options)
  // doing this because it's faster than dynamic enumeration.
  const parentVnode = options._parentVnode
  opts.parent = options.parent
  opts._parentVnode = parentVnode

  // 拿到通过 `extractPropsFromVNodeData` 之后的 props 数据
  const vnodeComponentOptions = parentVnode.componentOptions
  opts.propsData = vnodeComponentOptions.propsData


  if (options.render) {
    opts.render = options.render
    opts.staticRenderFns = options.staticRenderFns
  }
}
```

也就是子组件的 `$options.propsData` 放的是父组件注入的数据。在 `initState`（src/core/instance/state.js）的时候，会调用 `initProps`。

```js
function initProps (vm: Component, propsOptions: Object) {
  const propsData = vm.$options.propsData || {}
  const props = vm._props = {}
  // cache prop keys so that future props updates can iterate using Array
  // instead of dynamic object key enumeration.
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent
  // root instance props should be converted
  if (!isRoot) {
    toggleObserving(false)
  }
  for (const key in propsOptions) {
    keys.push(key)
    const value = validateProp(key, propsOptions, propsData, vm)
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      const hyphenatedKey = hyphenate(key)
      if (isReservedAttribute(hyphenatedKey) ||
          config.isReservedAttr(hyphenatedKey)) {
        warn(
          `"${hyphenatedKey}" is a reserved attribute and cannot be used as component prop.`,
          vm
        )
      }
      defineReactive(props, key, value, () => {
        if (!isRoot && !isUpdatingChildComponent) {
          warn(
            `Avoid mutating a prop directly since the value will be ` +
            `overwritten whenever the parent component re-renders. ` +
            `Instead, use a data or computed property based on the prop's ` +
            `value. Prop being mutated: "${key}"`,
            vm
          )
        }
      })
    } else {
      defineReactive(props, key, value)
    }
    // static props are already proxied on the component's prototype
    // during Vue.extend(). We only need to proxy props defined at
    // instantiation here.
    if (!(key in vm)) {
      proxy(vm, `_props`, key)
    }
  }
  toggleObserving(true)
}
```

`validateProp` 内部会拿 `$options.propsData`(**父组件提供的数据**) 与 `propsOptions`(子组件的 props 选项) 做一层校验。最后调用 `defineReactive` 将 `vm._props` 做成响应式，这样子组件的 props 功能就完成了。

整体流程就是通过外部注入的 `propsData` 与自己配置的 `propsOptions` 做比较，最后将子组件的 `_props` 属性做成响应式。

## 如何监控子组件修改 props 呢。

```js
defineReactive(props, key, value, () => {
  if (!isRoot && !isUpdatingChildComponent) {
    warn(
      `Avoid mutating a prop directly since the value will be ` +
      `overwritten whenever the parent component re-renders. ` +
      `Instead, use a data or computed property based on the prop's ` +
      `value. Prop being mutated: "${key}"`,
      vm
    )
  }
})
```

`defineProperty` 提供了 `customerSetter` 内部判断 `if (!isRoot && !isUpdatingChildComponent)`，第一个是非根组件，根组件是通过 new 生成的，子组件是在根组件 patch 过程中，发现是子组件 VNode 之后再实例化的，这是两者的区别，因为根组件没有外部注入的 `propsData`，所以[官网](https://cn.vuejs.org/v2/api/#propsData)也是给了 `propsData` 配置项去测试。而 `isUpdatingChildComponent` 是 `src/core/instance/lifecycle.js` 的一个开关变量，默认是为 false 的，所以你如果在子组件触发了 `props` 的 setter，那么 Vue 会给出警告。

## 父组件更新，子组件更新 props

那么 `isUpdatingChildComponent` 什么时候为 true 呢。

在父组件更新的时候，会重新走到 `patch`，而这个时候由于 prevChildVNode 以及现在的 ChildVNode 会做 `patchVnode` 的逻辑，内部会执行专属于组件 VNode 的 `prepatch` 钩子，定义在 `src/core/vdom/create-component.js`

```js
const componentVNodeHooks = {
  prepatch (oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
    const options = vnode.componentOptions
    const child = vnode.componentInstance = oldVnode.componentInstance
    updateChildComponent(
      child,
      options.propsData, // updated props
      options.listeners, // updated listeners
      vnode, // new parent vnode
      options.children // new children
    )
  }
}
```

内部调用 `updateChildComponent`,

```js
export function updateChildComponent (
  vm: Component,
  propsData: ?Object,
  listeners: ?Object,
  parentVnode: MountedComponentVNode,
  renderChildren: ?Array<VNode>
) {
  if (process.env.NODE_ENV !== 'production') {
    isUpdatingChildComponent = true
  }
  // ......
  // update props
  if (propsData && vm.$options.props) {
    toggleObserving(false)
    const props = vm._props
    const propKeys = vm.$options._propKeys || []
    for (let i = 0; i < propKeys.length; i++) {
      const key = propKeys[i]
      const propOptions: any = vm.$options.props // wtf flow?
      props[key] = validateProp(key, propOptions, propsData, vm)
    }
    toggleObserving(true)
    // keep a copy of raw propsData
    vm.$options.propsData = propsData
  }

  if (process.env.NODE_ENV !== 'production') {
    isUpdatingChildComponent = false
  }
}
```

`isUpdatingChildComponent` 开关会被置为 true，这样就不会触发上述的 `customerSetter`，就不会给出警告。

```js
props[key] = validateProp(key, propOptions, propsData, vm)
```

由于 `props` 已经是响应式的，所以这一段代码会触发 setter，如果前后两次值发生变化，驱使子组件的重新渲染。

