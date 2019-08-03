# 生命周期函数

## 概念

![lifecycle](https://raw.githubusercontent.com/theniceangel/vue-in-deep/master/assets/lifecycle.png)

生命周期函数总共有 `beforeCreate`、`created`、`beforeMount`、`mounted`、`beforeUpdate`、`updated`、`activated`、`deactivated`、`beforeDestroy`、`destroyed`、`errorCaptured`。

理清在父子组件之中这些函数的执行顺序是有必要的。主要分为如下三种情况：

## 父组件实例化

父 beforeCreate -> 父 created -> 父 beforeMount -> 子 beforeCreate -> 子 created -> 子 beforeMount -> 子 mounted -> 父 mounted

__原因__：在 **beforeMount** 之后，父组件在初始化的时候，先调用 _render 生成 VNode，进而执行 patch 的逻辑，在 patch 的过程中，如果遇到组件 Vnode，则会进行子组件的初始化过程，也就是走 `beforeCreate -> created -> beforeMount -> mounted`，最后回到父组件的 patch 过程，再执行完 patch 之后，调用父组件的 `mounted`，从整体流程来看，执行逻辑类似于**树结构的深度遍历**

## 父组件数据更新

父 beforeUpdate -> 子 beforeUpdate -> 子 updated -> 父 updated

__原因__：在数据发生变化之后，父组件的 renderWatcher 会在 `nextTick` 执行 `flushSchedulerQueue` 函数，内部会调用  `renderWatcher.before()`，并且顺序是根据 Watcher 的 id 决定的。，也就是先执行父组件的 `beforeUpdate`，接着是子组件的 `beforeUpdate`，最后调用 `callUpdatedHooks(updatedQueue)`，内部执行 `updated` 钩子。

## 父组件销毁

父 beforeDestroy -> 子 beforeDestroy -> 子 destroyed -> 父 destroyed

__原因__：父组件的 $destroy 函数先触发 `beforeDestroy`，接着执行 `patch` 函数，在 patch 的时候，会调用 `invokeDestroyHook`，如果是组件 VNode，也就是子组件的占位符 VNode，就会执行子组件VNode 的 destroy hook，函数体内会执行 $destroy，进而走到了子组件的 `beforeDestroy` 和 `destroyed`，等所有的子组件都执行完了，就回到了父组件的 $destroy 函数，触发父组件的 destroyed。

