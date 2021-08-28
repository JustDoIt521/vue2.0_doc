# AST 解析 及 render函数

## $mount 	

src/platforms/web/entry-runtime-with-compiler.js

```javascript
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
		....

      const { render, staticRenderFns } = compileToFunctions(template, {
        outputSourceRange: process.env.NODE_ENV !== 'production',
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      options.render = render
      options.staticRenderFns = staticRenderFns

      /* istanbul ignore if */
     ...
  return mount.call(this, el, hydrating)
}
```

## 方式一 template是 string



## 方式二 template 不是 string

### getOuterHTML

src/platforms/web/entry-runtime-with-compiler.js

获取了该dom节点及后代dom节点。 如果没有dom节点 则创建一个空的dom节点

```javascript
function getOuterHTML (el: Element): string {
  if (el.outerHTML) {
    return el.outerHTML
  } else {
    const container = document.createElement('div')
    container.appendChild(el.cloneNode(true))
    return container.innerHTML
  }
}
```

### compileToFunctions

src/platforms/web/compiler/index.js

```javascript
/* @flow */

import { baseOptions } from './options'
import { createCompiler } from 'compiler/index'

const { compile, compileToFunctions } = createCompiler(baseOptions)

export { compile, compileToFunctions }
```

### baseOptions

src/platforms/web/compiler/baseOptions.js

```javascript
import {
  isPreTag,
  mustUseProp,
  isReservedTag,
  getTagNamespace
} from '../util/index'

import modules from './modules/index'
import directives from './directives/index'
import { genStaticKeys } from 'shared/util'
import { isUnaryTag, canBeLeftOpenTag } from './util'

export const baseOptions: CompilerOptions = {
  expectHTML: true,
  modules,
  directives,
  isPreTag,
  isUnaryTag,
  mustUseProp,
  canBeLeftOpenTag,
  isReservedTag,
  getTagNamespace,
  staticKeys: genStaticKeys(modules)
}
```

#### modules

src/platforms/web/compiler/modules/index.js

```javascript
import klass from './class'
import style from './style'
import model from './model'

export default [
  klass,
  style,
  model
]

```

##### klass

src/platforms/web/compiler/modules/class.js

添加删除dom的 class。 包括静态的class （class="xxx"） 和 动态的class （:class="{xxx}"）

##### style

src/platforms/web/compiler/modules/style.js

添加删除dom的style。包含静态的style （style="xxxx" ）和动态的style （:style="")

##### model

src/platforms/web/compiler/modules/model.js

元素的 v-bind v-for v-if v-else等属性

### createCompiler

src/compiler/index.js

```javascript
// `createCompilerCreator` allows creating compilers that use alternative
// parser/optimizer/codegen, e.g the SSR optimizing compiler.
// Here we just export a default compiler using the default parts.
export const createCompiler = createCompilerCreator(function baseCompile (
  template: string,
  options: CompilerOptions
): CompiledResult {
  const ast = parse(template.trim(), options)
  if (options.optimize !== false) {
    optimize(ast, options)
  }
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
}) 
```

## 