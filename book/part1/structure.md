# 概览

分析代码之前肯定要弄清楚项目的目录结构，清楚每个文件是做什么的，从而对项目有个整体的认识。下面是vue2.x源码的整体目录结构。

```
├── BACKERS.md
├── LICENSE
├── README.md
├── benchmarks
│   ├── big-table
│   ├── dbmon
│   ├── reorder-list
│   ├── ssr
│   ├── svg
│   └── uptime
├── dist
│   ├── README.md
│   ├── vue.common.js
│   ├── vue.esm.browser.js
│   ├── vue.esm.js
│   ├── vue.js
│   ├── vue.min.js
│   ├── vue.runtime.common.js
│   ├── vue.runtime.esm.js
│   ├── vue.runtime.js
│   └── vue.runtime.min.js
├── examples
│   ├── commits
│   ├── elastic-header
│   ├── firebase
│   ├── grid
│   ├── markdown
│   ├── modal
│   ├── move-animations
│   ├── select2
│   ├── svg
│   ├── todomvc
│   └── tree
├── flow
│   ├── compiler.js         # 编译相关
│   ├── component.js·       # 组件数据结构
│   ├── global-api.js       # Global API 结构
│   ├── modules.js          # 第三方库定义
│   ├── options.js          # 选项相关
│   ├── ssr.js              # 服务端渲染相关
│   ├── vnode.js            # 虚拟 dom 相关
│   └── weex.js             # 平台相关
├── package.json
├── packages
│   ├── vue-server-renderer
│   ├── vue-template-compiler
│   ├── weex-template-compiler
│   └── weex-vue-framework
├── scripts
│   ├── alias.js            # 路径别名
│   ├── build.js            # 构建
│   ├── config.js           # 构建配置
│   ├── gen-release-note.js
│   ├── get-weex-version.js
│   ├── git-hooks           # git 钩子
│   ├── release-weex.sh
│   ├── release.sh          # 发布脚本
│   └── verify-commit-msg.js# 校验git commit脚本，规范化commit信息
├── src
│   ├── compiler
│   ├── core
│   ├── platforms
│   ├── server
│   ├── sfc
│   └── shared
├── test
│   ├── e2e
│   ├── helpers
│   ├── ssr
│   ├── unit
│   └── weex
├── types
│   ├── index.d.ts
│   ├── options.d.ts
│   ├── plugin.d.ts
│   ├── test
│   ├── tsconfig.json
│   ├── typings.json
│   ├── vnode.d.ts
│   └── vue.d.ts
└── yarn.lock
```

其中dist文件夹放置打包构建之后的文件，test测试文件夹，examples放置例子。由于vue项目使用flow来做静态类型检测，这里flow文件夹放置相关内容，types放置类型定义。我们把重点放在scripts文件夹和src文件夹上。

## Src

    src
    ├── compiler                # 编译相关代码，包括ast语法生成及其优化，代码生成等
    │   ├── codegen
    │   ├── create-compiler.js
    │   ├── directives
    │   ├── error-detector.js
    │   ├── helpers.js
    │   ├── index.js
    │   ├── optimizer.js
    │   ├── parser
    │   └── to-function.js
    ├── core                    # 核心代码，细讲
    │   ├── components          # 组件
    │   ├── config.js           # 配置文件
    │   ├── global-api          # 公共api
    │   ├── index.js            # 入口
    │   ├── instance            # vue 实例化
    │   ├── observer            # 数据观察
    │   ├── util                # 工具函数
    │   └── vdom                # virtual dom
    ├── platforms               # 不同平台的支持，分为web和weex
    │   ├── web                 # 浏览器web端
    │   └── weex                # 可以理解为native端，本文不考虑这部分
    ├── server                  # 服务端渲染，采用ssr方式的解决方案
    │   ├── bundle-renderer
    │   ├── create-basic-renderer.js
    │   ├── create-renderer.js
    │   ├── optimizing-compiler
    │   ├── render-context.js
    │   ├── render-stream.js
    │   ├── render.js
    │   ├── template-renderer
    │   ├── util.js
    │   ├── webpack-plugin
    │   └── write.js
    ├── sfc                     # 负责.vue 文件解析,生成javascript对象
    │   └── parser.js
    └── shared                  # 共享代码，共同常量和工具函数
        ├── constants.js
        └── util.js

可以知道，vue的核心就在core文件夹中，这也是本文需要细讲和实现的核心部分。

## Scripts

scripts 文件夹是放置构建项目的构建入口，构建脚本所在地。负责根据不同命令构建不同环境下的代码放置到dist目录。其核心是`build.js`, `alias.js`, `config.js`。其详细内容看下一章[构建](/book/part1/build.md)