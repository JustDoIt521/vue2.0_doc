

#### VUE源码阅读笔记

### 1 寻找vue的主入口

首先看 `package.json`找到 `scripts:{build:xxx}`  可以找到编译入口为 `node scripts/build.js`。通过  `build.js`找到 `buildEntry`来自文件 `config.js`。通过 `config.js`可以找到编译不同版本的所有配置。

`package.js`的入口为 `module:dist/vue.runtime.esm.js`。那么我们在 `config.js`中找到配置为

```javascript
web-runtime-esm': {
    entry: resolve('web/entry-runtime.js'),
    dest: resolve('dist/vue.runtime.esm.js'),
    format: 'es',
    banner
 },
```

`reolve`函数通过 `alias.js`解析得到路径， `alias.js`中配置如下

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

到 `Vue`入口为 `src/platforms/web/entry-runtime-with-compiler`,通过 该文件 找到 `import Vue from './runtime/index'`，

```javascript
import Vue from 'core/index'
...
// 此处在Vue的 原型上挂载了 $mount方法
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
```

 在该文件中继续寻找`import Vue from 'core/index.js'`，继续在 `core/index.js` 文件中找到 `import Vue from './instance/index'`。

至此 寻找到 `Vue`的核心部分 `core/instance/index.js` 内部即为 `Vue`核心源码。

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

