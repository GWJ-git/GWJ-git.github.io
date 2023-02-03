---
title: npm相关学习
date: 2022-10-20T20:39:10+08:00
draft: false
toc: true
images: null
categories:
  - 学习笔记
tags:
  - npm
  - '学习笔记'
slug: ''
---
# npm 生态


## npm,cnpm,yarn,pnpm

**node**的包管理工具,用于**node**插件管理（包括安装、卸载、管理依赖等）

```kotlin
//递归依赖
├── node_modules
│   ├── A@1.0.0
│   │   └── node_modules
│   │   │   └── D@1.0.0
│   ├── B@1.0.0
│   │   └── node_modules
│   │   │   └── D@2.0.0
│   └── C@1.0.0
│   │   └── node_modules
│   │   │   └── D@2.0.0
--------------------
// 扁平化依赖
├── node_modules
│   ├── A@1.0.0
│   │   └── node_modules
│   │   │   └── D@1.0.0
│   ├── B@1.0.0
│   ├── C@1.0.0
│   └── D@2.0.0

```



### npm ,yarn

`npm`为`node`自带，npm1,2采用递归依赖，npm3开始采用扁平化依赖

npm原有问题：

1. 安装速度慢
2. 可能出现相同模块大量重复冗余
3. 不支持离线安装



最新版的`npm`和`yarn`安装速度和使用体验并没有多大的差距，`yarn`稍好一些



`npm3+`和`yarn`

1. 依赖结构的**不确定性**。
2. 扁平化算法本身的**复杂性**很高，耗时较长。
3. 项目中仍然可以**非法访问**没有声明过依赖的包(**幽灵依赖**)
4. 可能会导致有大量的依赖的被重复安装

### cnpm

淘宝镜像 国内下载速度快
构建私人npm cli



### pnpm

网状 + 平铺的node_modules结构

store（基于内容寻址**CAS**,**Virtual store**）+link(**symbolic link**)



1. 速度快
2. 高效利用磁盘空间。利用硬链接和符号链接来避免复制所有本地缓存源文件。
3. 安全性高。避免了非法访问



### ni

> ~~*`npm i` in a yarn project, again? F**k!*~~
>
> **ni** - use the right package manager // 使用正确的包管理器

ni假设使用锁文件(而且您应该使用)

在运行之前，它将检测您`yarn.lock` / `pnpm-lock.yaml` / `package-lock.json` / `bun.lockb`来了解当前的包管理器(如果指定，则在 packages.json 中指定 `packageManager` 字段) ，并运行相应的命令。

#### install

```sh
ni

# npm install
# yarn install
# pnpm install
# bun install
---------------------------------------
ni vite

# npm i vite
# yarn add vite
# pnpm add vite
# bun add vite
--------------------------------------
ni @types/node -D

# npm i @types/node -D
# yarn add @types/node -D
# pnpm add -D @types/node
# bun add -d @types/node
------------------------------------
ni --frozen

# npm ci
# yarn install --frozen-lockfile (Yarn 1)
# yarn install --immutable (Yarn Berry)
# pnpm install --frozen-lockfile
# bun install --no-save
-----------------------------------
ni -g eslint

# npm i -g eslint
# yarn global add eslint (Yarn 1)
# pnpm add -g eslint
# bun add -g eslint

# this uses default agent, regardless your current working directory
```

#### run

```shell
nr dev --port=3000

# npm run dev -- --port=3000
# yarn run dev --port=3000
# pnpm run dev -- --port=3000
# pnpm dev pnpm可以别名运行
# bun run dev --port=3000
----------------
nr

# interactively select the script to run
# supports https://www.npmjs.com/package/npm-scripts-info convention
----------------
nr -

# rerun the last command 重新执行最后一次执行的命令
```

#### execute

```shell
nx vitest

# (not available for bun)
# npx vitest
# yarn dlx vitest
# pnpm dlx vitest
```

#### upgrade

```shell
nu

# (not available for bun)
# npm upgrade
# yarn upgrade (Yarn 1)
# yarn up (Yarn Berry)
# pnpm update
-----------------------
nu -i

# (not available for npm & bun)
# yarn upgrade-interactive (Yarn 1)
# yarn up -i (Yarn Berry)
# pnpm update -i
```

####  uninstall

```shell
nun webpack

# npm uninstall webpack
# yarn remove webpack
# pnpm remove webpack
# bun remove webpack
--------------------
nun -g silent

# npm uninstall -g silent
# yarn global remove silent
# pnpm remove -g silent
# bun remove -g silent
```

#### clean install

```she
nci

# npm ci
# yarn install --frozen-lockfile
# pnpm install --frozen-lockfile
# bun install --no-save
```

如果没有相应的node管理器，这个命令将在全局范围内安装它

#### agent alias

```shell
na

# npm
# yarn
# pnpm
# bun
------------------------
na run foo

# npm run foo
# yarn run foo
# pnpm run foo
# bun run foo
```

#### Change Directory

```shell
ni -C packages/foo vite
nr -C playground dev
```

#### Config

```
; ~/.nirc

; fallback when no lock found
defaultAgent=npm # default "prompt"

; for global installs
globalAgent=npm
----------------------
# ~/.bashrc

# custom configuration file path
export NI_CONFIG_FILE="$HOME/.config/ni/nirc"
```



## package.json

```json
{
  "name": "",
  "version": "2.6.0",
  "description": "",
  "author": "",
  "license": "Apache-2.0",
  "scripts": {
    "dev": "vue-cli-service serve",
    "build:prod": "vue-cli-service build",
    "build:stage": "vue-cli-service build --mode staging",
    "preview": "node build/index.js --preview",
    "lint": "eslint --ext .js,.vue src",
    "test:unit": "jest --clearCache && vue-cli-service test:unit",
    "svgo": "svgo -f src/assets/icons/svg --config=src/assets/icons/svgo.yml",
    "new": "plop"
  },
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged"
    }
  },
  "lint-staged": {
    "src/**/*.{js,vue}": [
      "eslint --fix",
      "git add"
    ]
  },
  "repository": {
    "type": "git",
    "url": ""
  },
  "bugs": {
    "url": ""
  },
  "dependencies": {
    "@riophae/vue-treeselect": "^0.4.0",
    "axios": "^0.21.1",
    "clipboard": "2.0.4",
    "codemirror": "^5.49.2",
    "core-js": "^2.6.12",
    "echarts": "^4.2.1",
    "echarts-wordcloud": "^1.1.3",
    "element-ui": "^2.15.8",
    "file-saver": "1.3.8",
    "fuse.js": "3.4.4",
    "js-beautify": "^1.10.2",
    "js-cookie": "2.2.0",
    "jsencrypt": "^3.0.0-rc.1",
    "jszip": "^3.7.1",
    "mavon-editor": "^2.9.1",
    "normalize.css": "7.0.0",
    "nprogress": "0.2.0",
    "path-to-regexp": "2.4.0",
    "qs": "^6.10.1",
    "screenfull": "4.2.0",
    "sortablejs": "1.8.4",
    "vue": "^2.6.14",
    "vue-count-to": "^1.0.13",
    "vue-cropper": "0.4.9",
    "vue-echarts": "^5.0.0-beta.0",
    "vue-image-crop-upload": "^2.5.0",
    "vue-router": "3.0.2",
    "vue-splitpane": "1.0.4",
    "vuedraggable": "2.20.0",
    "vuex": "3.1.0",
    "wangeditor": "^4.7.11",
    "xlsx": "^0.17.4"
  },
  "devDependencies": {
    "@babel/parser": "^7.7.4",
    "@babel/register": "7.0.0",
    "@vue/babel-plugin-transform-vue-jsx": "^1.2.1",
    "@vue/cli-plugin-babel": "3.5.3",
    "@vue/cli-plugin-eslint": "^3.9.1",
    "@vue/cli-plugin-unit-jest": "3.5.3",
    "@vue/cli-service": "3.5.3",
    "@vue/test-utils": "1.0.0-beta.29",
    "autoprefixer": "^9.5.1",
    "babel-core": "7.0.0-bridge.0",
    "babel-eslint": "10.0.1",
    "babel-jest": "23.6.0",
    "babel-plugin-dynamic-import-node": "2.3.0",
    "babel-plugin-transform-remove-console": "^6.9.4",
    "chalk": "2.4.2",
    "chokidar": "2.1.5",
    "connect": "3.6.6",
    "compression-webpack-plugin": "5.0.2",
    "eslint": "5.15.3",
    "eslint-plugin-vue": "5.2.2",
    "html-webpack-plugin": "3.2.0",
    "http-proxy-middleware": "^0.19.1",
    "husky": "1.3.1",
    "lint-staged": "8.1.5",
    "plop": "2.3.0",
    "sass": "1.32.13",
    "sass-loader": "10.2.0",
    "script-ext-html-webpack-plugin": "2.1.3",
    "script-loader": "0.7.2",
    "serve-static": "^1.13.2",
    "svg-sprite-loader": "4.1.3",
    "svgo": "1.2.0",
    "tasksfile": "^5.1.1",
    "vue-template-compiler": "2.6.14"
  },
  "engines": {
    "node": ">=8.9",
    "npm": ">= 3.0.0"
  },
  "browserslist": [
    "> 1%",
    "last 2 versions"
  ]
}

```

- `name`:项目名
- `version`：包唯一的版本号
- `resolved`：安装源
- `integrity`：表明包完整性的hash值（验证包是否已失效）
- `dev`：如果为true，则此依赖关系仅是顶级模块的开发依赖关系或者是一个的传递依赖关系
- `repository`：项目的仓库地址以及版本控制信息
- `description`：项目描述
- `keywords`：技术关键词
- `homepage`：主页链接
- `license`：开源许可证
- `author`：作者
- `files`：指定发布的内容
- `type`：声明npm包遵循的模块化规范 `type: "module"||"commonjs" `

  - 不指定，默认为`commonjs`
  - `module` => `ESModule`
  - 所有`.js`后缀结尾的文件，都遵循`type`所指定的模块化规范
  - 以`.mjs`结尾的文件就是使用的ESModule规范，以`.cjs`结尾的遵循的是`commonjs`规范

- `main`：指定项目入口文件，不设置 `main` 字段，入口文件为根目录下的 `index.js`。browser 和 Node 环境
- `module`：定义 npm 包的 `ESM` 规范的入口文件
- `browser`：指定browser环境项目入口文件

  - ```json
    "browser": {
        "./dist/index.js": "./dist/index.browser.js", // browser+cjs
        "./dist/index.mjs": "./dist/index.browser.mjs"  // browser+mjs
    }
    ---------------------
    "browser": "./dist/index.browser.js" // browser

- `scripts`：脚本命令
- `dependencies`：依赖包`node_modules`中依赖的包，与顶层的`dependencies`一样的结构
- `dependencies `：-运行依赖，项目生产环境下需要用到的依赖
- `devDependencies`： 开发依赖，项目开发环境需要用到而运行时不需要的依赖，用于辅助开发，通常包括项目工程化工具比如 webpack，vite，eslint 等。
- `peerDependencies` ： 同伴依赖，一种特殊的依赖，不会被自动安装，通常用于表示与另一个包的依赖与兼容性关系来警示使用者。
- `bundledDependencies` 打包依赖，在发布包时，bundleDependencies 里面的依赖都会被一起打包
- `optionalDependencies` ：可选依赖
- `browserslist`：设置项目的浏览器兼容情况
- `unpkg`:开启 CDN 服务
- `requires`：依赖包所需要的所有依赖项，对应依赖包`package.json`里`dependencies`中的依赖项

## 本地包调试

使用npm link/yarn link/yalc 等工具，通过软链接的方式进行调试
 原理：

- 在全局包路径（Global Path）下创建一个软连接(Symlinked)指向本地包的dist包;
- 在测试项目里通过软连接，将全局的软链接指向其`node_modules/xxxxx` 

### npm link/yarn link

```bash
# 第一步 在本地包中执行：
npm link
# or
yarn link
# 第二步 在测试项目中中执行：
npm link <packageName>
# or
yarn link <packageName>
```

###  yalc

#### 安装

```
npm i yalc -g
# or
yarn global add yalc
```

#### 使用

```bash
# 本地包发布
yalc publish

# 调试项目中添加
yalc add <packageName>

# 调试项目中导入
# import { Xxx } from 'packageName';

# 移除依赖
yalc remove <packageName>

# 更新部署
yalc publish --push
# 简化为：
yalc push
```

优点：

- 原有依赖被缓存，依赖移除后即还原
- HRM

## webpack-bundle-analyzer

包分析工具，查看项目打包情况，每个包的体积

### 安装

```shell
npm install webpack-bundle-analyzer --save-dev
```

### 配置webpack.config.js文件

```js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin

module.exports={
  plugins: [
    // ... 其他 plugin 配置
    new BundleAnalyzerPlugin()  // 使用默认配置
  ]
}
```

### package.json

```json
"scripts": {
    "analyzer": "cross-env ENV_TYPE=staging ANALYZER=true node scripts/build.js"
}
```

### 执行

`npm run analyzer`

## rollup-plugin-visualizer

结合vite

### 安装

```sh
npm install --save-dev rollup-plugin-visualizer
```

### vite.config.js

```js
const { visualizer } = require("rollup-plugin-visualizer");
// ....
export default defineConfig({
  plugins: [
      vue(),
      visualizer({
        open:true,  //注意这里要设置为true，否则无效
        gzipSize:true,
        brotliSize:true
      }),
      // ...其他插件
           ]
})
```





[package.json](https://juejin.cn/post/7145001740696289317)

[yacl](https://juejin.cn/post/7033400734746066957)
