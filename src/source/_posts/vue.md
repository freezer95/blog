# Vue 源码分析

分析 Vue.js(v2.6.10) 核心代码，分为以下三部分：

### 生命周期

Vue 的生命周期包含 **beforeCreate, created, beforeMount, mounted, beforeUpdate, updated, beforeDestroy, destroyed**

- beforeCreate & created

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

- beforeMount

编译 template 或 el 对应的 DOM 元素的 OuterHTML 为 AST，然后根据 AST 生成 vm.$option.render 函数，优化（检查 static 节点，static 子树）

- mounted

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

- beforeUpdate & updated

检测到组件级wachter相关的数据变化，在 flushSchedulerQueue 中排序后遍历 queue 中的 watcher 执行 watcher.before() 和 watcher.run()。在执行 watcher.before() 中触发 beforeUpdate，在 watcher.run() 后触发 updated

- beforeDestroy & destroyed

### MVVM

### 虚拟 DOM
