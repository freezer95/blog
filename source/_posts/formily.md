---
title: Formily 架构及实现
date: 2020-07-21 9:32:10
tags: formily
description: Formily 作为目前最流行的复杂表单解决方案之一，提供了面向“数据关联，逻辑联动，校验联动”等业务层代码的解决方案。本篇文章适用于有一定 Formily 使用经验的同学，进一步了解 Formily 的设计和实现，适用于版本 v1.1.4。
---
> Formily 是一个由阿里巴巴集团多 BU 共建的面向中后台复杂场景的表单解决方案，它也是一个表单框架。
它的前身是供应链平台在 2019 年初对外开源的 UForm 解决方案，UForm 的前身又是在供应链平台内部自研的某个表单框架。

官方文档：[Formily](https://formilyjs.org/#/bdCRC5/dzUZU8il)、[Uform](https://uform-next.netlify.app/#/MpI2Ij/rRCmCPsOHO)
Formily 作为目前最流行的复杂表单解决方案之一，提供了面向“数据关联，逻辑联动，校验联动”等业务层代码的解决方案。本篇文章**适用于有一定 Formily 使用经验的同学**，进一步了解 Formily 的设计和实现，适用于版本 [v1.1.4](https://github.com/alibaba/formily/releases/tag/v1.1.4)。

# 架构篇
Fomily 拆分成了多包，通过 Lerna 管理，具体的分包如下图。面对这么多包，你是否感觉无从下手？这里，我们可以从架构切入，先了解 Formily 的架构。
<img src="/images/formily_structure_file.png" width="100%">

## 架构设计
[Fomily](https://formilyjs.org/#/bdCRC5/dzUZU8il) 官方文档的架构图和相关说明如下。自底向上，值得重点关注两个层次： 
1. 核心库：@Formily/core。core 仅依赖于 validator 和 shared，设计上与框架、组件库完全解耦；
2. 协议层：Schema Editor。定制标准表单协议，设计上使一份协议多端多组件库通用成为可能。

<img src="/images/formily_structure_layer.png" width="100%">

>- UI 桥接层(React/Vue/Angular/小程序....)，这一层主要是对接各种组件化框架，对不同体系的用户提供更便捷的表单管理方案
>- Schema 动态渲染层(React/Vue/Angular/小程序...)，这一层主要提供了针对 Schema 场景下的各种上层能力，比如典型的协议化联动，协议化布局能力
>- Schema 编辑器层，这一层主要提供了可视化配置 Schema 能力，方便非技术人员快速配置表单
>- 研发工具层，这一层主要提供了针对 Formily 的开发者调试能力

## 架构实践
了解 Formily 的架构设计后，接下来结合 Formily 实际分包和包之间的依赖分析。
### 准备
下载 Fomily 源码，，切换到指定 tag: [v1.1.4](https://github.com/alibaba/formily/releases/tag/v1.1.4)
### 全量的 Dependency Graph
执行以下命令打印 formily 包之间的 dependency
注：相关知识详见 [lerna/commands/list](https://github.com/lerna/lerna/blob/master/commands/list#readme)、[Linux grep 命令](https://man.linuxde.net/grep)
```
lerna ls --graph | grep -e @formily -e ], -e ]  -e { -e }
```
打印结果如下：
```json
{
  "@formily/antd-components": [
    "@formily/antd",
    "@formily/react-schema-renderer",
    "@formily/react-shared-components",
    "@formily/shared",
  ],
  "@formily/antd": [
    "@formily/react-schema-renderer",
    "@formily/react-shared-components",
    "@formily/shared",
  ],
  "@formily/core": [
    "@formily/shared",
    "@formily/validator",
  ],
  "@formily/devtools": [
    "@formily/core",
    "@formily/shared",
  ],
  "@formily/meet-components": [
    "@formily/react",
    "@formily/react-schema-renderer",
    "@formily/react-shared-components",
    "@formily/shared",
  ],
  "@formily/meet": [
    "@formily/react",
    "@formily/react-schema-renderer",
    "@formily/react-shared-components",
    "@formily/shared",
  ],
  "@formily/next-components": [
    "@formily/next",
    "@formily/react-schema-renderer",
    "@formily/react-shared-components",
    "@formily/shared",
  ],
  "@formily/next": [
    "@formily/react-schema-renderer",
    "@formily/react-shared-components",
    "@formily/shared",
  ],
  "@formily/printer": [
    "@formily/react-schema-renderer",
  ],
  "@formily/react-schema-editor": [
    "@formily/antd",
  ],
  "@formily/react-schema-renderer": [
    "@formily/core",
    "@formily/react",
    "@formily/shared",
    "@formily/validator",
  ],
  "@formily/react-shared-components": [
    "@formily/shared",
  ],
  "@formily/react": [
    "@formily/core",
    "@formily/shared"
  ],
  "@formily/shared": [
  ],
  "@formily/validator": [
    "@formily/shared"
  ]
}
```

### 代表性的 Dependency Graph
自底向上（暂不考虑纵向的 Formily DevTools），整理各个层级的代表包后，依赖如下（以 Antd 为代表，忽略非必须的扩展包）：
```json
{
  "@formily/shared": [
  ],
  "@formily/validator": [
    "@formily/shared"
  ],
  "@formily/react-shared-components": [
    "@formily/shared",
  ],
  "@formily/core": [
    "@formily/shared",
    "@formily/validator",
  ],
  "@formily/react": [
    "@formily/core",
    "@formily/shared"
  ], // UI 桥接层(React/Vue/Angular/小程序....)，这一层主要是对接各种组件化框架，对不同体系的用户提供更便捷的表单管理方案
  "@formily/react-schema-renderer": [
    "@formily/core",
    "@formily/react",
    "@formily/shared",
    "@formily/validator",
  ], // Schema 动态渲染层(React/Vue/Angular/小程序...)，这一层主要提供了针对 Schema 场景下的各种上层能力，比如典型的协议化联动，协议化布局能力
  "@formily/antd": [
    "@formily/react-schema-renderer",
    "@formily/react-shared-components",
    "@formily/shared",
  ], // UI 组件层
  "@formily/react-schema-editor": [
    "@formily/antd",
  ], // Schema 编辑器层，这一层主要提供了可视化配置 Schema 能力，方便非技术人员快速配置表单
}
```

# 实现篇
本篇分析上述代表性的包的实现，会涉及到较多源码

## 基础库 - @formily/shared

### 路径定位系统 - FormPath

>FormPath/FormPathPattern 是一个抽象数据路径形式，FormPath 是路径类 ，FormPathPattern 是可以被 FormPath 解析的路径形式，在这里主要使用了 [cool-path](https://github.com/janrywang/cool-path) 路径解析匹配，求值取值能力

上面的说法看着莫名难懂，下面看看代码实现：
@formily/shared - 相关代码
```javascript
import FormPath, { Pattern as FormPathPattern } from 'cool-path'
```
cool-path - Pattern 相关代码
```javascript
export type Pattern = string | number | Path | Segments | MatcherFunction
```
cool-path - FormPath 相关代码
```javascript
export class Path {
  constructor(input: Pattern) {
  }
}
export default Path
```
所以 FormPath 其实就是 cool-path 包中的 Path，FormPathPattern 就是入参类型。支持的 API 和 Path Match Pattern Syntax 详见：[cool-path](https://github.com/janrywang/cool-path)。我们在 Fomily 中常用的以下 "路径模式"，就是 cool-path 支持的。当然，也可以使用 cool-path 支持的其他 "路径模式"。
Expand String
```javascript
"aaa~" or "~" or "aaa~.bbb.cc"
```
Part Wildcard
```javascript
"a.b.*.c.*"
```

## 校验库 - @formily/validator

### 校验系统 - FormValidator
表单校验类，创建表单时，自动在内部创建一个 FormValidator 实例，根据不同 path 管理校验器。**了解这部分代码实现，能对 Formily 中 validator 的设计有清晰认知。**

#### 概述
FormValidator 提供了以下两个重要的实例方法，用法详见下方的单元测试部分。
- register: 注册校验规则，将校验规则保存到名为 nodes 的私有变量集合中，key 为路径FormPath.parse(path || name)，value 为封装了校验规则、校验信息重组等功能的校验器。
- validate: 执行校验，遍历私有变量 nodes，寻找 path 匹配的校验器，使用校验器校验数据，返回校验结果。

此外，FormValidator 还提供了一些静态方法：
- FormValidator.registerRules: 注册校验规则，实例化 FormValidator 时，会自动注册一些默认值，如常见的 required、pattern 等。
- FormValidator.registerFormats：注册校验正则，在校验器执行时，先转换为校验规则的 pattern 类型。
- FormValidator.registerMTEngine：注册校验信息模板方法，如在信息字符串中写{{a.b}}，在方法中替换为校验器提供的 context 中对应路径的值。

#### 源码分析
```javascript
// 校验规则集合
const ValidatorRules = {
  required(value: any, rule: ValidateDescription, rules: ValidateRulesMap) {
    if (rule.required === false) return ''
    return isValidateEmpty(value) ? getRuleMessage(rule, 'required', rules) : ''
  },
  pattern(value: any, rule: ValidateDescription, rules: ValidateRulesMap) {
    if (isValidateEmpty(value)) return ''
    return !new RegExp(rule.pattern).test(value)
      ? getRuleMessage(rule, 'pattern', rules)
      : ''
  },
  // ...省略其他默认值
}
// 校验正则集合
const ValidatorFormators = {
  number: /^[+-]?\d+(\.\d+)?$/,
  // ...省略其他默认值
}

// 调用校验信息模板方法重组校验信息
const template = (message, context): string => {
  if (isStr(message)) {
    if (isFn(FormValidator.template)) {
      return FormValidator.template(message, context)
    }
    return message.replace(/\{\{\s*([\w.]+)\s*\}\}/g, (_, $0) => {
      return FormPath.getIn(context, $0)
    })
  } else if (isObj(message) && !message['$$typeof'] && !message['_owner']) {
    return template(message.message, context)
  } else {
    return message as any
  }
}

class FormValidator {
  private nodes
  static template
  //注册通用规则
  static registerRules(rules: ValidateRulesMap) {
    each(rules, (rule, key) => {
      if (isFn(rule)) {
        ValidatorRules[key] = rule
      }
    })
  }
 // 注册通用正则
  static registerFormats(formats: ValidateFormatsMap) {
    each(formats, (pattern, key) => {
      if (isStr(pattern) || pattern instanceof RegExp) {
        ValidatorFormators[key] = new RegExp(pattern)
      }
    })
  }
  // 注册校验信息模板方法
  static registerMTEngine = template => {
    if (isFn(template)) {
      FormValidator.template = template
    }
  }

  // 注册校验规则
  register (path, calculator) {
    this.nodes[path] = (options) => {
      return new Promise((resolve, reject) => {
        let tmpResult: any
        const validate = async (value, rules) => {
          return this._internalValidate(
            value,
            this._transformRules(rules), // 预处理 rules
            options
          ).then(
            payload => {
              tmpResult = payload
              return payload
            },
            payload => {
              tmpResult = payload
              return Promise.reject(payload)
            }
          )
        }
        Promise.resolve(calculator(validate)).then(
          () => {
            resolve(tmpResult)
          },
          () => {
            reject(tmpResult)
          }
        )
      })
    }
  }

  // 预处理 rules，如:将校验正则转换为校验规则 pattern 
  _transformRules(rules: ValidatePatternRules) {
    if (isStr(rules)) {
      if (!ValidatorFormators[rules]) {
        throw new Error('Can not found validator pattern')
      }
      return [
        {
          pattern: ValidatorFormators[rules],
          message: getMessage(rules) || 'Can not found validator message.'
        }
      ]
    } else if (isFn(rules)) {
      return [
        {
          validator: rules
        }
      ]
    } else if (isArr(rules)) {
      return rules.reduce((buf: any, rule) => {
        return buf.concat(this._transformRules(rule))
      }, [])
    } else if (isObj(rules)) {
      if (rules.format) {
        if (!ValidatorFormators[rules.format]) {
          throw new Error('Can not found validator pattern')
        }
        rules.pattern = ValidatorFormators[rules.format]
        rules.message = rules.message || getMessage(rules.format)
      }
      return [rules]
    }
    return []
  }
  
  // 调用匹配的校验规则函数，获取校验信息，并通过校验信息模板方法重组校验信息
  async _internalValidate( value, rules ) {
    const errors = []
    const warnings = []
    for (let i = 0; i < rules.length; i++) {
      const ruleObj = rules[i]
      for (let l = 0; l < keys.length; l++) {
        const key = keys[l]
        if (ruleObj.hasOwnProperty(key) && isValid(ruleObj[key])) {
          const rule = ValidatorRules[key]
          if (rule) {
            const payload = await rule(value, ruleObj, ValidatorRules)
            const message = template(payload, {
              ...ruleObj,
              rule: ruleObj,
              value
            })
            if (message) errors.push(message)
          }
        }
      }
    }
    return {
      errors,
      warnings
    }
  }

  // 执行校验
  validate (path, options) {
    const pattern = FormPath.getPath(path || '*')
    let errors = []
    await Promise.all(reduce(
        this.nodes,
        (buf, validator, path) => {
          if (pattern.match(path)) {
            return buf.concat(
              // validator 为 register
              validator(options).then(result => {
                if (result.errors.length) {
                  errors = errors.concat({
                    path: path.toString(),
                    messages: result.errors
                  })
                }
              })
            )
          }
          return buf
        }, []
      )
    )
    return {
      errors,
    }
  }
}
```

#### 单元测试
```javascript
test('all', async () => {
  const validator = new FormValidator();
  validator.register('a.b.c.e', validate => {
    return validate('', [
      { required: true },
      {
        format: 'number',
        message: 'This field is not a number.'
      }
    ])
  })
  const validateResponse = await validator.validate()
  expect(validateResponse).toEqual({
    errors: [{ path: 'a.b.c.e', messages: ['This field is required'] }],
    warnings: []
  })
})
```

## 核心库 - @formily/core
顾名思义，核心库是 Formily 中最关键的部分。

### 事件系统 - FormLifeCycle

#### 概述
遵循发布-订阅模式的简单事件系统。
- 创建订阅者时，支持三种模式：单个 handler、指定 type 触发的 handler，包含多个{type: handler}的映射表。
- 发布时，支持指定触发 type、函数实参、函数执行上下文。

#### 源码分析
```javascript
export class FormLifeCycle<Payload = any> {
  private listener: FormLifeCyclePayload<Payload>

  constructor(handler: FormLifeCycleHandler<Payload>)
  constructor(type: string, handler: FormLifeCycleHandler<Payload>)
  constructor(handlerMap: { [key: string]: FormLifeCycleHandler<Payload> })
  constructor(...params: any[]) {
    this.listener = this.buildListener(params)
  }
  buildListener(params: any[]) {
    return function(payload: { type: string; payload: Payload }, ctx: any) {
      //  找到匹配的 type 的 handler，执行 handler
      if (type === payload.type) {
        handler.call(this, payload.payload, ctx)
      }
    }
  }

  notify = <Payload>(type: any, payload: Payload, ctx?: any) => {
    if (isStr(type)) {
      this.listener.call(ctx, { type, payload }, ctx)
    }
  }
}
```

### 事件系统 - FormHeart

#### 概述
基于 FormLifeCycle 的包装器，主要提供了多个 FormLifeCycle 实例注册、默认 context 、发布事件前的钩子、发布事件后的钩子等配置。且 FormHeart 继承了发布-订阅类 Subscribable，支持在 FormHeart 实例上设置事件订阅器、取消事件订阅器、发布事件。

#### 源码分析
```javascript
export class FormHeart<Payload = any, Context = any> extends Subscribable {
  private lifecycles: FormLifeCycle<Payload>[]
  private context: Context
  private beforeNotify?: (...args: any[]) => void
  private afterNotify?: (...args: any[]) => void
  constructor({
    lifecycles,
    context,
    beforeNotify,
    afterNotify
  }: {
    lifecycles?: FormLifeCycle[]
    context?: Context
    beforeNotify?: (...args: any[]) => void
    afterNotify?: (...args: any[]) => void
  } = {}) {
    super()
    this.lifecycles = this.buildLifeCycles(lifecycles || [])
    this.context = context
    this.beforeNotify = beforeNotify
    this.afterNotify = afterNotify
  }
  buildLifeCycles(lifecycles: FormLifeCycle[]) {
   // 处理 lifecycles 参数，主要是深度遍历后铺平成一位数组，元素为 FormLifeCycle 实例
  }
  publish = <P, C>(type: any, payload: P, context?: C) => {
    if (isStr(type)) {
      // 发布事件前的钩子
      if (isFn(this.beforeNotify)) {
        this.beforeNotify(type, payload, context)
      }
      // 发布事件，依次执行 FormLifeCycle 实例上的 handler
      this.lifecycles.forEach(lifecycle => {
        lifecycle.notify(type, payload, context || this.context)
      })
      // 发布事件，执行 FormHeart 实例上的 handler
      this.notify({
        type,
        payload
      })
      // 发布事件后的钩子
      if (isFn(this.afterNotify)) {
        this.afterNotify(type, payload, context)
      }
    }
  }
}
```

### 状态系统 - FormGraph
包含了表单状态和表单项状态，以及扁平化存储状态的 Graph。

#### 概述
表单项状态图。内部主要维护了表单项状态表和父子节点快速索引表，并提供了一系列查询和更新数据的方法。且 FormGraph 继承了发布-订阅类 Subscribable，当 FormGraph 更新时，在更新函数内部自动触发相应类型时间。
重要数据：
- nodes：从表单项路径的维度，**扁平化存储表单及表单项状态**。{[key in string]: NodeType}，其中，key 为 path，value 为对应路径的表单或表单项状态（FormState 、FieldState、VirtualFieldState 的实例）
- refrences：从表单项路径的维度，**扁平化存储表单及表单项父子节点的索引信息**。{[key in string]: FormGraphNodeRef}，其中，key 为 path，value 类型为{parent?: FormGraphNodeRef, path: FormPath, children: FormPath[]}。

查询类型方法（部分）：
- get：精确查询指定 path 的数据节点。get(path: FormPathPattern)
- select：模糊查询指定 path 的数据节点。select(path: FormPathPattern)
- selectParent：精确查询指定 path 的父节点。selectParent(path: FormPathPattern)
- selectChildren：精确查询指定 path 的所有孩子节点，基于 DFS。selectChildren(path: FormPathPattern)
- getLatestParent：精确查询指定 path 的最近有效父节点。getLatestParent(path: FormPathPattern)
- eachChildren：递归遍历所有孩子节点，可以通过参数 recursion 控制是否要深层遍历，默认为 true

更新类型方法：
- appendNode：新增数据节点。appendNode(path: FormPathPattern, node: NodeType)
- remove：移除数据节点。remove(path: FormPathPattern)
- replace：替换数据节点。replace(path: FormPathPattern, node: NodeType)

#### 源码分析
```javascript
export class FormGraph<NodeType = any> extends Subscribable {
  private refrences: {
    [key in string]: FormGraphNodeRef
  }
  private nodes: {
    [key in string]: NodeType
  }
  constructor(props: FormGraphProps = {}) {
    super()
    this.refrences = {}
    this.nodes = {}
  }

  // 模糊查询指定 path 的数据节点
  select(path: FormPathPattern) {
    const pattern = FormPath.parse(path)
    const node = this.get(pattern)
      if (node) {
        return node
      }
    for (let nodePath in this.nodes) {
      const node = this.nodes[nodePath]
      if (pattern.match(nodePath)) {
        return node
      }
    }
  }

  // 精确查询指定 path 的数据节点
  get(path: FormPathPattern) {
    return this.nodes[FormPath.parse(path).toString()]
  }

  // 精确查询指定 path 的父节点
  selectParent(path: FormPathPattern) {
    const selfPath = FormPath.parse(path)
    const parentPath = FormPath.parse(path).parent()
    if (selfPath.toString() === parentPath.toString()) return undefined
    return this.get(parentPath)
  }

  // 精确查询指定 path 的所有孩子节点，基于 DFS
  selectChildren(path: FormPathPattern) {
    const ref = this.refrences[FormPath.parse(path).toString()]
    if (ref && ref.children) {
      return reduce(
        ref.children,
        (buf, path) => {
          return buf.concat(this.get(path)).concat(this.selectChildren(path))
        },
        []
      )
    }
    return []
  }

  // 精确查询指定 path 的最近有效父节点
  getLatestParent(path: FormPathPattern) {
    const selfPath = FormPath.parse(path)
    const parentPath = FormPath.parse(path).parent()
    if (selfPath.toString() === parentPath.toString()) return undefined
    if (this.refrences[parentPath.toString()])
      return {
        ref: this.refrences[parentPath.toString()],
        path: FormPath.parse(parentPath.toString())
      }
    return this.getLatestParent(parentPath)
  }

  // 递归遍历所有孩子节点，可以通过参数 recursion 控制是否要深层遍历
  eachChildren(eacher: FormGraphEacher<NodeType>, recursion?: boolean): void
  eachChildren(
    path: FormPathPattern,
    eacher: FormGraphEacher<NodeType>,
    recursion?: boolean
  ): void
  eachChildren(
    path: FormPathPattern,
    selector: FormPathPattern,
    eacher: FormGraphEacher<NodeType>,
    recursion?: boolean
  ): void
  eachChildren(
    path: any,
    selector: any = true,
    eacher: any = true,
    recursion: any = true
  ) {
    if (isFn(path)) {
      recursion = selector
      eacher = path
      path = ''
      selector = '*'
    }
    if (isFn(selector)) {
      recursion = eacher
      eacher = selector
      selector = '*'
    }
    const ref = this.refrences[FormPath.parse(path).toString()]
    if (ref && ref.children) {
      return each(ref.children, path => {
        if (isFn(eacher)) {
          const node = this.get(path)
          if (
            node &&
            (isFn(this.matchStrategy)
              ? this.matchStrategy(selector, path)
              : FormPath.parse(selector).match(path))
          ) {
            eacher(node, path)
            if (recursion) {
              this.eachChildren(path, selector, eacher, recursion)
            }
          }
        }
      })
    }
  }

  // 新增指定 path 的数据节点
  appendNode(path: FormPathPattern, node: NodeType) {
    const selfPath = FormPath.parse(path)
    const parentPath = selfPath.parent()
    const parentRef = this.refrences[parentPath.toString()]
    const selfRef: FormGraphNodeRef = {
      path: selfPath,
      children: []
    }
    if (this.get(selfPath)) return
    this.nodes[selfPath.toString()] = node
    this.refrences[selfPath.toString()] = selfRef
    if (parentRef) {
      parentRef.children.push(selfPath)
      selfRef.parent = parentRef
    } else {
      const latestParent = this.getLatestParent(selfPath)
      if (latestParent) {
        latestParent.ref.children.push(selfPath)
        selfRef.parent = latestParent.ref
      }
    }
    this.notify({
      type: 'GRAPH_NODE_DID_MOUNT',
      payload: selfRef
    })
  }

  // 移除节点
  remove(path: FormPathPattern) {
    const selfPath = FormPath.parse(path)
    const selfRef = this.refrences[selfPath.toString()]
    if (!selfRef) return
    this.notify({
      type: 'GRAPH_NODE_WILL_UNMOUNT',
      payload: selfRef
    })
    if (selfRef.children) {
      selfRef.children.forEach(path => {
        this.remove(path)
      })
    }
    delete this.nodes[selfPath.toString()]
    delete this.refrences[selfPath.toString()]
    if (selfRef.parent) {
      selfRef.parent.children.forEach((path, index) => {
        if (path.match(selfPath)) {
          selfRef.parent.children.splice(index, 0)
        }
      })
    }
  }

  // 替换节点
  replace(path: FormPathPattern, node: NodeType) {
    const selfPath = FormPath.parse(path)
    const selfRef = this.refrences[selfPath.toString()]
    if (!selfRef) return
    this.notify({
      type: 'GRAPH_NODE_WILL_UNMOUNT',
      payload: selfRef
    })
    this.nodes[selfPath.toString()] = node
    this.notify({
      type: 'GRAPH_NODE_DID_MOUNT',
      payload: selfRef
    })
  }
}
```
### 状态系统 - FormState、FieldState、VirtualFieldState

#### 概述
表单状态、表单项状态、虚拟表单项状态

#### 源码分析
**公用函数 createStateModel**
```javascript
import { Immer } from 'immer'
const { produce } = new Immer({
  autoFreeze: false
})
/**
 * 符合抽象工厂模式，生成状态类的公用函数，适用于表单状态、表单项状态及虚拟表单项状态。
 */
export const createStateModel = <State = {}, Props = {}>(
  Factory: IStateModelFactory<State, Props>
): IStateModelProvider<State, Props> => {
  return class Model<DefaultProps = any> extends Subscribable<State>
    implements IModel<State, Props & DefaultProps> {
    public state: State & { displayName?: string }
    public props: Props &
      DefaultProps & {
        useDirty?: boolean
        computeState?: (draft: State, prevState: State) => void
      }
    public cache?: any
    public displayName?: string
    public dirtyNum: number
    public dirtys: StateDirtyMap<State>
    public prevState: State
    public batching: boolean
    public stackCount: number
    public controller: StateModel<State>
    constructor(defaultProps: DefaultProps) {
      super()
      this.state = { ...Factory.defaultState } // 记录状态
      this.prevState = this.state              // 记录上次的状态，用于判断是否有更新
      this.props = defaults(Factory.defaultProps, defaultProps)
      this.dirtys = {}                         // 记录脏数据
      this.cache = {}
      this.dirtyNum = 0
      this.stackCount = 0
      this.batching = false
      this.controller = new Factory(this.state, this.props)
      this.displayName = Factory.displayName
      this.state.displayName = this.displayName
    }
    // 在 callback 中可批量处理多个状态更新
    batch = (callback?: () => void) => {
      this.batching = true
      if (isFn(callback)) {
        callback()
      }
      if (this.dirtyNum > 0) { //callback 执行完后若存在脏数据，则触发 notify
        this.notify(this.getState())
      }
      this.dirtys = {} // 执行 notify 后，会触发对 dirtys 的一次消费，所以重置
      this.dirtyNum = 0
      this.batching = false
    }
    // 获取实例的 state 属性持久化处理的结果
    getState = (callback?: (state: State) => any) => {
      if (isFn(callback)) {
        return callback(this.getState())
      } else {
        if (isFn(this.controller.publishState)) {
          return this.controller.publishState(this.state)
        }
        if (!hasProxy || this.props.useDirty) {
          return clone(this.state)             // 返回 state 的深拷贝
        } else {
          return produce(this.state, () => {}) // 返回经过 Immer produce 的 state
        }
      }
    }
    // 获取实例的 state 属性
    getSourceState = (callback?: (state: State) => any) => {
      if (isFn(callback)) {
        return callback(this.state)
      } else {
        return this.state
      }
    }
    // 更新实例的 state 属性
    setSourceState = (callback: (state: State) => void) => {
      if (isFn(callback)) {
        callback(this.state)
      }
    }
    setState = (
      callback: (state: State | Draft<State>) => State | void,
      silent = false
    ) => {
      if (isFn(callback)) {
        this.stackCount++
        if (!hasProxy || this.props.useDirty) { // 手动逐个检测更新 state 属性
          const draft = this.getState()
          if (!this.batching) {
            this.dirtys = {}
            this.dirtyNum = 0
          }
          callback(draft)
          if (isFn(this.props.computeState)) {
            this.props.computeState(draft, this.state)
          }
          if (isFn(this.controller.computeState)) {
            this.controller.computeState(draft, this.state)
          }
          const draftKeys = Object.keys(draft || {})
          const stateKeys = Object.keys(this.state || {})
          // 逐个赋值更新 state 属性
          each(
            draftKeys.length > stateKeys.length ? draft : this.state,
            (value, key) => {
              if (!isEqual(this.state[key], draft[key])) {
                this.state[key] = draft[key]
                this.dirtys[key] = true // 标记脏数据
                this.dirtyNum++
              }
            }
          )
          if (this.dirtyNum > 0 && !silent) {
            if (this.batching) {
              this.stackCount--
              return
            }
            this.notify(this.getState())
            this.dirtys = {}
            this.dirtyNum = 0
          }
        } else { // 借助 Immer produce 的能力解决数据更新的问题
          if (!this.batching) {
            this.dirtys = {} // 批量处理过程中需要保存脏数据状态
            this.dirtyNum = 0
          }
          //用proxy解决脏检查计算属性问题
          this.state = produce(
            this.state,
            draft => {
              callback(draft)
              if (isFn(this.props.computeState)) {
                this.props.computeState(draft, this.state)
              }
              if (isFn(this.controller.computeState)) {
                this.controller.computeState(draft, this.state)
              }
            },
            patches => {
              patches.forEach(({ path, op, value }) => {
                if (op === 'replace') {
                  if (path.length > 1 || !isEqual(this.state[path[0]], value)) {
                    this.dirtys[path[0]] = true // 标记脏数据
                    this.dirtyNum++
                  }
                } else {
                  this.dirtys[path[0]] = true   // 标记脏数据
                  this.dirtyNum++
                }
              })
            }
          )
          if (this.dirtyNum > 0 && !silent) {
            if (this.batching) {
              this.stackCount--
              return
            }
            this.notify(this.getState())
            this.dirtys = {} // 执行 notify 后，会触发对 dirtys 的一次消费，所以重置
            this.dirtyNum = 0
          }
        }
        this.stackCount--
        if (!this.stackCount) {
          this.prevState = this.state // state 更新队列清空后，更新 prevState
        }
      }
    }
    // 当前操作的变化情况
    isDirty = (key?: string) =>
      key ? this.dirtys[key] === true : this.dirtyNum > 0
    getDirtyInfo = () => this.dirtys
    // 在一组操作过程中的变化情况
    hasChanged = (path?: FormPathPattern) => {
      return path
        ? !isEqual(
            FormPath.getIn(this.prevState, path),
            FormPath.getIn(this.state, path)
          )
        : !isEqual(this.prevState, this.state)
    }
  } as any
}
```
**FormState**
```javascript
/**
 * 核心数据结构，描述 Form 级别状态。主要逻辑都在 createStateModel 方法中
 */
export const FormState = createStateModel<IFormState, IFormStateProps>(
  class FormState {
    static displayName = 'FormState'
    static defaultState = {
      valid: true,        //是否处于合法态，只要errors长度大于0的时候valid为false
      invalid: false,     //是否处于非法态，只要errors长度大于0的时候valid为true
      loading: false,     // 是否处于loading状态，注意：如果字段处于异步校验时，loading为true
      validating: false,  // 是否处于校验态
      initialized: false, // 是否已经初始化
      submitting: false,
      editable: true,     // 是否可编辑
      modified: false,    // 是否被修改，如果值发生变化，该属性为true，同时在整个字段的生命周期内都会为true
      errors: [],         // 字段错误消息
      warnings: [],       // 字段告警消息
      values: {},         // 表单数据
      initialValues: {},  // 表单数据初始值
      mounted: false,     // 是否挂载
      unmounted: false,   // 是否卸载
      props: {}           // 字段扩展属性
    }
    static defaultProps = {
      lifecycles: []
    }
    private state: IFormState
    constructor(state: IFormState, props: IFormStateProps) {
      this.state = state
      this.state.initialValues = clone(props.initialValues || {})
      this.state.values = clone(props.values || props.initialValues || {})
      this.state.editable = props.editable
    }
    // 在调用 FormState 实例的 setState 方法时，作为 controller.computeState 执行
    computeState(draft: IFormState, prevState: IFormState) {
      // ...省略根据数据，计算 invalid、valid、modified、loading、unmounted、mounted 、props 并更新
     }
  }
)
```
**FieldState**
```javascript
/**
 * 核心数据结构，描述表单字段的所有状态。主要逻辑都在 createStateModel 方法中
 */
export const FieldState = createStateModel<IFieldState, IFieldStateProps>(
  class FieldState {
    static displayName = 'FieldState'
    static defaultState = {
      name: '',                 // 数据路径
      path: '',                 // 节点路径
      dataType: 'any',          // 数据类型
      initialized: false,       // 是否已经初始化
      invalid: false,           // 是否处于非法态，只要errors长度大于0的时候valid为true
      pristine: true,           // 是否处于原始态，只有value===intialValues时的时候该状态为true
      valid: true,              // 是否处于合法态，只要errors长度大于0的时候valid为false
      modified: false,          // 是否被修改，如果值发生变化，该属性为true，同时在整个字段的生命周期内都会为true
      touched: false,           // 是否被触碰
      active: false,            // 是否被激活，字段触发onFocus事件的时候，它会被触发为true，触发onBlur时，为false
      visited: false,           // 是否访问过，字段触发onBlur事件的时候，它会被触发为true
      visible: true,            // 是否可见，注意：该状态如果为false，那么字段的值不会被提交，同时UI不会显示
      display: true,            // 是否展示，注意：该状态如果为false，那么字段的值会提交，UI不会展示，类似于表单隐藏域
      loading: false,           // 是否处于loading状态，注意：如果字段处于异步校验时，loading为true
      validating: false,        // 是否处于校验态
      warnings: [],             // 字段告警消息
      errors: [],               // 字段错误消息
      value: undefined,         // 字段值，与values[0]是恒定相等
      values: [],               // 字段多参值，比如字段onChange触发时，给事件回调传了多参数据，那么这里会存储所有参数的值
      editable: true,           // 是否可编辑
      initialValue: undefined,  // 初始值
      rules: [],                // 校验规则
      required: false,          // 是否必填，为 true 会同时设置校验规则  s
      mounted: false,           // 是否挂载
      unmounted: false,         // 是否卸载
      unmountRemoveValue: true, // 是否在卸载时自动删除字段值
      props: {}                 // 字段扩展属性
      // ...省略其他可忽略属性
    }
    static defaultProps = {
      path: '',
      dataType: 'any'
    }
    private state: IFieldState
    private nodePath: FormPath
    private dataPath: FormPath
    constructor(state: IFieldState, props: IFieldStateProps) {
      this.state = state
      this.nodePath = FormPath.getPath(props.nodePath)
      this.dataPath = FormPath.getPath(props.dataPath)
      this.state.name = this.dataPath.entire
      this.state.path = this.nodePath.entire
      this.state.dataType = props.dataType || 'any'
    }
    // 在调用 FieldState 实例的 setState 方法时，作为 controller.computeState 执行
    computeState(draft: IFieldState, prevState: IFieldState) {
      // ...省略根据数据，计算 invalid、valid、modified、loading、unmounted、mounted、props、rules、required、errors、warnings、editable、pristine 并更新
    }
  }
)
```
**VirtualFieldState**
```javascript
/**
 * Formily特有，描述一个虚拟字段，
 * 它不占用数据空间，但是它拥有状态，
 * 可以联动控制Field或者VirtualField的状态
 * 类似于现在Formily的Card之类的容器布局组件
 * 主要逻辑都在 createStateModel 方法中
 */
export const VirtualFieldState = createStateModel<
  IVirtualFieldState,
  IVirtualFieldStateProps
>(
  class VirtualFieldState {
    static displayName = 'VirtualFieldState'
    static defaultState = {
      name: '',
      path: '',
      initialized: false,
      visible: true,
      display: true,
      mounted: false,
      unmounted: false,
      props: {}
    }
    static defaultProps = {
      path: '',
      props: {}
    }
    private state: IVirtualFieldState
    private path: FormPath
    private dataPath: FormPath
    constructor(state: IVirtualFieldState, props: IVirtualFieldStateProps) {
      this.state = state
      this.path = FormPath.getPath(props.nodePath)
      this.dataPath = FormPath.getPath(props.dataPath)
      this.state.path = this.path.entire
      this.state.name = this.dataPath.entire
    }
    // 在调用 VirtualFieldState 实例的 setState 方法时，作为 controller.computeState 执行
    computeState(draft: IVirtualFieldState, prevState: IVirtualFieldState) {
      // ...省略根据数据，计算 unmounted、mounted、props 并更新
    }
  }
)
```

### 表单 API - 创建表单实例
整个核心库向外导出的 API，将路径系统、校验系统、事件系统、状态系统等都连接起来，构成完整的表单体系。

#### 概述
创建 Form 实例，做好准备工作。

#### 源码分析
```javascript
export interface IFormStateProps {
  initialValues?: {}           // 表单初始值
  values?: {}                  // 表单值
  lifecycles?: FormLifeCycle[] // 生命周期监听器，在这里主要传入FormLifeCycle的实例化对象
  useDirty?: boolean           // 是否使用脏检查，默认会走immer精确更新
  editable?: boolean | ((name: string) => boolean) // 是否可编辑
  validateFirst?: boolean      //是否走悲观校验，遇到第一个校验失败就停止后续校验
}

export interface IFormCreatorOptions extends IFormStateProps {
  onChange?: (values: IFormState['values']) => void
  onSubmit?: (values: IFormState['values']) => any | Promise<any>
  onReset?: () => void
  onValidateFailed?: (validated: IFormValidateResult) => void
}

export function createForm(options: IFormCreatorOptions = {}) {
  function onFormChange(published: IFormState) {
    // ...暂时省略，在"监听表单更新"中详细介绍
  }
  function onFieldChange({ field, path }) {
    // ...暂时省略，在"监听表单更新"中详细介绍
  }
  function onVirtualFieldChange({ field, path }) {
    // ...暂时省略，在"监听表单更新"中详细介绍
  }
  function onGraphChange({ type, payload }) {
    // ...暂时省略，在"监听表单更新"中详细介绍
  }
  function registerField({
    path,
    name,
    value,
    initialValue,
    required,
    rules,
    editable,
    visible,
    display,
    computeState,
    dataType,
    useDirty,
    unmountRemoveValue,
    props
  }: Exclude<IFieldStateProps, 'dataPath' | 'nodePath'>): IField {
    // ...暂时省略，在"注册表单项"中详细介绍
  }
  function registerVirtualField({
    name,
    path,
    props,
    display,
    visible,
    computeState,
    useDirty
  }: IVirtualFieldStateProps): IVirtualField {
    // ...暂时省略，在"注册表单项"中详细介绍
  }
  function getFormState(callback?: (state: IFormState) => any) {
    // ...暂时省略，在"读写表单状态"中详细介绍
  }
  function setFormState(
    callback?: (state: IFormState) => any,
    silent?: boolean
  ) {   // ...暂时省略，在"读写表单状态"中详细介绍
  }
  function getFieldState(
    path: FormPathPattern,
    callback?: (state: IFieldState<FormilyCore.FieldProps>) => any
  ) {    // ...暂时省略，在"读写表单状态"中详细介绍
  }
  function setFieldState(
    path: FormPathPattern,
    callback?: (state: IFieldState<FormilyCore.FieldProps>) => void,
    silent?: boolean
  ) {
    // ...暂时省略，在"读写表单状态"中详细介绍
  }
  function getFormGraph() {
    // ...暂时省略，在"读写表单状态"中详细介绍
  }
  function setFormGraph(nodes: {}) {
    // ...暂时省略，在"读写表单状态"中详细介绍
  }
  async function validate(
    path?: FormPathPattern,
    opts?: IFormExtendedValidateFieldOptions
  ): Promise<IFormValidateResult> {
    // ...暂时省略，在"表单校验和提交"中详细介绍
  }
  async function submit(
    onSubmit?: (values: IFormState['values']) => any | Promise<any>
  ): Promise<IFormSubmitResult> {
    // ...暂时省略，在"表单校验和提交"中详细介绍
  }
  // 定义表单项路径匹配策略，返回值为 boolean。true 为匹配，false 为不匹配
  function matchStrategy(pattern: FormPathPattern, nodePath: FormPathPattern) {
    const matchPattern = FormPath.parse(pattern)
    const node = graph.get(nodePath) // 根据 nodePath 从 FormGraph 实例中取出 node
    if (!node) return false          // 异常情况：未找到匹配的 node，返回 false
    return node.getSourceState(state =>
      // 别名组匹配，path 属性作为别名，由于 VirtualFieldState 的存在，path 与 name 不恒等。在匹配路径时，一般匹配规则下，name 和 path 匹配一个即可。特殊的，面对"非"匹配规则，取 name 和 path 匹配程度更高者做校验，否则存在 bug: https://github.com/alibaba/formily/issues/474
        matchPattern.matchAliasGroup(state.name, state.path)
    )
  }
  const state = new FormState(options)    // 创建 FormState 实例
  const validator = new FormValidator({   // 创建 FormValidator 实例
    ...options,
    matchStrategy
  })
  const graph: FormGraph = new FormGraph({ // 创建 FormGraph 实例
    matchStrategy
  })

  const formApi = {                       // Form 实例向外输出的 API
    registerField,                        // 注册表单项
    registerVirtualField,                 // 注册虚拟表单项
    
    getFormState,                         // 读取表单状态
    setFormState,                         // 更新表单状态
    getFieldState,                        // 读取表单项状态
    setFieldState,                        // 更新表单项状态
    getFormGraph,                         // 读取表单状态图
    setFormGraph,                         // 更新表单状态图
    validate,                             // 校验表单
    submit,                               // 提交表单
    subscribe: (callback?: FormHeartSubscriber) => {
      return heart.subscribe(callback)
    },
    unsubscribe: (id: number) => {
      heart.unsubscribe(id)
    },
    notify: <T>(type: string, payload: T) => {
      heart.publish(type, payload)
    },
    // ...省略其他方法
  }

  const heart = new FormHeart({           // 创建 FormHeart 实例
    ...options,
    context: formApi,
    beforeNotify: payload => {
      env.publishing[payload.path || ''] = true
    },
    afterNotify: payload => {
      env.publishing[payload.path || ''] = false
    }
  })
 
  heart.publish(LifeCycleTypes.ON_FORM_WILL_INIT, state)
  state.subscription = {                  // 订阅 FormState 实例上的事件
    notify: onFormChange
  }
  graph.appendNode('', state)             // FormState 实例添加到 FormGraph 实例中
  state.setState((state: IFormState) => { // 标记 FormState 实例初始化完成
    state.initialized = true
  })
  graph.subscribe(onGraphChange)          // 订阅 FormGraph 实例上的事件
  return formApi
}
```

### 表单 API - 注册表单项

#### 概述
在创建表单实例时，内部创建了两个创建表单项的重要方法，并作为 FormApi 提供给开发者使用：
- registerField
- registerVirtualField

#### 源码分析
**registerField**
```javascript
function registerField({
    path,
    name,
    value,
    initialValue,
    // ...省略其他参数属性
  }: Exclude<IFieldStateProps, 'dataPath' | 'nodePath'>): IField {
    const nodePath = FormPath.parse(path || name) // 路径，包含虚拟表单项
    const dataPath = transformDataPath(nodePath)  // 数据路径，一般不包含虚拟表单项（除非匹配的表单项本身是虚拟表单项）
    const createField = () => {
      const field = new FieldState({    // 创建 FieldState 实例
          nodePath,
          dataPath
        })
      field.subscription = {            // 监听 FieldState 实例上的事件
        notify: onFieldChange({ field, path: nodePath })
      }
      heart.publish(LifeCycleTypes.ON_FIELD_WILL_INIT, field)
      graph.appendNode(nodePath, field) // FieldState 实例添加到 FormGraph 实例中

      // 设置 FieldState 实例上的状态属性
      field.setState((state: IFieldState<FormilyCore.FieldProps>) => {
        state.initialValue = initialValue || formInitialValue // 设置 initialValue 属性
        state.value = value || formValue || initialValue // 设置 value 属性
        // ...省略设置 visible、display、props、required、rules、editable 属性
       state.initialized = true                          // 设置 initialized 标志
      })

      // 注册校验规则, validator 为 FormValidator 实例，详见 FormValidator 部分
      validator.register(nodePath, validate => {
        const {value, rules} = field.getState()
        heart.publish(LifeCycleTypes.ON_FIELD_VALIDATE_START, field)
        return validate(value, rules).then(({ errors, warnings }) => {
          return new Promise(resolve => {
            field.setState((state: IFieldState<FormilyCore.FieldProps>) => {
              state.ruleErrors = errors                 // 设置 ruleErrors 属性
              state.ruleWarnings = warnings             // 设置 ruleWarnings 属性
            })
            heart.publish(LifeCycleTypes.ON_FIELD_VALIDATE_END, field)
            resolve({errors, warnings})                 // 返回校验结果
          })
        })
      })
      return field
    }
    return createField()
  }
```
**registerVirtualField**
```javascript
/**
   * registerVirtualField 相比 registerField 更简单，没有 value 值的初始化、设置、校验
   */
  function registerVirtualField({
    name,
    path,
    // ...省略其他参数属性
  }: IVirtualFieldStateProps): IVirtualField {
    const nodePath = FormPath.parse(path || name)
    const dataPath = transformDataPath(nodePath)
    const createField = () => {
      const field = new VirtualFieldState({ // 创建 VirtualFieldState 实例
          nodePath,
          dataPath,
        })
      field.subscription = {
        notify: onVirtualFieldChange({ field, path: nodePath })
      }
      field.setState(
        (state: IVirtualFieldState<FormilyCore.VirtualFieldProps>) => {
          state.initialized = true
          // 设置 visible、display、props 属性
          state.props = props
          if (isValid(visible)) {
            state.visible = visible
          }
          if (isValid(display)) {
            state.display = display
          }
        }
      )
      return field
    }
    return createField()
  }
```

### 表单 API - 读写表单状态

#### 概述
在创建表单实例时，内部创建了几个读写状态的重要方法，并作为 FormApi 提供给开发者使用：
- getFormState、setFormState
- getFieldState、setFieldState
- getFormGraph、setFormGraph

#### 源码分析
**getFormState、setFormState**
```javascript
function getFormState(callback?: (state: IFormState) => any) {
  return state.getState(callback) // FormState 实例 getState 方法读取状态
}

function setFormState(
  callback?: (state: IFormState) => any,
  silent?: boolean
) {
  state.setState(callback, silent) // FormState 实例 setState 方法更新状态
}
```
**getFieldState、setFieldState**
```javascript
function getFieldState(
  path: FormPathPattern,
  callback?: (state: IFieldState<FormilyCore.FieldProps>) => any
) {
  const field = graph.select(path) // FormGraph 实例 select 方法模糊匹配路径
  return field && field.getState(callback) // FormState 、FieldState、VirtualFieldState 实例 getState 方法读取状态
}

function setFieldState(
  path: FormPathPattern,
  callback?: (state: IFieldState<FormilyCore.FieldProps>) => void,
  silent?: boolean
) {
  if (!isFn(callback)) return
  const pattern = FormPath.getPath(path)
  graph.select(pattern, field => {   // FormGraph 实例 select 方法模糊匹配路径
    field.setState(callback, silent) // FormState 、FieldState、VirtualFieldState 实例 setState 方法更新状态
  }
}
```
**getFormGraph、setFormGraph**
```javascript
function getFormGraph() {
  return graph.map(node => { // FormGraph 实例 map 方法遍历映射表，返回新的映射表
    return node.getState()   // FormState 、FieldState、VirtualFieldState 实例 getState 方法读取状态
  })
}

function setFormGraph(nodes: {}) {
  each(                            // 遍历所有需要更新的节点，逐个更新
    nodes,
    (
      node:
        | IFieldState<FormilyCore.FieldProps>
        | IVirtualFieldState<FormilyCore.VirtualFieldProps>,
      key
    ) => {
      let nodeState: any
      if (graph.exist(key)) {      // 已存在对应状态节点
        nodeState = graph.get(key) // FormGraph 实例 get 方法精确匹配路径
        nodeState.setSourceState(state => { // FormState 、FieldState、VirtualFieldState 实例 setSourceState 返回 state 对象，可通过 js 对象引用特性修改
          Object.assign(state, node)
        })
      } else {                     // 不存在对应状态节点
        if (node.displayName === 'VirtualFieldState') { // 新建虚拟表单项后更新
          nodeState = registerVirtualField({
            path: key
          })
          nodeState.setSourceState(state => {
            Object.assign(state, node)
          })
        } else if (node.displayName === 'FieldState') { // 新建表单项后更新
          nodeState = registerField({
            path: key
          })
          nodeState.setSourceState(state => {
            Object.assign(state, node)
          })
        }
      }
      if (nodeState) {
        nodeState.notify(state.getState()) // FormState 、FieldState、VirtualFieldState 实例 notify 方法发布事件通知更新（因为 setSourceState 方法不会通知，而 getState 方法会通知）。触发 onFormChange、onFieldChange、onVirtualFieldChange
      }
    }
  )
}
```
### 表单 API - 监听表单更新

#### 概述
在创建表单实例时，内部创建了几个监听状态更新的重要方法，仅表单实例内部使用：
- onFormChange
- onFieldChange
- onVirtualFieldChange
- onGraphChange

#### 源码分析
**公共函数**
```javascript
// 从 FormState 实例的 values 属性中获取对应数据路径的值，更新 value
function syncFieldValues(state: IFieldState) {
  state.value = getFormValuesIn(state.name)
}
// 从 FormState 实例的 initialValues 属性中获取对应数据路径的值，更新 initialValue。并结合 value 是否有效，判断是否更新到 value
function syncFieldIntialValues(state: IFieldState) {
  const initialValue = getFormInitialValuesIn(state.name)
  state.initialValue = initialValue
  if (!isValid(state.value) || isEmpty(state.value)) {
      state.value = initialValue
    }
}
```
**onGraphChange**
```javascript
/**
 * 监听表单状态图更新
 * 订阅时机：上面"创建表单实例" createForm 方法中 graph.subscribe(onGraphChange)
 * 触发时机：FormGraph 实例上的 appendNode、remove、replace 方法内部会触发，也可直接调用 notify 方法触发
 */   
function onGraphChange({ type, payload }) {
  heart.publish(LifeCycleTypes.ON_FORM_GRAPH_CHANGE, graph)
  if (type === 'GRAPH_NODE_WILL_UNMOUNT') {
    // 移除对将取消挂载的节点的校验
    validator.unregister(payload.path.toString())
  }
}
```
**onFormChange**
```javascript
/**
 * 监听表单状态更新
 * 订阅时机：上面"创建表单实例" createForm 方法中 state.subscription = {notify: onFormChange}
 * 触发时机：FormState 实例上的 batch、setState 方法内部会触发，也可直接调用 notify 方法触发
 */
function onFormChange(published: IFormState) {
  heart.publish(LifeCycleTypes.ON_FORM_CHANGE, state)
  const valuesChanged = state.isDirty('values')
  const initialValuesChanged = state.isDirty('initialValues')
  const unmountedChanged = state.isDirty('unmounted')
  const mountedChanged = state.isDirty('mounted')
  const initializedChanged = state.isDirty('initialized')
  const editableChanged = state.isDirty('editable')
  // values 或 initialValues 变化
  if (valuesChanged || initialValuesChanged) {
    const updateFields = (field: IField | IVirtualField) => {
      if (isField(field)) {
        field.setState(state => {
          if (valuesChanged) {
            syncFieldValues(state)
          }
          if (initialValuesChanged) {
            syncFieldIntialValues(state)
          }
        })
      }
    }
    // values 或 initialValues 变化，更新每个表单项对应的值
    if (valuesChanged || initialValuesChanged) {
      graph.eachChildren(updateFields) // 遍历更新表单项
    }
    // values 变化，主动调用创建 Form 实例时传入的 onChange 方法，并
    if (valuesChanged) {
      if (
        isFn(options.onChange) &&
        state.state.mounted &&
        !state.state.unmounted
      ) {
        options.onChange(clone(getFormValuesIn(''))) // 调用 onChange 方法，实参为 FormState.values 属性
      }
      heart.publish(LifeCycleTypes.ON_FORM_VALUES_CHANGE, state)
    }
    if (initialValuesChanged) {
      heart.publish(LifeCycleTypes.ON_FORM_INITIAL_VALUES_CHANGE, state)
    }
  }
  // 编辑态变化
  if (editableChanged) {
    graph.eachChildren((field: IField | IVirtualField) => {
      if (isField(field)) {
        field.setState(state => {
          state.formEditable = published.editable
        })
      }
    })
  }
  // 挂载状态变化
  if (unmountedChanged && published.unmounted) {
    heart.publish(LifeCycleTypes.ON_FORM_UNMOUNT, state)
  }
  if (mountedChanged && published.mounted) {
    heart.publish(LifeCycleTypes.ON_FORM_MOUNT, state)
  }
  // 初始化状态变化
  if (initializedChanged) {
    heart.publish(LifeCycleTypes.ON_FORM_INIT, state)
  }
}
```
**onFieldChange**
```javascript
/**
 * 监听表单项状态更新
 * 订阅时机：上面"注册表单项" registerField 方法中 field.subscription = {notify: onFieldChange({ field, path: nodePath })}
 * 触发时机：FieldState 实例上的 batch、setState 方法内部会触发，也可直接调用 notify 方法触发
 */
function onFieldChange({ field, path }) {
  // 更新表单项及所有层次的父子节点的 value
  function notifyTreeFromValues() {
    field.setState(syncFieldValues)
    graph.eachParent(path, (field: IField) => {
      if (isField(field)) {
        field.setState(syncFieldValues, true)
      }
    })
    graph.eachChildren(path, (field: IField) => {
      if (isField(field)) {
        field.setState(syncFieldValues)
      }
    })
    notifyFormValuesChange()
  }
  // 更新表单项及所有层次的父子节点的 initialValue
  function notifyTreeFromInitialValues() {
    field.setState(syncFieldIntialValues)
    graph.eachParent(path, (field: IField) => {
      if (isField(field)) {
        field.setState(syncFieldIntialValues, true)
      }
    })
    graph.eachChildren(path, (field: IField) => {
      if (isField(field)) {
        field.setState(syncFieldIntialValues)
      }
    })
    notifyFormInitialValuesChange()
  }
  return (published: IFieldState<FormilyCore.FieldProps>) => {
    const valueChanged = field.isDirty('value')
    const initialValueChanged = field.isDirty('initialValue')
    const visibleChanged = field.isDirty('visible')
    const displayChanged = field.isDirty('display')
    const mountedChanged = field.isDirty('mounted')
    const initializedChanged = field.isDirty('initialized')
    // ...省略其他状态
    if (initializedChanged) {
      heart.publish(LifeCycleTypes.ON_FIELD_INIT, field)
      // ...省略对空 value 或空 initialValue 的特殊处理
    }
    // 表单项是否隐藏或未挂载
    const wasHidden =
      published.visible == false || published.unmounted === true
    // 表单项 value 变化，显示在页面中时，先更新表单中的 values，再更新表单项及父子表单项的 value
    if (valueChanged) {
      if (!wasHidden) {
        setFormValuesIn(path, published.value, true)
        notifyTreeFromValues()
      }
      heart.publish(LifeCycleTypes.ON_FIELD_VALUE_CHANGE, field)
    }
    // 表单项 initialValue 变化，显示在页面中时，先更新表单中的 initialValues，再更新表单项及父子表单项的 initialValue
    if (initialValueChanged) {
      if (!wasHidden) {
        setFormInitialValuesIn(path, published.initialValue, true)
        notifyTreeFromInitialValues()
      }
      heart.publish(LifeCycleTypes.ON_FIELD_INITIAL_VALUE_CHANGE, field)
    }
    // display 或 visible 变化；visible：数据与样式是否可见，display：样式是否可见
    if (displayChanged || visibleChanged) {
      if (visibleChanged) {               // visible 变化引起的数据的删除与恢复
        if (!published.visible) {         // visible 从 true 更新为 false
          if (isValid(published.value)) { // 暂存有效的 value
            field.setSourceState(
              (state: IFieldState<FormilyCore.FieldProps>) => {
                state.visibleCacheValue = published.value
              }
            )
          }
          deleteFormValuesIn(path)        // 删除 FormState 实例 values 对应数据 
          notifyTreeFromValues()          // 更新表单项及所有层次的父子节点的 value，由于是从 FormState 实例 values 中获取的，表单项及所有层级的子节点的 value 都会置为 undefined
        } else {                          // visible 从 false 更新为 true
          if (!existFormValuesIn(path)) { // 若数据已删除，则从尝试从之前的暂存中设置值，否则直接取初始化值。先更新到 FormState 实例 values 中，再更新表单项及所有层次的父子节点的 value
            setFormValuesIn(
              path,
              isValid(published.visibleCacheValue)
                ? published.visibleCacheValue
                : published.initialValue,
              true
            )
            notifyTreeFromValues()
          }
        }
      }
      // 更新表单项所有层次的子节点的 visible 或 display
      graph.eachChildren(path, childState => {
        childState.setState((state: IFieldState<FormilyCore.FieldProps>) => {
          if (visibleChanged) {
            updateRecoverableShownState(published, state, 'visible')
          }
          if (displayChanged) {
            updateRecoverableShownState(published, state, 'display')
          }
        }, true)
      })
    }
    // unmount 变化，删除或恢复数据
    if (
      unmountedChanged &&
      (published.display !== false || published.visible === false) &&
      published.unmountRemoveValue === true
    ) {
      if (published.unmounted) {
        if (isValid(published.value)) {
          field.setSourceState(
            (state: IFieldState<FormilyCore.FieldProps>) => {
              state.visibleCacheValue = published.value
            }
          )
        }
        deleteFormValuesIn(path, true)
        notifyTreeFromValues()
      } else {
        if (!existFormValuesIn(path)) {
          setFormValuesIn(
            path,
            isValid(published.visibleCacheValue)
              ? published.visibleCacheValue
              : published.initialValue,
            true
          )
          notifyTreeFromValues()
        }
      }
      heart.publish(LifeCycleTypes.ON_FIELD_UNMOUNT, field)
    }
    if (mountedChanged && published.mounted) {
      heart.publish(LifeCycleTypes.ON_FIELD_MOUNT, field)
    }
    // ...省略 errors 和 warnings 变化的同步
    heart.publish(LifeCycleTypes.ON_FIELD_CHANGE, field)
  }
}
```
**onVirtualFieldChange**
```javascript
/**
 * 监听虚拟表单项状态更新，onFieldChange 去除 value 和 initialValue 的简单版本
 * 订阅时机：上面"注册表单项" registerVirtualField 方法中 field.subscription = {notify: onVirtualFieldChange({ field, path: nodePath })}
 * 触发时机：VirtualFieldState 实例上的 batch、setState 方法内部会触发，也可直接调用 notify 方法触发
 */
function onVirtualFieldChange({ field, path }) {
  return (published: IVirtualFieldState<FormilyCore.VirtualFieldProps>) => {
    const visibleChanged = field.isDirty('visible')
    const displayChanged = field.isDirty('display')
    const mountedChanged = field.isDirty('mounted')
    const initializedChanged = field.isDirty('initialized')
    if (initializedChanged) {
      heart.publish(LifeCycleTypes.ON_FIELD_INIT, field)
    }
    // visible 或 display 变化时, 更新表单项所有层次的子节点的 visible 或 display
    if (visibleChanged || displayChanged) {
      graph.eachChildren(path, childState => {
        childState.setState(
          (state: IVirtualFieldState<FormilyCore.VirtualFieldProps>) => {
            if (visibleChanged) {
              updateRecoverableShownState(published, state, 'visible')
            }
            if (displayChanged) {
              updateRecoverableShownState(published, state, 'display')
            }
          },
          true
        )
      })
    }
    if (mountedChanged && published.mounted) {
      heart.publish(LifeCycleTypes.ON_FIELD_MOUNT, field)
    }
    heart.publish(LifeCycleTypes.ON_FIELD_CHANGE, field)
  }
}
```

### 表单 API - 表单校验和提交

#### 概述
在创建表单实例时，内部创建了校验和提交表单的方法，并作为 FormApi 提供给开发者使用：
- validate
- submit

#### 源码分析
**validate**
```javascript
async function validate(
  path?: FormPathPattern,
  opts?: IFormExtendedValidateFieldOptions
): Promise<IFormValidateResult> {
  const { throwErrors = true, hostRendering } = opts || {}
  // 设置 FormState 实例的 validating 为 true，并通知更新
  if (!state.getState(state => state.validating)) {
    state.setSourceState(state => {
      state.validating = true
    })
    // 渲染优化
    clearTimeout(env.validateTimer)
    env.validateTimer = setTimeout(() => {
      state.notify()
    }, 60)
  }
  heart.publish(LifeCycleTypes.ON_FORM_VALIDATE_START, state)
  // 调用 FormValidator 实例的 validate 方法校验指定路径的数据
  const payload = await validator.validate(path, opts)
  // 设置 FormState 实例的 validating 为 false
  state.setState(state => {
    state.validating = false
  })
  heart.publish(LifeCycleTypes.ON_FORM_VALIDATE_END, state)
  // 增加校验结果增加 name 数据路径，和 0.x 保持一致
  const result = {
    errors: payload.errors.map(item => ({
      ...item,
      name: getFieldState(item.path).name
    })),
    warnings: payload.warnings.map(item => ({
      ...item,
      name: getFieldState(item.path).name
    }))
  }
  const { errors, warnings } = result
  // 打印 warnings 日志
  if (warnings.length) {
    log.warn(warnings)
  }
  // throwErrors 配置默认 true，检测到错误会抛错；若设置为 false，则不抛错直接返回
  if (errors.length > 0) {
    if (throwErrors) {
      throw result
    } else {
      return result
    }
  } else {
    return result
  }
}
```
**submit**
```javascript
async function submit(
  onSubmit?: (values: IFormState['values']) => any | Promise<any>
): Promise<IFormSubmitResult> {
  // 重复提交，直接返回，且 submit 方法的返回值不重要，主要在于 validate 过程和 onSubmit 回调
  if (state.getState(state => state.submitting)) return env.submittingTask
  heart.publish(LifeCycleTypes.ON_FORM_SUBMIT_START, state)
  onSubmit = onSubmit || options.onSubmit
  // 设置 submitting 为 true
  state.setState(state => {
    state.submitting = true
  })
  env.submittingTask = async () => {
    heart.publish(LifeCycleTypes.ON_FORM_SUBMIT_VALIDATE_START, state)
    // '' 路径对应的节点是 FormState 的实例，即校验整个表单状态
    await validate('', { throwErrors: false, hostRendering: true })
    // 从 FormState 的实例上获取 errors、warnings
    const validated: IFormValidateResult = state.getState(state => ({
      errors: state.errors,
      warnings: state.warnings
    }))
    const { errors } = validated
    // 校验失败
    if (errors.length) {
      // 由于校验失败导致 submit 退出
      state.setState(state => {
        state.submitting = false
      })
      // 增加 onFormSubmitValidateFailed 来明确结束 submit 的类型
      heart.publish(LifeCycleTypes.ON_FORM_SUBMIT_VALIDATE_FAILED, state)
      heart.publish(LifeCycleTypes.ON_FORM_SUBMIT_END, state)
      if (isFn(options.onValidateFailed) && !state.state.unmounted) {
        options.onValidateFailed(validated)
      }
      throw errors
    }
    // 增加 onFormSubmitValidateSucces 来明确 submit 引起的校验最终的结果
    heart.publish(LifeCycleTypes.ON_FORM_SUBMIT_VALIDATE_SUCCESS, state)
    heart.publish(LifeCycleTypes.ON_FORM_SUBMIT, state)
    // 获取 payload、values，应该是为了避免在 onSubmit 中修改数据，采用了深拷贝
    let payload, values = state.getState(state => clone(state.values))
    if (isFn(onSubmit) && !state.state.unmounted) {
      try {
        payload = await Promise.resolve(onSubmit(values))
        heart.publish(LifeCycleTypes.ON_FORM_ON_SUBMIT_SUCCESS, payload)
      } catch (e) {
        heart.publish(LifeCycleTypes.ON_FORM_ON_SUBMIT_FAILED, e)
        new Promise(() => {
          throw e
        })
      }
    }
    // 设置 submitting 为 true
    state.setState(state => {
      state.submitting = false
    })
    heart.publish(LifeCycleTypes.ON_FORM_SUBMIT_END, state)
    return {
      values,
      validated,
      payload
    }
  }
  return env.submittingTask()
}
```

## UI 桥接层 - @formily/react
>- UI 桥接层(React/Vue/Angular/小程序....)，这一层主要是对接各种组件化框架，对不同体系的用户提供更便捷的表单管理方案

目前只接入了 React

### 组件

#### 概述
由于封装的组件和提供的 API 很多，而且逻辑比较分散独立，这里不再一一介绍。以具有代表性的 FormSpy 组件为例，初步体会桥接层的设计分工：对接框架，提供便捷实用的版本。

#### 源码分析
**FormSpy**
```javascript
/**
 * @param {string | string[]} props.selector - 事件类型选择器
 * @param {(state: any, action: { type: string; payload: any }, form: IForm) => any} reducer - reducer 函数，状态叠加处理，action 为当前命中的生命周期的数据
 * @param {React.ReactElement | ((api: IFormSpyAPI) => React.ReactElement)} children - 内容 
 */
export const FormSpy: React.FunctionComponent<IFormSpyProps> = props => {
  if (isFn(props.children)) {
    return props.children(useFormSpy(props)) || <Fragment />
  } else {
    return props.children || <Fragment />
  }
}
FormSpy.displayName = 'ReactInternalFormSpy'
FormSpy.defaultProps = {
  selector: `*`,
  reducer: (state, action) => {
    return {
      ...state,
      ...action.payload
    }
  }
}

export const useFormSpy = (props: IFormSpyProps): ISpyHook => {
  const form = useContext(FormContext)
  const subscriberId = useRef<number>()
  const [type, setType] = useState<string>(LifeCycleTypes.ON_FORM_INIT)
  const [state, dispatch] = useReducer(
    (state, action) => props.reducer(state, action, form),
    props.initialState || {}
  )
  const subscriber = useCallback<FormHeartSubscriber>(({ type, payload }) => {
    payload = payload && isFn(payload.getState) ? payload.getState() : payload
    if (isStr(props.selector) && FormPath.parse(props.selector).match(type)) {
      // selector 为字符串类型，正则匹配事件类型
      setType(type)
      dispatch({
        type,
        payload
      })
    } else if (isArr(props.selector)) {
      // selector 为数组类型，精确匹配多个事件类型
      if (
        !!(props.selector as any[]).find(str => {
          if (isStr(str)) {
            return str === type
          } else if (isArr(str)) {
            if (str.length < 2) {
              return str[0] === type
            }
          }
        })
      ) {
        setType(type)
        dispatch({
          type,
          payload
        })
      }
    }
  }, [])
  useMemo(() => {
    // form subscribe 实际上调用的是 heart.subscribe，监听整个 FormHeart 实例上的事件
    subscriberId.current = form.subscribe(subscriber)
  }, [])
  useEffect(() => {
    return () => {
      if (form) {
        form.unsubscribe(subscriberId.current)
      }
    }
  }, [])
  return {
    form,
    type,
    state
  }
}
```

## Schema 动态渲染层 - @formily/react-schema-renderer
>- Schema 动态渲染层(React/Vue/Angular/小程序...)，这一层主要提供了针对 Schema 场景下的各种上层能力，比如典型的协议化联动，协议化布局能力

### 组件

#### 概述
由于封装的组件和透传的 API 很多，而且逻辑比较分散独立，这里不再一一介绍。以具有代表性的 SchemaField 组件为例，初步体会动态渲染层的设计分工：处理 schema。

#### 源码分析

**FormSpy**
```javascript
// 由于这一层主要是针对 Schema 场景的处理，不涉及 FormSpy 的实现，因此直接从 react 包导出
export { FormSpy } from '@formily/react'
```
**SchemaField**
```jsx
export const SchemaField: React.FunctionComponent<ISchemaFieldProps> = (
  props: ISchemaFieldProps
) => {
  const path = FormPath.parse(props.path)
  const formSchema = useContext(SchemaContext)
  const fieldSchema = new Schema(props.schema || formSchema.get(path))
  const formRegistry = useContext(FormComponentsContext)
  const schemaType = fieldSchema.type                       // 'type'
  const schemaComponent = fieldSchema.getExtendsComponent() // 'x-component'
  const schemaRenderer = fieldSchema.getExtendsRenderer()   // 'x-render'
  const initialComponent = schemaComponent || schemaType    // 'x-component' || 'type'
  const renderField = ( // 遍历渲染子节点
    addtionKey: string | number,
    reactKey?: string | number
  ) => {
    return <SchemaField key={reactKey} path={path.concat(addtionKey)} />
  }
  const renderFieldDelegate = (
    callback: (props: ISchemaFieldComponentProps) => React.ReactElement
  ) => {
    return (
      <Field
        path={path}
        // ...省略其他属性的设置
      >
        {({ state, mutators, form }) => {
          const props: ISchemaFieldComponentProps = {
            ...state,
            schema: new Schema(fieldSchema).merge(state.props),
            form,
            mutators,
            renderField
          }
          return callback(props)
        }}
      </Field>
    )
  }
  const renderVirtualFieldDelegate = (
    callback: (props: ISchemaVirtualFieldComponentProps) => React.ReactElement
  ) => {
    return (
      <VirtualField
        path={path}
        // ...省略其他属性的设置
      >
        {({ state, form }) => {
          const props: ISchemaVirtualFieldComponentProps = {
            ...state,
            schema: new Schema(fieldSchema).merge(state.props),
            form,
            renderField,
            children: fieldSchema.mapProperties(
              (schema: Schema, key: string) => {
                return (
                  <SchemaField
                    schema={schema}
                    key={key}
                    path={path.concat(key)}
                  />
                )
              }
            )
          }
          return callback(props)
        }}
      </VirtualField>
    )
  }
  // object 类型，每个 kv 都是一个表单项，遍历生成 SchemaField 组件，插入到当前节点
  if (fieldSchema.type === 'object' && !schemaComponent) {
    const properties = fieldSchema.mapProperties(
      (schema: Schema, key: string) => {
        const childPath = path.concat(key)
        return (
          <SchemaField
            schema={schema}
            key={childPath.toString()}
            path={childPath}
          />
        )
      }
    )
    return renderFieldDelegate(props => {
      const renderComponent = () => {
        return React.createElement(
          formRegistry.formItemComponent,
          props,
          properties
        )
      }
      if (isFn(schemaRenderer)) {
        return schemaRenderer({ ...props, renderComponent })
      }
      return renderComponent()
    })
  } else {
    if (isStr(initialComponent)) { // schema 中配置的组件是字符串
      if (formRegistry.fields[initialComponent]) {
        // 组件字符串对应表单组件，用 FormItem 包裹
        return renderFieldDelegate(props => {
          const stateComponent = lowercase(
            props.schema.getExtendsComponent() || props.schema.type
          )
          const renderComponent = (): React.ReactElement => {
            return React.createElement(
              formRegistry.formItemComponent,
              props,
              React.createElement(formRegistry.fields[stateComponent], props)
            )
          }
          if (isFn(schemaRenderer)) {
            return schemaRenderer({ ...props, renderComponent })
          }
          return renderComponent()
        })
      } else if (formRegistry.virtualFields[initialComponent]) {
        // 组件字符串对应虚拟组件，直接渲染
        return renderVirtualFieldDelegate(props => {
          const stateComponent = lowercase(
            props.schema.getExtendsComponent() || props.schema.type
          )
          const renderComponent = () => {
            return React.createElement(
              formRegistry.virtualFields[stateComponent],
              props
            )
          }
          if (isFn(schemaRenderer)) {
            return schemaRenderer({ ...props, renderComponent })
          }
          return renderComponent()
        })
      }
    }
  }
}
```

## UI 组件层 - @formily/antd
这一层主要负责借助 @formily/react-schema-renderer 提供的组件和能力，结合 antd 组件库，封装成带组件库的包。

### 组件

#### 概述
由于封装的组件和透传的 API 很多，而且逻辑比较分散独立，这里不再一一介绍。以具有代表性的 Submit 组件为例，初步体会组件层的设计分工：结合三方组件库。

#### 源码分析
**Submit**
```jsx
import { FormSpy } from '@formily/react-schema-renderer'
import { Button } from 'antd'

// 在 FormSpy 的基础上封装：内置 antd Button 组件
export const Submit = ({ showLoading, onSubmit, ...props }: ISubmitProps) => {
  return (
    <FormSpy
      selector={[
        LifeCycleTypes.ON_FORM_SUBMIT_START,
        LifeCycleTypes.ON_FORM_SUBMIT_END
      ]}
      reducer={(state, action) => {
        switch (action.type) {
          case LifeCycleTypes.ON_FORM_SUBMIT_START:
            return {
              ...state,
              submitting: true
            }
          case LifeCycleTypes.ON_FORM_SUBMIT_END:
            return {
              ...state,
              submitting: false
            }
          default:
            return state
        }
      }}
    >
      {({ state, form }) => {
        return (
          <Button
            onClick={e => {
              if (props.htmlType !== 'submit') {
                form.submit(onSubmit)
              }
              if (props.onClick) {
                props.onClick(e)
              }
            }}
            disabled={showLoading ? state.submitting : undefined}
            {...props}
            loading={showLoading ? state.submitting : undefined}
          >
            {props.children || '提交'}
          </Button>
        )
      }}
    </FormSpy>
  )
}
Submit.defaultProps = {
  showLoading: true,
  type: 'primary',
  htmlType: 'submit'
}
```

## Schema 编辑器层 - @formily/react-schema-editor
>- Schema 编辑器层，这一层主要提供了可视化配置 Schema 能力，方便非技术人员快速配置表单

### 体验
之前没有使用过这个包，官方文档也没有任何说明。按照 README.md 用法使用，实际体验后，目前应该就是一个 json schema 编辑器，在同一个页面中提供了视图操作、直接编辑和预览效果的功能。但视图操作不好用，直接编辑没有高亮和错误提示，目前整体体验不好，期待以后的发展吧。

<img src="/images/formily_schema.png" width="100%">

# QA

## 路径系统 name（不包含虚拟表单项） 和 path（包含虚拟表单项）共存

### 用法
路径系统中存在 name 和 path，在寻找对应表单项时，可以根据 name 的规则来匹配，也可以根据 path 的规则来匹配。如下图：

<img src="/images/formily_qa_1.png" width="100%">

### 原因
对于路径系统中 name 和 path 均可使用的设计，开发者可能会存在疑问。下面尝试分析原因：
首先，根据 Form 实例的 registerField 方法，可以看到 FormGraph 和 FormValidator 实例的 key 都是 nodePath，即包含虚拟表单项的路径。可以理解对于 Formily 内部来说，path 是保持内部机制正常运作的重要路径。(补充，dataPath，即一般不包含虚拟表单项的路径，会作为 FormState 实例中 values 属性的路径，也充分表达了其数据路径的含义 ，代码在 registerVirtualField 部分有列出)

```javascript
function registerField({path, name}): IField {
  let field: IField
  const nodePath = FormPath.parse(path || name) // 路径，包含虚拟表单项
  const dataPath = transformDataPath(nodePath)  // 数据路径，一般不包含虚拟表单项（除非匹配的表单项本身是虚拟表单项）
  const createField = (field?: IField) => {
    const alreadyHaveField = !!field
    field =
      field ||
      new FieldState({
        nodePath,
        dataPath,
        // ...省略
      })
    field.subscription = {
      notify: onFieldChange({ field, path: nodePath })
    }
    heart.publish(LifeCycleTypes.ON_FIELD_WILL_INIT, field)
    if (!alreadyHaveField) {
      graph.appendNode(nodePath, field)
    }
    validator.register(nodePath, validate => {
      // ...省略
    })
    return field
  }
  if (graph.exist(nodePath)) {
    // ...省略
  } else {
    field = createField()
  }
  return field
}
```

然后，再观察 FieldState 构造函数，可以看到 FieldState 实例的 state.name 属性根据 dataPath 计算，最终一般不包含虚拟表单项（除非匹配的表单项本身是虚拟表单项）；state.path 属性根据 nodePath 计算，最终包含虚拟表单项。关于修改这点的原因，是因为开发者需求，详见 [issues543](https://github.com/alibaba/formily/issues/543)。

```javascript
export const FieldState = createStateModel<IFieldState, IFieldStateProps>(
  class FieldState {
    private state: IFieldState
    private nodePath: FormPath
    private dataPath: FormPath
    constructor(state: IFieldState, props: IFieldStateProps) {
      this.state = state
      this.nodePath = FormPath.getPath(props.nodePath)
      this.dataPath = FormPath.getPath(props.dataPath)
      this.state.name = this.dataPath.entire
      this.state.path = this.nodePath.entire
      this.state.dataType = props.dataType || 'any'
    }
  }
}
```
因此，猜测路径系统中 name 和 path 的并存的原因在于：内部主要是根据 path 存储和计算的，但是为了方便开发者理解和使用，增加了 name，只关注数据路径，忽略无意义的虚拟表单项路径。
