# Vue.prototype._update

谈谈 _update 内部比较疑惑的地方。

## 1.更新父组件的 $el

```js
Vue.prototype._update = function (vnode, hydrating) {
  // ......
  // Vue.prototype.__patch__ is injected in entry points
  // based on the rendering backend used.
  if (!prevVnode) {
    // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */);
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode);
  }
  restoreActiveInstance();
  // ......
  // if parent is an HOC, update its $el as well
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el;
  }
  // updated hook is called by the scheduler to ensure that children are
  // updated in a parent's updated hook.
};
```

这一块之前不能理解，既然 `vm.__patch__()` 会返回 VNode.elm，所以当前 vm 的 `$el` 一定是根结点 VNode 返回的 elm。那么 ` if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode)` 这一段又是为什么呢？多此一举啊。

其实一般情况下，就算没有这段代码，是不会有什么问题的。但是考虑到下面的一个场景

```js
new Vue({
  data: {
    color: 'red'
  },
  template: '<test id="foo" :class="color"></test>',
  components: {
    test: {
      data () {
        return { tag: 'div' }
      },
      render (h) {
        return h(this.tag, { class: 'test' }, 'hi')
      }
    }
  }
}).$mount()
// 修改 test 组件的标签名
vm.$children[0].tag = 'p'
```

rootVm(通过 new 创建的) 的根节点 VNode 是 test 这个子组件 VNode，在执行 rootVm 的 _update 逻辑里面，执行 `vm.$el = vm.__patch__()`，能拿到 test 这个子组件 VNode 的 elm，所以就算没有上面所说的那个 ` if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode)` 这行代码，rootVm 的 $el 依然能够拿到 test 组件 VNode 的 elm，但是这只是初始化的过程，那么如果是数据更新的过程呢，就如上面的这行代码：

```js
// 修改 test 组件的标签名
vm.$children[0].tag = 'p'
```

test 组件更新了，并且更新的是它的根结点的类型，从 `div` 变成 `p` 标签了，因为仅仅触发了子组件的更新，并不会走到 `rootVm._update`，这个时候如果没有上面的那一句 ` if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode)` 表达式更新 `rootVm.$el` 的话，`rootVm.$el` 的指向依然是 test 子组件未更新前的根元素，它的类型是 `div`，而其实现在 `test` 子组件根元素已经是 `p` 标签了，所以会造成 bug。