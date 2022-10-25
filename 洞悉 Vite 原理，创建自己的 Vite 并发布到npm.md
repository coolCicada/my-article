> 代码已上传 github 和 npm


[github地址](https://github.com/coolCicada/cool-vite)
[npm地址](https://www.npmjs.com/package/cool-vite)

# 简述
```
基于 esModule 的 Vite 现在是很火的技术，我们来探究下它的实现原理
```

# 使用方法

## 下载

可全局安装
```
npm install cool-vite
```

## 启动

```
npx cool-vite
```

# 流程梳理
```
Vite 本质上是一个本地服务器
因为 浏览器已原生支持 ES 模块，所以根据浏览器的请求进行处理，返回处理后的文件，就能实现一个简单的 Vite
```
**具体流程**
1. 使用 koa 搭建本地服务
2. 拦截请求 使用不同文件对应的 plugin 进行处理
3. 返回 plugin 处理后的文件

# 代码步骤

## 项目结构

![1666683858515-image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ca88a75214474ef399334c22d25f2abf~tplv-k3u1fbpfcp-watermark.image?)

## 1. 使用 KOA 搭建文件服务器
`index.js`
```javascript
#!/usr/bin/env node
const Koa = require('koa');
const app = new Koa();

app.use(async (ctx) => {
  const { url } = ctx.request;
  const content = url;
  ctx.body = content;
});

app.listen(3000, () => {
  console.log('start at 3000');
});
```

## 2. 插件入口
```
为了解耦，使用职责链模式，将所有插件放到数组里，规则配置使用插件，规则不匹配跳过插件
```
`lib/plugin.js`
```javascript
const plugins = [];

function addPlugin(...fn) {
  plugins.push(...fn);
}

function execute(url) {
  const first = plugins[0];
  let index = 0;
  function next () {
    index ++;
    if (index === plugins.length) return {};
    return plugins[index](url, next);
  }
  return first(url, next);
}

module.exports = {
  addPlugin,
  execute,
};
```

插件返回格式为 `{ content: '', type: '' }` content 为模块内容， type 为 格式 默认为 `'application/javascript'`

## 3. 插件执行

`index.js`

```javascript
#!/usr/bin/env node
const Koa = require('koa');
const app = new Koa();

const { addPlugin, execute } = require('./lib/plugins');
addPlugin(
  require('./lib/plugin/htmlPlugin'),
  require('./lib/plugin/jsPlugin'),
  require('./lib/plugin/modulePlugin'),
  require('./lib/plugin/vuePlugin'),
  require('./lib/plugin/cssPlugin'),
);

app.use(async (ctx) => {
  const { url } = ctx.request;
  const { content, type } = execute(url);
  ctx.type = type ? type : 'application/javascript';
  ctx.body = content;
});

app.listen(3000, () => {
  console.log('start at 3000');
});
```

```
我们只需要返回插件返回的结果就行了，执行交给插件
```

## 4. html插件

`lib/plugin/htmlPlugin.js`

```javascript
const fs = require('fs');
const path = require('path');

module.exports = function(url, next) {
  if (!(url === '/' || url.endsWith('.html'))) {
    return next();
  }
  
  if (url === '/') url = '/src/index.html';

  let content = fs.readFileSync(path.resolve() + url, 'utf-8');
  content = content.replace(
    '<script ', 
    `<script>window.process = { env: { NODE_ENV: 'dev' }}</script><script `
  );

  return {
    content: content,
    type: 'text/html'
  };
}
```

其实就是简单的根据请求路径读取文件，然后返回，中间有块逻辑:
```javascript
content = content.replace( '<script ', 
`<script>window.process = { env: { NODE_ENV: 'dev' }}</script><script ` );
```
后续引入的库会有 读 process 变量，先处理下

## 5. js插件

`lib/plugin/jsPlugin.js`

```javascript
const fs = require('fs');
const path = require('path');
const { rewriteImport } = require('../utils');

module.exports = function(url, next) {
  if (!url.endsWith('.js')) return next();
  return {
    content: rewriteImport(fs.readFileSync(path.resolve() + url, 'utf-8'))
  };
}
```
rewriteImport 方法

`lib/utils.js`

```javascript
function rewriteImport(content) {
  return content.replace(/ from ['|"]([^'"]+)['|"]/g, function(s0, s1) {
    if (s1[0] !== '.' && s1[0] !== '/') {
      return ` from '/@modules/${s1}'`;
    } else {
      return s0;
    }
  });
}

module.exports = {
  rewriteImport,
};
```

对引入 `node_modules` 的库进行处理，加前缀，方便我们写插件拦截

## 6. vue 插件

`lib/plugin/vuePlugin.js`

*思路*
- 使用 `@vue/compiler-sfc` 把 .vue 文件解析，让我们可以获取 `template` `script` `css` 三部分
- 为了解耦，把一个对 .vue文件的请求分为三部分
    - **.vue** `script` 部分
    - **.vue?type=template** `template` 部分 也就是 render 函数
    - **.vue?type=css `css`** 部分
- 对于 `script` 部分 我们对源文件进行重写，添加对`template` `css` 的引用
- 对于 `template` `css` 部分 我们使用 vue 提供的 compiler 库进行处理并返回

```javascript
const fs = require('fs');
const path = require('path');
const qs = require('qs');
const compilerSFC = require('@vue/compiler-sfc');
const compilerDOM = require('@vue/compiler-dom');
const { rewriteImport } = require('../utils');

module.exports = function (url, next) {
  if (url.indexOf('.vue') === -1) return next();
  const [fileName, arg] = url.split('?');
  const p = path.resolve() + '/' + fileName.slice(1);
  const { descriptor } = compilerSFC.parse(fs.readFileSync(p, 'utf-8'));
  if (arg) {
    const obj = qs.parse(arg);
    if (obj.type === 'template') {
      const template = descriptor.template;
      const render = compilerDOM.compile(template.content, { mode: 'module' }).code;
      return {
        content: rewriteImport(render),
      };
    }
  }
  return {
    content: `${rewriteImport(descriptor.script.content).replace('export default ', 'const __script = ')}
      import { render as __render } from '${url}?type=template';
      __script.render = __render;
      export default __script;
    `
  };
}

```

## 7. css 插件

`lib/plugin/cssPlugin.js`

就是包装成js模块 执行这个模块 可以把 `style` `append` 到 `body` 上

```javascript
const fs = require('fs');
const path = require('path');

module.exports = function (url, next) {
  if (!url.endsWith('.css')) return next();
  const p = path.resolve() + '/' + url;
  const content = fs.readFileSync(p, 'utf-8');
  const res = `
    const css = '${content.replace(/\n/g, "")}';
    const link = document.createElement('style');
    link.setAttribute('type', 'text/css');
    document.head.appendChild(link);
    link.innerHTML = css;
    export default css;
  `;
  return { content: res };
}
```
# 项目发布

## 登录

```
npm login
```

## 上传

```
npm publish
```



