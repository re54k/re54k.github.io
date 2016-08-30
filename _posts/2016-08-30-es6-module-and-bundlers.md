---
layout:     post
title:      ECMAScript 6 module and bundlers
date:       2016-08-30 12:26
summary:    随着 ECMAScript 6 的普及，ES6 module 相比 CommonJS 有哪些新特新？针对这些特性，又有哪些模块打包工具做了相应的适配？
categories: original
keywords:   ECMAScript 6、module、tree-shaking、rollup、webpack
---

无论什么开发语言，当逻辑变的复杂，规模变的庞大之时，都应有 module/package 体系支持，浏览 Javascript 的发展史，模块化之路可谓坎坷艰难。

---

## Module

### Stone age

为了封装我们写自执行函数：

``` js
var exports = function(){
    var foo = 'foo';
    return {
        bar: foo + ' bar',
        ...
    }
}();
```

为了避免冲突我们使用对象作为命名空间：

``` js
global.defaults.date = '2016-01-01';
global.util.date = function() {
    return ...
}
```

为了“导入模块”我们利用参数：

``` js
!function($) {
    $.ajax(...);
}(jQuery);
```

### CommonJS

直到 commonJS 的诞生，javascript 模块化才算正式登上舞台。但是 commonJS 规范源于服务端，浏览器端是没有模块作用域的（默认挂载到 `window` 上），而且文件请求全都需要通过 HTTP 请求，于是经过“进化”，有了浏览器端专用的 AMD/CMD 规范。配合打包工具（bundler），我们才可以真正使用模块化方式进行开发。这些内容已经相当成熟，不再赘述。

``` js
var _ = require('lodash');

module.exports = {
    escape: _.escape,
    unescape: _.unescape,
    ...
};
```

### ECMAScript 6 module

ES6 模块保留了 commonJS 和 AMD 的一些思想，实现的更为简洁，同时支持同步和异步加载，而且可以静态解析，意在成为服务器端和浏览器端的通用解决方案。如果标准规范得到统一支持，也意味着：

- 再也不用借助 UMD （[Universal Module Definition](https://github.com/umdjs/umd)）适配多个模块系统。
- 新的浏览器 API 可以封装成模块提供，不必再做为全局变量或者 `navigator` 对象的属性。
- 无需利用对象充当命名空间，像 ES5 中的 `Math`、`JSON` 类似的功能可以通过模块提供。

ES6 模块标准分为两个部分：

- 声明式语法（Declarative syntax），针对模块内部的 `import` 和 `export`。
- 程序化加载 API（Programmatic loader API），用于外部对模块的加载使用。该部分尚在标准协会的讨论中。

语法部分，阮一峰老师的 [ES6 系列教程](http://es6.ruanyifeng.com/#docs/module) 里已经讲的相当详尽，我就不再搬运了。为了便于 bundlers 部分的理解，这里重复几个要点：

- **ES6 模块可以静态解析，在编译时就能确定模块的依赖关系，以及输入和输出的变量。**
- `import { square, diag } from 'lib';` 是 `import` 的特有语法，并不是 destructing 语法。
- ES6 模块输出的是值的引用（binding）而非拷贝。
- 需要 `<module>` 标签来嵌入到 HTML 中，或者使用 polyfill `<script type="module">` 来替代。

象征性的来个代码示例：

``` js
// underscore.js
export default function (obj) {
    ...
};
export function each(obj, iterator, context) {
    ...
}

// main.js
import _, { each } from 'underscore';
...
```

`import` 必须**静态**声明依赖，但很多时候我们希望能**动态**的加载模块，比如在代码中满足一定条件后再加载某个模块，所以需要一套加载 API 来：

- 在 `<script>` 中加载并使用模块。
- 对模块加载进行配置。

ES6 规定了构造器及一些接口来解析模块标示符和加载模块，交由各平台维护一个自定义实例实现其中的细节，并挂载到全局变量 `System` 上。下边是一个简单的 `import` 方法示例：

``` js
if ( someFeatureNotSupported ) {
  System.import('my-polyfill').then(myPolyFill => {
    // use the module from here
  });
}
```

另外，加载 API 提供了多种配置选项（还没出细则）对加载的模块在使用前进行转换（参考 webpack 的 loader）。比如在导入的时候使用 JSLint/JSHint 校验模块；进行 coffeeScript/typeScript 类似语法的转换；导入其它语法的模块（AMD/NodeJS）。

总的来说，**ES6 模块的语法部分已经完全标准化，也逐步成为通用的书写格式；但如何加载模块的标准还在进行中，而且 JS 引擎还没有原生支持**，所以目前仍不能在浏览器中直接使用 ES6 模块，需要借助工具进行转换。那么接下来我们就聊聊比较实用的打包工具（bundler）。

---

## Bundlers

目前热门的打包工具对 commonJS/AMD/CMD 规范的代码支持都比较成熟，这里主要说下 [rollup](http://rollupjs.org/) 和 webpack 2，这两个工具使用 tree-shaking 技术对于 ES6 模块规范的代码能发挥更优秀的打包效果。

### Tree-shaking

[Rich Harris](https://medium.com/@Rich_Harris) 在他开发的 rollup 中推广了一个重要特性 -- *tree-shaking*：仅打包模块中需要使用的代码（Only includes the bits of code your bundle actually needs to run）。虽然目标和 DCE（Dead Code Elimination）一致，都是删除死码，但作者在 [Tree-shaking versus dead code elimination](https://medium.com/@Rich_Harris/tree-shaking-versus-dead-code-elimination-d3765df85c80#.fz0rc29bu) 一文中解释了两者的区别。**针对模块打包而言，比起打包后再移除死码，打包前通过静态分析，仅打包需要的代码更符合逻辑，而且通常也会得到更好的输出结果。**

Tree-shaking 带来了一个很大的便利：我们可以随意的维护模块代码，不用担心打包后的体积问题。比如维护一份功能丰富的 *util* 文件，在任何项目中使用而不再需要手动清理不用的方法。

但 tree-shaking 并不完善，仅处理了顶层 AST 结点（对应 ES6 模块 `export` 声明必须位于模块顶层），并没有深入更深层的部分，所以打包后的代码依旧会比理想情况下需要的代码多，而且它并不处理对象中不用的代码，所以作者建议：**先使用 rollup 再通过 UglifyJS 压缩**。

这里引用[知乎回复](http://www.zhihu.com/question/41922432/answer/93068390)中的一段话，这也是前边强调 ES6 模块几个要点的原因：

> 正是基于这个基础上（ES6 模块静态分析），才使得 tree-shaking 成为可能（这也是为什么 rollup 和 webpack 2 都要用 ES6 module syntax 才能 tree-shaking），所以说与其说 tree-shaking 这个技术怎么了不起，不如说是 ES6 module 的设计在模块静态分析上的种种考量值得赞赏。

### Rollup

Tree-shaking 无疑是 rollup 最大的亮点，通常打包后的 bundle 会更小一些。合并的模块会被 flatten 后放在同一个作用域下，代码更加直观，可读性高一些；而且直接通过变量名就可以访问用各模块接口，性能有少许提升。

``` js
// add.js
const base = 1;
export function add(x) {
	return x + base;
}

export function unused() {
	console.log('This function will be eliminated');
}

// double.js
import { add } from './add';
export function compute(x) {
	return add(x) * 2;
};

// main.js
import { compute } from './double';
console.log( compute(1) );
```

经过 rollup 打包：

``` js
(function () {
	'use strict';

	const base = 1;
	function add(x) {
		return x + base;
	}

	function compute(x) {
		return add(x) * 2;
	}

	console.log( compute(1) );

}());
```

对比目前的各种打包工具来看，rollup：

- 功能比较纯粹，没有眼花缭乱的配置项，对于想快速搭建一套简单的打包系统是一个好选择。
- 可以将代码打包成任意格式（amd/cjs/es6/iife/umd），非常适合库和插件的打包。
- 提供了丰富的插件，可以对多种格式/规范的代码进行转换编译（类似于 webpack 的 *loader*），所以不用担心对接的问题。

### Webpack 2

Webpack 2 中有很多让人期待的新特性和修改，而且针对诟病的复杂配置也做了一定简化，可以从 [这篇文章](https://gist.github.com/sokra/27b24881210b56bbaff7) 看到较为详细的介绍。对于本文，我们比较关心的主要是下边三点：

- 原生支持 ES6 模块，能直接处理 `import` 和 `export`。
- 支持 tree-shaking。
- 使用 `System.import` 动态加载 ES6 模块；同时也作为拆分点，把请求的模块处理成独立 chunk（等同于之前的 async trunk）。忍不住感叹下，真心比 `require.ensure` 清爽多了，而且还能捕捉加载失败。

``` js
function onClick() {
    System.import("./module").then(module => {
        module.default;
    }).catch(err => {
        console.log("Chunk loading failed");
    });
}
```

先看下 webpack 2 是怎么实现 tree-shaking 的，分两步：

 1. 将所有依赖的 ES6 模块文件合并到一个单独文件，对于其中的每个模块，将那些被 `import` 的属性挂载到 `exports` 对象上。
 2. 利用代码压缩，移除死码。这样，既没有被导出也没有在内部使用的代码就被移除了。

``` js
// helper.js
export function foo() {
    return 'foo';
}
export function bar() {
    return 'bar';
}

// main.js
import { foo } from './helper';
...
```

上边代码在 webpack 2 编译后输出的 bundle 中， helper 模块代码如下：

``` js
function(module, exports, __webpack_require__) {
    /* harmony export */ exports["foo"] = foo;
    /* unused harmony export bar */;

    function foo() {
        return 'foo';
    }
    function bar() {
        return 'bar';
    }
}
```

可以看出 `exports` 中已经没有了 `bar` 这个方法，`bar` 既没有被使用，也没有被导出，成了死码，最后经过压缩被移除。

``` js
function (t, n, r) {
    function e() {
        return "foo"
    }

    n.foo = e
}
```

对比 rollup 和 webpack 2 来看，rollup 不包装模块，代码可读性高，不实现模块加载器（`require`），支持多格式输出，对于打包类库/插件是比较好的选择。而 webpack 2 凭借丰富的配置和loader，支持代码拆分（async trunk），资源模块化，热替换，实用的插件如 *CommonsChunkPlugin* 等等，在打包 webApp 方面更胜一筹。

---

## 参考及引用的文章

- [JS 模块化历程](http://www.cnblogs.com/lvdabao/p/js-modules-develop.html)
- [ECMAScript 6 入门](http://es6.ruanyifeng.com/#docs/module)
- [深入浅出ES6（十六）：模块 Modules](http://www.infoq.com/cn/articles/es6-in-depth-modules?utm_source=tuicool&utm_medium=referral)
- [ECMAScript 6 modules: the final syntax](http://www.2ality.com/2014/09/es6-modules-final.html)
- [JavaScript Modules the ES6 Way](https://24ways.org/2014/javascript-modules-the-es6-way/)
- [The future of bundling JavaScript modules](http://www.2ality.com/2015/12/bundling-modules-future.html)
- [如何评价 Webpack 2 新引入的 Tree-shaking 代码优化技术？](http://www.zhihu.com/question/41922432/answer/93068390)
- [Tree-shaking versus dead code elimination](https://medium.com/@Rich_Harris/tree-shaking-versus-dead-code-elimination-d3765df85c80#.a2067lvow)
- [Tree-shaking with webpack 2 and Babel 6](http://www.2ality.com/2015/12/webpack-tree-shaking.html)