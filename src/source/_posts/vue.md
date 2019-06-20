# Vue 源码分析

分析 Vue 核心代码，分为以下三部分：

### 生命周期

Vue 的生命周期包含 **beforeCreate, created, beforeMount, mounted, beforeUpdate, updated, beforeDestroy, destroyed**

- beforeCreate

- created

包装数据

- beforeMount

编译 template 或 el 对应的 DOM 元素的 OuterHTML 为 AST，然后根据 AST 生成 vm.$option.render 函数，优化（检查 static 节点，static 子树）

- mounted

执行 render 函数生成虚拟 Dom，执行 update 函数，通过 patch vnode 和 oldVnode 操作真实 DOM 节点，创建 Watcher 监听组件内数据变化

- beforeUpdate

检测到数据变化

- updated

再次执行 render() 和 update()

- beforeDestroy

- destroyed

### MVVM

### 虚拟 DOM
