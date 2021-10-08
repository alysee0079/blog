# webpack

### loader：

css：style-loader css-loader (less-loader，sass-loader)
图片：file-loader url-loader
html：html-loader
js：babel-loader(@babel/preset-env)

### plugins：

自动清除输出目录：clean-webpack-plugin
自动生成 html(自动导入资源)：html-webpack-plugin
复制资源：copy-webpack-plugin
提取 css 文件(使用 link 标签引入)：mini-css-extract-plugin
压缩 css：optimize-css-assets-webpack-plugin
压缩 js：terser-webpack-plugin

### 其他：

webpack-dev-server：webpack 开发服务器(开启服务器，自动打包，自动刷新浏览器)

HMR(热替换)：在不刷新浏览器的基础上替换模块进行更新(热更新逻辑需要自己写，css 不需要，已经在 css-loader 处理)

webpack-merge：合并 webpack 配置

DefinePlugin：注入全局变量，默认注入 process.env.NODE_ENV

Tree-Shaking：删除未使用代码

Code Spliting(代码分割)：1.多入口打包(配置 entry 不同入口文件) 2.按需加载

### 高级配置：

抽离 css：mini-css-extract-plugin
抽离公共代码(第三方代码)：

```javascript
{
  optimization: {
    // 代码分割
    splitChunks: {
      chunks: 'all', // initial：对于异步导入的文件不处理，async：只对一部倒入的文件处理，all：全部处理
    },
    // 缓存分组
    cacheGroups: {
      // 第三方模块
      vendor: {
        name: 'vendor', // chunk 名称
        priority: 1, // 权限更高，优先抽离
        test: /node_modules/, // 模块路径
        minSize: 0, // 大小限制
        minChunks: 1 // 最少复用次数
      },
      // 公共模块
      common: {
        name: 'common', // chunk 名称
        priority: 0, // 权限更高，优先抽离
        minSize: 0, // 大小限制
        minChunks: 2 // 最少复用次数
      }
    }
  }
}
```

懒加载：import().then(() => {})

### module chunk bundle 区别

module：引入的各个源码文件(源码)
chunk：多个模块的合成(根据 module 合成)(entry，import()，splitChunk)
bundle：最终输出文件(根据 chunk 输出)

### webpack 性能优化

#### 优化打包构建速度：

优化 babel-loader:

```javascript
{
  test: /\.js$/,
  use: ['babel-loader?cacheDirectory], // 开启缓存
  include: path.resolve(__dirname, 'src') // 明确范围
}
```

IgnorePlugin 避免引入无用模块：

```javascript
{
  plugins: [new webpack.IgnorePlugin(/\.\/locale/, /moment/)] // 忽略 moment 文件下的 locale 文件
}
```

noParse：引入但不打包

happyPack 多进程打包：

```javascript
{
  module: {
    rules: [
      {
        test: /\.js$/,
        // 将 .js 文件使用 id为 babel的loader进行处理
        use: ['happypack/loader?id=babel']
      }
    ]
  },
  plugins: [
    new HappyPack({
      // 用唯一标识符 id 来代表当前的 HappyPack 是用来处理一类特定的文件
      id: 'babel',
      // 如何处理 .js 文件，用法和 loader配置一样
      loaders: ['babel-loader?cacheDirectory']
    })
  ]
}
```

ParallelUglifyPlugin 开启多进程压缩 js

DllPlugin 动态链接库插件：
DllPlugin 产出 dll 文件
DllReferenceplugin 使用 dll 文件

#### 优化产出代码：

小图片使用 base64 格式
bundle 加 hash(contentHash)
懒加载
提取公共代码(第三方，公共代码)
IgnorePlugin(禁用没用的代码)
CDN 加速：

```javascript
{
  output: {
    publicPath: 'cdn' // 生成所有静态文件的 url 前缀，配合打包后资源上传到 cdn
  }
}
```

production 模式：

自动开启压缩

自动删除调试代码

自动启用 tree-shaking(必须使用 esmodule，commonjs 无效)

esmodule 和 commonjs 区别：
esmodule 是静态引入，编译时引入，commonjs 是动态引入，执行代码时引入，只有 esmodule 才能静态分析，实现 tree-shaking

Scope Hosting 作用域提升，合并代码，缩小代码体积
ModuleConcatenationPlugin 插件

#### babel 相关

@babel/preset-env 转换 js 语法

babel-polyfill 注入 新的 es api 补丁，core-js 和 regenerator 的集合
core-js 所有 es5+ api 的兼容补丁
regenerator generator api 的 兼容补丁

babel-polyfill 按需引入：

```javascript
{
  presets: [
    [
      @babel/preset-env,
      {
        useBuiltIns: usage,
        corejs: 3
      }
    ]
  ]
}
```

babel-runtime 不会污染全局环境
