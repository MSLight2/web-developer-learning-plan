## vue模板编译流程
### 1.编译器的创建
```js
// --------------render函数的生成（简化版）-----------------

// entry-runtime-with-compiler.js
const { render, staticRenderFns } = compileToFunctions(template, option, vm)

// platform/web/compiler/index.js
const { compile, compileToFunctions } = createCompiler(baseOptions)
export { compile, compileToFunctions }

// compiler/index.js  一个工厂函数（'编译器函数创建者'的创建者）。可以根据需求定制生成不同平台的代码
export const createCompiler = createCompilerCreator(function baseCompile (
  template: string,
  options: CompilerOptions
) {
  // 这里的options就是baseOptions和compileToFunctions传入的options合并融合后的finalOptions
  const ast = parse(template.trim(), options) // html模板转AST
  if (options.optimize !== false) {
    optimize(ast, options)
  }
  const code = generate(ast, options) // 根据AST生成对应的平台代码
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
}

// compiler/create-compiler.js
export function createCompilerCreator (baseCompile: Function): Function {
  return function createCompiler (baseOptions: CompilerOptions) {
    // 这个options就是compileToFunctions传入的options配置
    function compile (template, options?: CompilerOptions) {
      // ...
      const compiled = baseCompile(template.trim(), finalOptions)
      // ...
      // 一些错误处理等
    }
    return {
      compile,
      compileToFunctions: createCompileToFunctionFn(compile)
    }
  }
}

// compiler/to-function.js
export function createCompileToFunctionFn (compile: Function): Function {
  // 缓存编译内容，防止重复编译，提高性能
  const cache = Object.create(null)

  return function compileToFunctions () {
    // ...
    // 一些错误处理等
    return (cache[key] = res)
  }
}
```
>`createCompilerCreator`调用返回`createCompiler`函数，`createCompiler`函数调用返回`compiled`编译方法和`compileToFunctions`编译方法。`compile`方法编译里会调用`createCompilerCreator`方法传入的`baseCompile`进行HTML模板编译

>`createCompiler`返回的就是编译器，它本身就是编译器创建者。

> **在创建编译器的时候传递了基本编译器选项参数，当真正使用编译器编译模板时，依然可以传递编译器选项，并且新的选项和基本选项会以合适的方式融合或覆盖**