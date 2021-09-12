



# 声明diff的过程



core/instance/lifecycle.js



mountComponent  

挂载组件的过程中 声明了一个watcher

```javascript
  let updateComponent
  
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    updateComponent = () => {
      const name = vm._name
      const id = vm._uid
      const startTag = `vue-perf-start:${id}`
      const endTag = `vue-perf-end:${id}`

      mark(startTag)
      const vnode = vm._render()
      mark(endTag)
      measure(`vue ${name} render`, startTag, endTag)

      mark(startTag)
      vm._update(vnode, hydrating)
      mark(endTag)
      measure(`vue ${name} patch`, startTag, endTag)
    }
  } else {
    updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }
  }
  
new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
```

其中_update 来自。lifeyclemixin（位置同上） 注册 _update事件

```javascript
export function lifecycleMixin (Vue: Class<Component>) {
。。。
  Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const restoreActiveInstance = setActiveInstance(vm)
    vm._vnode = vnode
    // Vue.prototype.__patch__ is injected in entry points
    // based on the rendering backend used.
    if (!prevVnode) {
      // initial render  初始渲染
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    restoreActiveInstance()
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    // if parent is an HOC, update its $el as well
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
    // updated hook is called by the scheduler to ensure that children are
    // updated in a parent's updated hook.
  }

 。。。
}
```



# diff过程



## 核心：

​	core/vdom/patch.js

整体diff过程 返回自 `createPatchFunction`

## 核心判断流程一 patch

```javascript
function patch (oldVnode, vnode, hydrating, removeOnly) {
  // isUndef 判断是否为 undefined 或者null。isDef 判读 是否非undefined&&非null
  // 如果vnode 不存在。 oldvnode存在 则销毁oldVnode
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }
  
	  //是否为初次渲染
    let isInitialPatch = false
    const insertedVnodeQueue = []

    if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      // oldVnode不存在，也就是初次渲染。则创建root节点
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
      //根据nodeType判断是否为真实节点 
      /*
      nodeType 属性返回节点类型。
      如果节点是一个元素节点，nodeType 属性返回 1。
      如果节点是属性节点, nodeType 属性返回 2。
      如果节点是一个文本节点，nodeType 属性返回 3。
      如果节点是一个注释节点，nodeType 属性返回 8。
      真实节点的nodetype为固有只读属性
      */
      const isRealElement = isDef(oldVnode.nodeType)
      // oldVnode为非真实节点(即虚拟节点) oldVnode 和vnode相同 则直接修改现有节点 即继续往下比较
      // 第一次渲染的时候没有 oldVnode， 传入的是 vm.$el	为真实dom节点。添加判断isReaalElement 是为了处理这种情况
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // patch existing root node
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } else {
        // 挂载到真实dom节点上
        if (isRealElement) {
          // mounting to a real element
          // check if this is server-rendered content and if we can perform
          // a successful hydration.
          
          //如果是服务端渲染 则删除SSR_ATTR属性 并将hydrating设置为true 
          // hydrating 是与服务端渲染相关
          if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
            oldVnode.removeAttribute(SSR_ATTR)
            hydrating = true
          }
          if (isTrue(hydrating)) {
            if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
              //调用插入钩子
              invokeInsertHook(vnode, insertedVnodeQueue, true)
              return oldVnode
            } else if (process.env.NODE_ENV !== 'production') {
              warn(
                'The client-side rendered virtual DOM tree is not matching ' +
                'server-rendered content. This is likely caused by incorrect ' +
                'HTML markup, for example nesting block-level elements inside ' +
                '<p>, or missing <tbody>. Bailing hydration and performing ' +
                'full client-side render.'
              )
            }
          }
          // either not server-rendered, or hydration failed.
          // create an empty node and replace it
          //如果 服务端渲染失败或者插入到vnode失败 则创建一个空节点代替
          oldVnode = emptyNodeAt(oldVnode)
        }

        // 创建真实dom 取代现有节点
        // replacing existing element
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)

        // create new node
        createElm(
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )
				
        //递归更新父级node节点
        // update parent placeholder node element, recursively
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            // 删除需要删除的祖先节点
            // cbs是一个对象 存储了 不同hooks和对应函数  对于cbs 和hooks的描述详见下文 cbs & hooks 
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            
            //vnode 如果可修补 则执行创建钩子
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)
              }
              // #6513
              // invoke insert hooks that may have been merged by create hooks.
              // e.g. for directives that uses the "inserted" hook.
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                // start at index 1 to avoid re-invoking component mounted hook
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }
				//删除父级节点
        // destroy old node
        if (isDef(parentElm)) {
          removeVnodes([oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }

    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
  }
```

### sameVnode

判断oldVnode 和 vnode 是否是同一个。即判断当前节点是否未发生改变

```javascript
function sameVnode (a, b) {
     /*
     asyncFactory 异步组件会存在一个asyncFactory 即异步组件的constructor
     key 值
     tag 标签
     isComment 是否未注释节点
      是否data（当前节点对应的对象，包含了具体的一些数据信息，是一个VNodeData类型，可以参考VNodeData类型中的数据信息）都有定义
  当标签是<input>的时候，type必须相同
    */
  return (
    a.key === b.key &&
    a.asyncFactory === b.asyncFactory && (
      (
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        isDef(a.data) === isDef(b.data) &&
        sameInputType(a, b)
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}

//如果是input类型节点。则判断input的type是否一样 或者 a、b的type不一致时 是否属于isTextInputType 
function sameInputType (a, b) {
  if (a.tag !== 'input') return true
  let i
  // 相当于判断 data 和 attrs 存在 情况下 将attrs赋值给了 i。返回i.type
  const typeA = isDef(i = a.data) && isDef(i = i.attrs) && i.type
  const typeB = isDef(i = b.data) && isDef(i = i.attrs) && i.type
  return typeA === typeB || isTextInputType(typeA) && isTextInputType(typeB)
}

/* 创建了一个对象 
isTextInputType = {
  text: true,
  number: true,
  password: true,
  search: true,
  email: true,
  tel: true,
  url: true
}
*/
export const isTextInputType = makeMap('text,number,password,search,email,tel,url')

```



## 核心判断流程二  patchVnode

core/vdom/patch.js

```javascript
function patchVnode (
    oldVnode,
    vnode,
    insertedVnodeQueue,
    ownerArray,
    index,
    removeOnly
  ) {
    //节点相同则直接返回
    if (oldVnode === vnode) {
      return
    }
		//存在可复用的情况则 直接复用。 updateChildren 函数中会存在对节点的四种类型比较 会出现这个情况
    if (isDef(vnode.elm) && isDef(ownerArray)) {
      // clone reused vnode
      vnode = ownerArray[index] = cloneVNode(vnode)
    }

    const elm = vnode.elm = oldVnode.elm
		
    // 异步组件的处理
    if (isTrue(oldVnode.isAsyncPlaceholder)) {
      if (isDef(vnode.asyncFactory.resolved)) {
        hydrate(oldVnode.elm, vnode, insertedVnodeQueue)
      } else {
        vnode.isAsyncPlaceholder = true
      }
      return
    }
		
    // 静态组件则直接复用元素  这时候是相当于复用了oldVnode的组件实例。  要注意 只有当vnode是clone的情况下会这样做。  
    // reuse element for static trees.
    // note we only do this if the vnode is cloned -
    // if the new node is not cloned it means the render functions have been
    // reset by the hot-reload-api and we need to do a proper re-render.
    if (isTrue(vnode.isStatic) &&
      isTrue(oldVnode.isStatic) &&
      vnode.key === oldVnode.key &&
      (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
    ) {
      vnode.componentInstance = oldVnode.componentInstance
      return
    }

    let i
    const data = vnode.data
    // i = data.hook.prepatch  参见 core/vdom/create-component.js    
    // componentVNodeHooks.prepatch 大致作用类似于更新子组件实例。 
    // 根据updateChildComponent的描述 大致类似于 组件有slot children的时候 更新slot children
    if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
      i(oldVnode, vnode)
    }

    const oldCh = oldVnode.children
    const ch = vnode.children
    // vnode是节点的情况 且vnode.data存在 调用update钩子
    if (isDef(data) && isPatchable(vnode)) {
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }
     
    //  text 当节点为文本节点的时候 为xxx 其余为 undefined。
    if (isUndef(vnode.text)) { // 如果vnode节点没有文本内容 即vnode为非文本节点
      if (isDef(oldCh) && isDef(ch)) {  
        // 当vnode 和oldVnode都有子节点的时候 执行updateChildren
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
      } else if (isDef(ch)) {
        // 如果仅vnode有子元素 则清空oldVnode的内容 在vnode上添加子节点
        if (process.env.NODE_ENV !== 'production') {
          checkDuplicateKeys(ch)
        }
        // 如果oldVnode文本有内容 把vnode的文本内入置空
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
        
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
      } else if (isDef(oldCh)) {
        // 如果仅oldVnode有子元素  则在vnode中删除 子元素
        removeVnodes(oldCh, 0, oldCh.length - 1)
      } else if (isDef(oldVnode.text)) {
        // 当新老节点均无子节点 则就是文本替换
        nodeOps.setTextContent(elm, '')
      }
    } else if (oldVnode.text !== vnode.text) {
      // 如果新老节点的文本不同 则直接替换文本
      nodeOps.setTextContent(elm, vnode.text)
    }
     //  调用 data.hook.postpatch钩子
    if (isDef(data)) {
      if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
    }
  }
```

## 核心判断流程三  updateChildren

core/vnode/patch.js

```javascript
function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
    let oldStartIdx = 0
    let newStartIdx = 0
    let oldEndIdx = oldCh.length - 1
    let oldStartVnode = oldCh[0]
    let oldEndVnode = oldCh[oldEndIdx]
    let newEndIdx = newCh.length - 1
    let newStartVnode = newCh[0]
    let newEndVnode = newCh[newEndIdx]
    let oldKeyToIdx, idxInOld, vnodeToMove, refElm

    // removeOnly is a special flag used only by <transition-group>
    // to ensure removed elements stay in correct relative positions
    // during leaving transitions
    const canMove = !removeOnly

    if (process.env.NODE_ENV !== 'production') {
      checkDuplicateKeys(newCh)
    }

    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (isUndef(oldStartVnode)) {
        oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
      } else if (isUndef(oldEndVnode)) {
        oldEndVnode = oldCh[--oldEndIdx]
      } else if (sameVnode(oldStartVnode, newStartVnode)) {  // 新旧头节点比较
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]
      } else if (sameVnode(oldEndVnode, newEndVnode)) { //  新旧尾节点比较
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right  // 旧头新尾比较
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left // 旧尾新头比较
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      } else {
        // 
        if (isUndef(oldKeyToIdx)) {
          // 创建一个 关于oldVnode的key 的哈希映射
         	oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx) 
        }
       //  根据newStartVnode的key是否存在 在oldKey的哈希映射中寻找对应的 节点
      //  idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
        if (isUndef(idxInOld)) { // New element  // 如果未找到 则创建新节点
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        } else {
          vnodeToMove = oldCh[idxInOld]
          if (sameVnode(vnodeToMove, newStartVnode)) {
            // 如果是相同的node则 执行patchVnode  以及insterBefore钩子 并将oldCh[idxInold]赋为undefined
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
            oldCh[idxInOld] = undefined
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
          } else {
            // key相同但是 是不同的节点 则以新节点对待
            // same key but different element. treat as new element
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
          }
        }
        newStartVnode = newCh[++newStartIdx]
      }
    }
    if (oldStartIdx > oldEndIdx) {
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
      addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {
      removeVnodes(oldCh, oldStartIdx, oldEndIdx)
    }
  }
```



## 工具函数

core/vdom/patch.js

### isPatchable

vnode的tag是否存在。 tag存在 应该是代表vnode是具名节点。 即"button", "div", "span"。etc

```javascript
function isPatchable (vnode) {
    while (vnode.componentInstance) {
      vnode = vnode.componentInstance._vnode
    }
    return isDef(vnode.tag)
  }
```

### createKeyToOldIdx

创建一个 hash映射 

```javascript
function createKeyToOldIdx (children, beginIdx, endIdx) {
  let i, key
  const map = {}
  for (i = beginIdx; i <= endIdx; ++i) {
    key = children[i].key
    if (isDef(key)) map[key] = i
  }
  return map
}
```

### findIdxInOld

在起始和结束范围之内能否找到与node 相同的节点 并返回 对应的index

```javascript
function findIdxInOld (node, oldCh, start, end) {
    for (let i = start; i < end; i++) {
      const c = oldCh[i]
      if (isDef(c) && sameVnode(node, c)) return i
    }
  }
```



## 注

### 关于hydrating的意思注解：

https://www.zhihu.com/question/66068748

### cbs&hooks

core/vdom/patch.js

```javascript
...
const hooks = ['create', 'activate', 'update', 'remove', 'destroy']
...

export function createPatchFunction (backend) {
  let i, j
  const cbs = {}

  const { modules, nodeOps } = backend

  for (i = 0; i < hooks.length; ++i) {
    cbs[hooks[i]] = []
    for (j = 0; j < modules.length; ++j) {
      if (isDef(modules[j][hooks[i]])) {
        cbs[hooks[i]].push(modules[j][hooks[i]])
      }
    }
  }
  ...
}
```

## 参考

https://juejin.cn/post/6844903607913938951#heading-11

https://github.com/answershuto/learnVue/blob/master/docs/VirtualDOM%E4%B8%8Ediff(Vue%E5%AE%9E%E7%8E%B0).MarkDown