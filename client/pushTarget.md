# pushTarget

源码内部 pushTarget 的逻辑是用来切换当前 Watcher，这样能保证 Dep 类能够精确的收集到依赖，但是源码里面有两处让我比较困惑的地方，第一处位于 `src/core/instance/state.js`，第二处位于 `src/core/instance/lifecycle.js`。

## state.js

```js
export function initState (vm: Component) {
  // ......
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  // ......
}

export function getData (data: Function, vm: Component): any {
  // #7573 disable dep collection when invoking data getters
  pushTarget()
  try {
    return data.call(vm, vm)
  } catch (e) {
    handleError(e, vm, `data()`)
    return {}
  } finally {
    popTarget()
  }
}
```

也就是在将 data 转化为 `getter/setter` 的时候，会调用 getData 里的 pushTarget() 与 popTarget()。这样做的原因就是在执行 getData 的时候，将当前 Watcher 设置为 null，这样对应的响应式数据就不能收集依赖了。

为什么要这么做呢，我想了很久，后面看了下 [issue](https://github.com/vuejs/vue/issues/7573)，考虑到如下的一个场景。

```js
let calls = 0
const vm = new Vue({
  template: `<child :msg="msg"></child>`,
  data: {
    msg: 'hello'
  },
  beforeUpdate () { calls++ },
  components: {
    child: {
      template: `<span>{{ localMsg }}</span>`,
      props: ['msg'],
      data () {
        return {
          localMsg: this.msg
        }
      },
      computed: {
        computedMsg () {
          return this.msg + ' world'
        }
      }
    }
  }
}).$mount()
const child = vm.$children[0]
vm.msg = 'hi'

setTimeout(() => {
  console.log(calls)
}, 1000)
```

如果父组件的 `msg` 触发了 setter。`beforeUpdate` 执行了两次，为啥呢？

首先，`msg` 收集了父组件的依赖，会触发一次 `beforeUpdate`，这个我知道。再者注意到子组件在 data 函数是依赖了 `props.msg`，也就是说在初始化子组件的时候，会先走到子组件的 `initData` 逻辑，但是这个时候当前的 Watcher 仍然是父组件的 renderWatcher，也就是说子组件的 `props.msg` 会收集当前父组件的 renderWatcher 作为依赖，只要子组件的 `props.msg` 触发了 setter，那么父组件的 renderWatcher 就会触发 update 逻辑。

那么什么时候触发子组件的 `props.msg` 的 setter 呢？

还是得理清 Vue 的执行流程。父组件执行 updateComponent 的时候会执行 _render 拿到自己对应的 VNode，接着执行 _update，走到 patch 的流程，发现如果是组件 VNode，会走到组件 VNode 的 prepatch 钩子，在这个钩子里面会执行 `updateChildComponent`。代码如下

```js
// update props
  if (propsData && vm.$options.props) {
    toggleObserving(false);
    var props = vm._props;
    var propKeys = vm.$options._propKeys || [];
    for (var i = 0; i < propKeys.length; i++) {
      var key = propKeys[i];
      var propOptions = vm.$options.props; // wtf flow?
      props[key] = validateProp(key, propOptions, propsData, vm);
    }
    toggleObserving(true);
    // keep a copy of raw propsData
    vm.$options.propsData = propsData;
  }
```

其中这一段 `props[key] = validateProp(key, propOptions, propsData, vm);`，就触发了子组件 `props.msg` 的 setter，这个时候在 `nextTick` 就会执行父组件的重新渲染了


这样的话，就触发了两次 `update`。这个 bug 在 Vue 的 `2.4.2` 被别人提了 issue。所以修复的代码就如上面 `getData` 的逻辑里面加了 `pushTarget` 与 `popTarget`。接下来看下第二处有疑惑的地方，原理是类似的。

## lifecycle.js

```js
export function callHook (vm: Component, hook: string) {
  // #7573 disable dep collection when invoking lifecycle hooks
  pushTarget()
  const handlers = vm.$options[hook]
  const info = `${hook} hook`
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      invokeWithErrorHandling(handlers[i], vm, null, vm, info)
    }
  }
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook)
  }
  popTarget()
}
```

也就是说在生命周期钩子里面，不需要触发依赖收集。拿上面的例子来说。

```js
let calls = 0
const vm = new Vue({
  template: `<child :msg="msg"></child>`,
  data: {
    msg: 'hello'
  },
  beforeUpdate () { calls++ },
  components: {
    child: {
      template: `<span>{{ localMsg }}</span>`,
      props: ['msg'],
      mounted () {
        this.a = this.msg
      },
      data: { localMsg: 123 },
      computed: {
        computedMsg () {
          return this.msg + ' world'
        }
      }
    }
  }
}).$mount()
const child = vm.$children[0]
vm.msg = 'hi'

setTimeout(() => {
  console.log(calls)
}, 1000)
```

假如 `callHook` 里面没有 `pushTarget` 和 `popTarget` 的话，那么在子组件的 `mounted` 里面，如果先对 `props.msg` 触发 getter，依然是会将当前 Watcher 收集进依赖，不过当前 Watcher 仍然是父组件的 renderWatcher。因此，`pushTarget` 和 `popTarget` 的作用就是将当前 Watcher 置为 null，这样子组件的 `props.msg` 收集依赖的时候就会把父组件的 renderWatcher 给忽略。