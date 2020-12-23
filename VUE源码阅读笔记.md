VUE源码阅读笔记

# 1 寻找vue的主入口

首先看 `package.json`   (为了阅读和调试的方便，我们这里使用的是`dev`版本) 。从配置可以看出来 编译入口是 `scripts/config.js`  。

```json
"scripts": {
    "dev": "rollup -w -c scripts/config.js --environment TARGET:web-full-dev",
  	...
    "build": "node scripts/build.js",
    ...
  },
```

`scripts/config.js`    package.json 文件已经给我标明了 来自  `web-full-dev`。 我们看一下 `web-full-dev`的配置

```javascript
const builds = {
 ...
  'web-full-dev': {
    entry: resolve('web/entry-runtime-with-compiler.js'),
    dest: resolve('dist/vue.js'),
    sourceMap: true,
    format: 'umd',
    env: 'development',
    alias: { he: './entity-decoder' },
    banner
  },
    ...
}
```

这里的 `resolve`定义如下

```javascript
const aliases = require('./alias')
const resolve = p => {
  const base = p.split('/')[0]
  if (aliases[base]) {
    return path.resolve(aliases[base], p.slice(base.length + 1))
  } else {
    return path.resolve(__dirname, '../', p)
  }
}
```

通过 `alias.js`统一配置并读取

```javascript
module.exports = {
  vue: resolve('src/platforms/web/entry-runtime-with-compiler'),
  compiler: resolve('src/compiler'),
  core: resolve('src/core'),
  shared: resolve('src/shared'),
  web: resolve('src/platforms/web'),
  weex: resolve('src/platforms/weex'),
  server: resolve('src/server'),
  sfc: resolve('src/sfc')
}
```

那么当我们运行 `npm run dev`的时候， 入口真实文件为 `src/platforms/web/entry-runtime-with-compiler.js`。

```javascript
...
import Vue from './runtime/index'
...
export default Vue
```

继续寻找 `runtime/index`

```javascript
import Vue from 'core/index'
...
export default Vue
```

`core/index`绝对路径为 `src/core/index.js`

```javascript
import Vue from './instance/index'
...
export default Vue
```

继续向下  `src/core/instance/index.js`

```javascript
import { initMixin } from './init'    
import { stateMixin } from './state'  
import { renderMixin } from './render'  
import { eventsMixin } from './events' 
import { lifecycleMixin } from './lifecycle' 
import { warn } from '../util/index'  // 警告

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)  // 初始化
stateMixin(Vue) // 状态
eventsMixin(Vue) // 事件
lifecycleMixin(Vue)  // 生命周期
renderMixin(Vue)  // 渲染

export default Vue
```

至此 算是找到了 `Vue`  的核心部分。

# 2 Vue 属性及方法的添加

## 路线

从 `import Vue from vue` 一共经历了以下几个文件。

```javascript
import Vue from 'vue'
        ↓
src/platforms/web/entry-runtime-with-compiler    // 4层
        ↓
src/platforms/web/runtime/index.js   //  3层
        ↓
src/core/index.js  // 2层
        ↓
src/core/instance/index.js   // 1层
  
```

根据js的加载顺序 我们从底部往上挨个捋。

## 1层

src/core/instance/index.js 

```javascript
...
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)
...
```

定义了 构造函数 `Vue`  这里有五个混入函数。 我们先看着五个函数添加的方法

### initMixin   

  core/instance/init.js

```javascript
export function initMixin (Vue: Class<Component>) {

  Vue.prototype._init = function (options?: Object) {
    ...
  }
}
```

添加 `_init`方法。该方法也是 `Vue`的执行入口

### stateMixin  

   core/instance/state.js

```javascript
import {
  set,
  del,
  ...
} from '../observer/index'

export function stateMixin (Vue: Class<Component>) {
  const dataDef = {}
  dataDef.get = function () { return this._data }
  const propsDef = {}
  propsDef.get = function () { return this._props }
  if (process.env.NODE_ENV !== 'production') {
    dataDef.set = function () {
      warn(
        'Avoid replacing instance root $data. ' +
        'Use nested data properties instead.',
        this
      )
    }
    propsDef.set = function () {
      warn(`$props is readonly.`, this)
    }
  }
  Object.defineProperty(Vue.prototype, '$data', dataDef)
  Object.defineProperty(Vue.prototype, '$props', propsDef)

  Vue.prototype.$set = set
  Vue.prototype.$delete = del

  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
    ...
  }
}
```

添加 `$set`、`$del`方法。设置`$data`、 `$props` 。当访问这两个属性的时候返回  `this._data`、 `this._props`。

### eventsMixin 

core/instance/events.js

```JavaScript
export function eventsMixin (Vue: Class<Component>) {
  const hookRE = /^hook:/
  Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
    ...
  }

  Vue.prototype.$once = function (event: string, fn: Function): Component {
    ...
  }

  Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {
    ...
  }

  Vue.prototype.$emit = function (event: string): Component {
   ...
  }
}

```

添加 `$on`、`$once` 、`$off` 、 `$emit`方法。

### lifecycleMixin

core/instance/lifecycle.js

```javascript
export function lifecycleMixin (Vue: Class<Component>) {
  Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  	...
  }

  Vue.prototype.$forceUpdate = function () {
  	...
  }

  Vue.prototype.$destroy = function () {
    ...
  }
}
```

添加 `_update`、`$forceUpdate` 、`$destroy`方法。

### renderMixin

core/instance/render.js

```JavaScript
export function renderMixin (Vue: Class<Component>) {
  // install runtime convenience helpers
  installRenderHelpers(Vue.prototype)

  Vue.prototype.$nextTick = function (fn: Function) {
    ...
  }

  Vue.prototype._render = function (): VNode {
   	...
  }
}
```

添加 `$nextTick` 、`_render`方法。 这里调用了函数  `installRenderHelpers`。这个函数在 `core/instance/render-helpers/index`

```javascript
import { toNumber, toString, looseEqual, looseIndexOf } from 'shared/util'
import { createTextVNode, createEmptyVNode } from 'core/vdom/vnode'
import { renderList } from './render-list'
import { renderSlot } from './render-slot'
import { resolveFilter } from './resolve-filter'
import { checkKeyCodes } from './check-keycodes'
import { bindObjectProps } from './bind-object-props'
import { renderStatic, markOnce } from './render-static'
import { bindObjectListeners } from './bind-object-listeners'
import { resolveScopedSlots } from './resolve-scoped-slots'
import { bindDynamicKeys, prependModifier } from './bind-dynamic-keys'

export function installRenderHelpers (target: any) {
  target._o = markOnce
  target._n = toNumber
  target._s = toString
  target._l = renderList
  target._t = renderSlot
  target._q = looseEqual
  target._i = looseIndexOf
  target._m = renderStatic
  target._f = resolveFilter
  target._k = checkKeyCodes
  target._b = bindObjectProps
  target._v = createTextVNode
  target._e = createEmptyVNode
  target._u = resolveScopedSlots
  target._g = bindObjectListeners
  target._d = bindDynamicKeys
  target._p = prependModifier
}
```

在 Vue的原型上添加 上述方法。

## 2层

看完 `core/instance/index.js`  我们往上走， 看 `src/core/index.js`。

```JavaScript
import Vue from './instance/index'
import { initGlobalAPI } from './global-api/index'
import { isServerRendering } from 'core/util/env'
import { FunctionalRenderContext } from 'core/vdom/create-functional-component'

initGlobalAPI(Vue)

Object.defineProperty(Vue.prototype, '$isServer', {
  get: isServerRendering
})

Object.defineProperty(Vue.prototype, '$ssrContext', {
  get () {
    /* istanbul ignore next */
    return this.$vnode && this.$vnode.ssrContext
  }
})

// expose FunctionalRenderContext for ssr runtime helper installation
Object.defineProperty(Vue, 'FunctionalRenderContext', {
  value: FunctionalRenderContext
})

Vue.version = '__VERSION__'

export default Vue
```

在 `Vue`的原型上添加 `$isServer`、`$ssrContext`属性，设置了它们的 `get`属性。  这里调用了 `initGlobalAPI`方法。 这个方法在 `core/global-api/index.js`

### initGlobalAPI

```javascript
export function initGlobalAPI (Vue: GlobalAPI) {
  // config
  const configDef = {}
  configDef.get = () => config
  if (process.env.NODE_ENV !== 'production') {
    configDef.set = () => {
      warn(
        'Do not replace the Vue.config object, set individual fields instead.'
      )
    }
  }
  Object.defineProperty(Vue, 'config', configDef)

  // exposed util methods.
  // NOTE: these are not considered part of the public API - avoid relying on
  // them unless you are aware of the risk.
  Vue.util = {
    warn,
    extend,
    mergeOptions,
    defineReactive
  }

  Vue.set = set
  Vue.delete = del
  Vue.nextTick = nextTick

  // 2.6 explicit observable API
  Vue.observable = <T>(obj: T): T => {
    observe(obj)
    return obj
  }

  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  // this is used to identify the "base" constructor to extend all plain-object
  // components with in Weex's multi-instance scenarios.
  Vue.options._base = Vue
  extend(Vue.options.components, builtInComponents)

  initUse(Vue)
  initMixin(Vue)
  initExtend(Vue)
  initAssetRegisters(Vue)
}
```

给构造函数 `Vue`添加 `config`， `util`， `set`，`delete`，`nextTick`, `observable`, `options`等方法。

这里有这几个方法调用  我们继续看一下 这结果方法的调用情况。

```JavaScript
 initUse(Vue)
 initMixin(Vue)
 initExtend(Vue)
 initAssetRegisters(Vue)
```

#### initUse

core/global-api/use.js

```javascript
import { toArray } from '../util/index'

export function initUse (Vue: GlobalAPI) {
  Vue.use = function (plugin: Function | Object) {
   ...
  }
}
```

添加 `use`方法

#### initMixin

core/global-api/mixin.js

```javascript
export function initMixin (Vue: GlobalAPI) {
  Vue.mixin = function (mixin: Object) {
    ...
  }
}
```

添加 `mixin`方法

#### initExtend

core/global-api/initExtend.js

```javascript
export function initExtend (Vue: GlobalAPI) {
  /**
   * Each instance constructor, including Vue, has a unique
   * cid. This enables us to create wrapped "child
   * constructors" for prototypal inheritance and cache them.
   */
  Vue.cid = 0
  let cid = 1

  /**
   * Class inheritance
   */
  Vue.extend = function (extendOptions: Object): Function {
    ...
  }
}
```

添加 `extend`方法

#### initAssetRegisters

core/global-api/assets.js

```javascript
export function initAssetRegisters (Vue: GlobalAPI) {
  /**
   * Create asset registration methods.
   */
  ASSET_TYPES.forEach(type => {
    Vue[type] = function (
      id: string,
      definition: Function | Object
    ): Function | Object | void {
      ...
      }
    }
  })
}

```

这里是遍历了 `ASSET_TYPES`的属性，并添加到 `Vue`函数上。  `ASSET_TYPES`的定义如下

src/share/constants.js

```javascript

export const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]
```

## 3层

src/platforms/web/runtime/index.js 

```javascript
import Vue from 'core/index'
...

// install platform specific utils
Vue.config.mustUseProp = mustUseProp
Vue.config.isReservedTag = isReservedTag
Vue.config.isReservedAttr = isReservedAttr
Vue.config.getTagNamespace = getTagNamespace
Vue.config.isUnknownElement = isUnknownElement

// install platform runtime directives & components
extend(Vue.options.directives, platformDirectives)
extend(Vue.options.components, platformComponents)

// install platform patch function
Vue.prototype.__patch__ = inBrowser ? patch : noop

// public mount method
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  ...
}

...

export default Vue
```

在vue的原型上添加 `$mount`方法。 并对 Vue 的 config  和 options 属性进行扩展

## 4层

src/platforms/web/entry-runtime-with-compiler

```javascript
...

import Vue from './runtime/index'
...

const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  ...
}

...

export default Vue

```

在 Vue的原型上添加 `$mount`方法





# new Vue()

从上一章可以知道， 当我们执行 `new Vue`的时候，其实是调用了 `this._init`方法。我们接下来就分析一下 在 `new`的过程中 发生了啥。











































定义了构造函数 `Vue`并暴露出去。通常项目的入口文件 `app.js`中调用 `new Vue({...})`也是调用了这里的 `Vue`函数

```JavaScript
// initMixin 的作用相当于将_init 函数挂载在了Vue 对象上   _init 也是实例化Vue对象后执行的操作入口
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
 	...
  }
}
```

```JavaScript
// stateMixin 定义数据的get方法。 以及绑定$set $delete  watch
export function stateMixin (Vue: Class<Component>) {

  const dataDef = {}
  dataDef.get = function () { return this._data }
  const propsDef = {}
  propsDef.get = function () { return this._props }
  ...
  Object.defineProperty(Vue.prototype, '$data', dataDef)
  Object.defineProperty(Vue.prototype, '$props', propsDef)

  Vue.prototype.$set = set
  Vue.prototype.$delete = del

  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
    ....
  }
}
```

```JavaScript
// 定义$on $off $once $emit 方法
export function eventsMixin (Vue: Class<Component>) {
  const hookRE = /^hook:/
  Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
    ...
  }

  Vue.prototype.$once = function (event: string, fn: Function): Component {
   ...
  }

  Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {
    ...
  }

  Vue.prototype.$emit = function (event: string): Component {
    ...
  }
}

```

```JavaScript
// 定义vue _update $forceUpdate $destroy 方法
export function lifecycleMixin (Vue: Class<Component>) {
  Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
   ...
  }

  Vue.prototype.$forceUpdate = function () {
    ...
  }

  Vue.prototype.$destroy = function () {
    ...
  }
}
```

```JavaScript
// 定义 $nextTick方法    _render渲染函数
export function renderMixin (Vue: Class<Component>) {
  // install runtime convenience helpers
  installRenderHelpers(Vue.prototype)

  Vue.prototype.$nextTick = function (fn: Function) {
    return nextTick(fn, this)
  }

  Vue.prototype._render = function (): VNode {
      ...
  }
}

/*------------------------*/
// installRenderHelpers  各种渲染的时候要用到的函数
export function installRenderHelpers (target: any) {
  ...
}
```

综上  在引入`VUe`之后， 通过 `initMixin`、`stateMixin`、`eventsMixin`、`lifecycleMixin`、`renderMixin`将Vue的 各种属性，状态，事件，钩子函数，渲染等绑定在了 `Vue`的原型上。

下面来看一下 在 `new Vue({...})`之后， 具体的执行部分。

首先执行了 `this_init`。 这个函数在 `src/core/instance/init.js`

```javascript
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    // a uid
    vm._uid = uid++

    let startTag, endTag
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      startTag = `vue-perf-start:${vm._uid}`
      endTag = `vue-perf-end:${vm._uid}`
      mark(startTag)
    }

    // a flag to avoid this being observed
    vm._isVue = true
    // merge options
    if (options && options._isComponent) {  
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }
    // expose real self
    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')

    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      vm._name = formatComponentName(vm, false)
      mark(endTag)
      measure(`vue ${vm._name} init`, startTag, endTag)
    }

    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
```

#### _isComponent 的来源

_isComponent 属性的加载 来自函数 `createElement` 位置 `core/vdom/create-element.js`。 这里调用 `createElement` 调用了 ` _createElement`。 在这个方法中调用了 `createComponent` 位置`core/vdom/create-component.js` 这里定义了一个 对象 `componentVNodeHooks`

```javascript
const componentVNodeHooks = {
  init (vnode: VNodeWithData, hydrating: boolean): ?boolean {
    if (
      vnode.componentInstance &&
      !vnode.componentInstance._isDestroyed &&
      vnode.data.keepAlive
    ) {
      // kept-alive components, treat as a patch
      const mountedNode: any = vnode // work around flow
      componentVNodeHooks.prepatch(mountedNode, mountedNode)
    } else {
      const child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance
      )
      child.$mount(hydrating ? vnode.elm : undefined, hydrating)
    }
  },
...
}
```

在这里面调用了 `createComponentInstanceForVnode` 在该函数中给 `options`添加上 `_isComponent:ture`

```javascript
export function createComponentInstanceForVnode (
  // we know it's MountedComponentVNode but flow doesn't
  vnode: any,
  // activeInstance in lifecycle state
  parent: any
): Component {
  const options: InternalComponentOptions = {
    _isComponent: true,
    _parentVnode: vnode,
    parent
  }
 ...
  return new vnode.componentOptions.Ctor(options)
}
```

后面再具体分析这块。 先看主流程





