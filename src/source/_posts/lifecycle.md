---
title: Vue(v2.6.10) - 生命周期
date: 2019-06-21 10:00:00
tags: vue
description: beforeCreate, created, beforeMount, mounted, beforeUpdate, updated, beforeDestroy, destroyed
---

## 背景
Vue 作为当前热门的框架之一，主要难点分为三大部分：模板编译、MVVM、虚拟DOM。而生命周期作为实例从创建到销毁全过程暴露的钩子，既为用户提供了在不同阶段注入代码的机会，也提供了了解整个框架源码的切入点。
本篇文章，以生命周期为入口，旨在帮助大家抓住 Vue 框架运行的主线，会涉及到三大部分（模板编译、MVVM、虚拟DOM）的概念，但不会做更深入的说明。如有兴趣，可自行阅读其他资料或源码。

## 图示
主要的生命周期钩子可分为四类，每类有开始前（beforeXx） 和 已完成（过去式）两个钩子：
|  类别   | 钩子（开始前）  | 钩子（已完成）  |
|  ----  | ----  | ----  |
| 创建  | beforeCreate | created |
| 挂载  | beforeMount | mounted |
| 更新  | beforeUpdate | updated |
| 销毁  | beforeDestroy | destroyed |
接下来通过 [Vue 官网生命周期图示](https://cn.vuejs.org/v2/guide/instance.html#生命周期图示)，了解生命周期钩子的含义及模糊逻辑。

## 源码解析
通过图示对生命周期有所了解后，下面从源码层面初步了解生命周期钩子的具体逻辑

#### beforeCreate & created
```javascript
// Vue 构造函数，通过 new 生成 Vue 实例时，调用 Vue.prototype.__init__
function Vue (options) {
    if (!(this instanceof Vue)
    ) {
      warn('Vue is a constructor and should be called with the `new` keyword');
    }
    this._init(options);
  }
```

```javascript
    Vue.prototype._init = function (options) {
      var vm = this;
      vm._uid = uid$3++; // 自增的标识符
      vm._isVue = true;  // a flag to avoid this being observed
      vm.$options = ...  // merge options
      {
        initProxy(vm);   // 若当前环境支持 Proxy，则将vm的proxy保存在vm._renderProxy中
      }
      vm._self = vm;     // expose real self
      initLifecycle(vm); // 设置生命周期相关的属性值，如: vm._isMounted, vm._isDestroyed 等为 false
      initEvents(vm);    // 初始化 vm._events = Object.create(null)
      initRender(vm);    // 初始化 vm._c
      callHook(vm, 'beforeCreate');
      initInjections(vm); // resolve injections before data/props
      /**
       * 初始化 vm._watchers 和 vm.$options
       * vm._watchers = []
       * initProps
       * initMethods
       * initData：vm._data为真实数据对象，代理 vm.key 的 get, set 到 vm._data.key 上; observe(data, true)
       * initComputed：初始化 vm._computedWatchers，new Watcher(vm, getter || noop, noop, computedWatcherOptions);，defineComputed
       * initWatch：用vm.$watch(expOrFn, handler, options)的原理实现
       **/
      initState(vm);
      initProvide(vm);    // resolve provide after data/props
      callHook(vm, 'created');

      if (vm.$options.el) {
        vm.$mount(vm.$options.el);
      }
    };
  }
```

#### beforeMount
编译 template 或 el 对应的 DOM 元素的 OuterHTML 为 AST，然后根据 AST 生成 vm.$option.render 函数，优化（检查 static 节点，static 子树）

#### mounted
创建组件级wachter，执行 render 函数生成虚拟 Dom，执行 update 函数，通过 patch vnode 和 oldVnode 操作真实 DOM 节点

```javascript
  Vue.prototype.$mount = function (el, hydrating) {
    el = el && inBrowser ? query(el) : undefined;
    return mountComponent(this, el, hydrating)
  };

  var mount = Vue.prototype.$mount;
  /**
   * 生成 render 和 staticRenderFns 函数，保存在 vm.$options 中
   **/
  Vue.prototype.$mount = function (el, hydrating) {
    el = el && query(el);
    var options = this.$options;
    // resolve template/el and convert to render function
    if (!options.render) {
      var template = options.template;
      if (template) {
        if (typeof template === 'string') {
          if (template.charAt(0) === '#') {
            template = idToTemplate(template);
          }
        } else if (template.nodeType) {
          template = template.innerHTML;
        } else {
          {
            warn('invalid template option:' + template, this);
          }
          return this
        }
      } else if (el) {
        template = getOuterHTML(el);
      }
      if (template) {
        var ref = compileToFunctions(template, {
          outputSourceRange: "development" !== 'production',
          shouldDecodeNewlines: shouldDecodeNewlines,
          shouldDecodeNewlinesForHref: shouldDecodeNewlinesForHref,
          delimiters: options.delimiters,
          comments: options.comments
        }, this);
        var render = ref.render;
        var staticRenderFns = ref.staticRenderFns;
        options.render = render;
        options.staticRenderFns = staticRenderFns;
      }
    }
    return mount.call(this, el, hydrating)
  };
```

```javascript
function mountComponent (vm, el, hydrating) {
    vm.$el = el;
    callHook(vm, 'beforeMount');

    var updateComponent = function () {
      vm._update(vm._render(), hydrating);
    };
    // 创建组件级别的 watcher
    // 实例化watcher时会执行this.get()获取value值，即执行updateComponent方法
    // 实例化的watcher实例保存在vm._watcher属性中，也会push进vm._watchers
    new Watcher(vm, updateComponent, noop, {
      before: function before () {
        if (vm._isMounted && !vm._isDestroyed) {
          callHook(vm, 'beforeUpdate');
        }
      }
    }, true /* isRenderWatcher */);
    hydrating = false;

    // manually mounted instance, call mounted on self
    // mounted is called for render-created child components in its inserted hook
    if (vm.$vnode == null) {
      vm._isMounted = true;
      callHook(vm, 'mounted');
    }
    return vm
  }
```

#### beforeUpdate & updated
检测到组件级wachter相关的数据变化，在 flushSchedulerQueue 中排序后遍历 queue 中的 watcher 执行 watcher.before() 和 watcher.run()。在执行 watcher.before() 中触发 beforeUpdate，在 watcher.run() 执行完毕后触发 updated

```javascript
function flushSchedulerQueue () {
  currentFlushTimestamp = getNow();
  flushing = true;
  var watcher, id;

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  queue.sort(function (a, b) { return a.id - b.id; });

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index];
    if (watcher.before) {
      watcher.before();
    }
    id = watcher.id;
    has[id] = null;
    watcher.run();
    // in dev build, check and stop circular updates.
    if (has[id] != null) {
      circular[id] = (circular[id] || 0) + 1;
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? ("in watcher with expression \"" + (watcher.expression) + "\"")
              : "in a component render function."
          ),
          watcher.vm
        );
        break
      }
    }
  }

  // keep copies of post queues before resetting state
  var activatedQueue = activatedChildren.slice();
  var updatedQueue = queue.slice();

  resetSchedulerState();

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue);
  callUpdatedHooks(updatedQueue);

  // devtool hook
  /* istanbul ignore if */
  if (devtools && config.devtools) {
    devtools.emit('flush');
  }
}

function callUpdatedHooks (queue) {
  var i = queue.length;
  while (i--) {
    var watcher = queue[i];
    var vm = watcher.vm;
    if (vm._watcher === watcher && vm._isMounted && !vm._isDestroyed) {
      callHook(vm, 'updated');
    }
  }
}
```

#### beforeDestroy & destroyed
在检测到数据变化后，执行watcher.run()的过程中，会执行watcher.get()，执行实例化wacher时传入的expOrFn。实例化 watcher的代码如下：
```javascript
    var updateComponent = function () {
      vm._update(vm._render(), hydrating);
    };
    new Watcher(vm, updateComponent, noop, {
      before: function before () {
        if (vm._isMounted && !vm._isDestroyed) {
          callHook(vm, 'beforeUpdate');
        }
      }
    }, true /* isRenderWatcher */);
```
因此，当检测到组件内的数据变化时，执行对该数据有依赖的watcher.run()方法时，会执行上述的updateComponent方法，在updateComponent中，通过 Vue.prototype._render()生成新的虚拟DOM树，在Vue.prototype._update中，会对新旧虚拟DOM树通过patch方法进行比对，在patch 方法中通过patchVnode比较新旧节点，通过 updateChildren 比较新旧子树（虚拟DOM部分详见另一篇文章）

对于存在于旧的虚拟DOM树，不存在于新的虚拟DOM树的节点，调用 removeVnodes 方法，若节点为待删除的组件，则执行对应组件的 Vue.prototype.$destroy方法，先后触发 beforeDestroy 和 destroyed

```javascript
Vue.prototype.$destroy = function () {
  var vm = this;
  if (vm._isBeingDestroyed) {
    return
  }
  callHook(vm, 'beforeDestroy');
  vm._isBeingDestroyed = true;
  // remove self from parent
  var parent = vm.$parent;
  if (parent && !parent._isBeingDestroyed && !vm.$options.abstract) {
    remove(parent.$children, vm);
  }
  // teardown watchers
  if (vm._watcher) {
    vm._watcher.teardown();
  }
  var i = vm._watchers.length;
  while (i--) {
    vm._watchers[i].teardown();
  }
  // remove reference from data ob
  // frozen object may not have observer.
  if (vm._data.__ob__) {
    vm._data.__ob__.vmCount--;
  }
  // call the last hook...
  vm._isDestroyed = true;
  // invoke destroy hooks on current rendered tree
  vm.__patch__(vm._vnode, null);
  // fire destroyed hook
  callHook(vm, 'destroyed');
  // turn off all instance listeners.
  vm.$off();
  // remove __vue__ reference
  if (vm.$el) {
    vm.$el.__vue__ = null;
  }
  // release circular reference (#6759)
  if (vm.$vnode) {
    vm.$vnode.parent = null;
  }
}
```
**注：重点**
```javascript
var updateComponent = function () { // 连接数据更新和 Dom 更新
  vm._update(vm._render(), hydrating);
};
```
## 示例
初步了解钩子的逻辑后，接下来借助 devtools 调试 Vue ( v2.6.10 ) 源码，进一步了解 Vue 实例不同阶段的具体逻辑。调试的 demo 地址详见[示例代码地址](https://github.com/freezer95/vue_code_analysis_demo)。

## 总结
至此，以生命周期钩子为切入点，初步介绍了 Vue 框架 Core 的运行机制。如果将 Vue 框架 Core 比喻为一栋楼，生命周期的钩子就是不同楼层的标志，三大部分（模板编译、MVVM、虚拟DOM）就是几批分布在不同楼层的多个房间，暴露出来的 API 就是房间的门窗。愿每一位开发者都能透过门窗，面朝源码，拥抱 Bug。

## 扩展
推荐书籍：《深入浅出 Vue.js》刘博文 著
