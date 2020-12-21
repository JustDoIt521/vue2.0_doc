VUE源码阅读笔记

### 1 寻找vue的主入口

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

### 2 Vue 属性及方法的添加

从 `import Vue from vue` 一共经历了以下几个文件。

```javascript
import Vue from 'vue'
        ↓
src/platforms/web/entry-runtime-with-compiler
        ↓
src/platforms/web/runtime/index.js
        ↓
src/core/index.js
        ↓
src/core/instance/index.js
  
```

根据js的加载顺序 我们从底部往上挨个捋。

`src/core/instance/index.js` 

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

`initMixin`   |  `core/instance/init.js`

```javascript
export function initMixin (Vue: Class<Component>) {

  Vue.prototype._init = function (options?: Object) {
    ...
  }
}
```

添加 `_init`方法。该方法也是 `Vue`的执行入口

`stateMixin`  |   `core/instance/state.js `

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

添加 `$set`、`$del`方法。 这两个方法来自 `core/observer/index.js`。后面再看



















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





