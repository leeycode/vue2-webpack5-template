## 实现思路

从不同阶段的角度来划分，不同的阶段所需要的功能不同，因此整体构建可划分为：本地开发阶段、生产部署阶段

> 开发阶段: 需要本地启动服务器环境，支持代码即时预览、模块热更新、devtool调试功能等；

> 生产部署阶段: 需要开启压缩优化、静态资源复制目录、代码tree shaking、chunk抽离等功能；阶段不同所需功能不同。

不同阶段所需功能不同，但工具的基础能力一致，因此可将整体构建拆分为如下结构：

- 基础通用功能

  - entry 入口配置
  - output 输出配置
  - resolve 自动文件识别与简化路径配置
  - externals 排除指定依赖而减小打包体积的外部拓展配置
  - module-loader 以loader转换非js模块配置
  - plugins 全局变量、页面模板、语法工具等插件配置
  - optimization 模块抽取等性能相关配置
  - 其他相关配置

- 开发阶段配置

  - dev server 本地服务器配置
  - devtool 开发调试配置
  - proxy 接口代理转发配置
  - hot HMR模块热更新配置
  - compress gzip等压缩优化配置
  - router historyApiFallback本地路由重定向配置
  - optimization 模块名称chunkIds配置

- 生产部署阶段
   
  - 清理dist目录
  - 复制静态资源
  - css单独拆包
  - css/js丑化压缩
  - 作用域提升
  - tree shaking
  - 代码拆分
  
  > 想看完整配置文件小伙伴可直接移步文章末尾

## 构建具体实现

### 目录

预定使用以下目录结构

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8baef760feaa4f829381340d9b06fc74~tplv-k3u1fbpfcp-watermark.image)

- config
    
  webpack构建相关配置目录
  
  - paths.js 获取指定目录绝对路径
  - webpack.common.js 基础通用配置
  - webpack.config.js 主构建配置
  - webpack.dev.js 开发环境配置
  - webpack.prod.js 生产环境配置
 
- public
    
    公共通用的静态资源目录
    
- src
    
    - assets 项目内模块使用的静态资源目录
    - components 项目的公共组件目录
    - App.vue 项目主页面
    - main.js 项目入口
    - style.less 项目通用样式-less
    
- .eslintrc.js
   
   代码风格配置文件

- babel.config.js

  工具babel转换配置，设置预设等
  
- postcss.config.js

  css样式转换配置
  
- package.json
   
  依赖包等信息配置文件

### 基础通用功能
    
项目打包需要依赖webpack且需要在命令行执行，因此先行安装webpck、webpack-cli到开发依赖

```js
npm i webpack webpack-cli --save-dev
```
  
安装完毕后在webpack.common.js中增加通用配置

#### entry 入口配置

```js
module.exports = {
    entry: './src/main.js' // 默认是此目录，如若文件名称或者入口变更可修改
}
```

#### output 输出配置

```js
const resolveApp = require('./paths')

module.exports = {
    output: {
        path: resolveApp('dist'),
        filename: 'js/[name].[hash:6].js',
        chunkFilename: 'js/[name].chunk.[hash:4].js'
    }
}
```

paths.js文件内容

```js
const path = require('path') // webpack内置模块

const appRoot = process.cwd() // 命令行运行的根目录

const resolveApp = (resolvePath) => {
  return path.resolve(appRoot, resolvePath) // 获取指定目录的完整绝对路径
}

module.exports = resolveApp
```

path：文件输出到dist目录中，根目录路径的获取通过命令行终端函数获取

filename：构建后的脚本文件输出到js目录下，且使用动态文件名称，附带文件6为hash值

chunkFilename：构建时拆分出来的chunk文件同样放入js目录下，且名称除了以上规则外，增加chunk标识

#### resolve 路径简化配置

在模块源码中，通过import加载其他模块时需要写出其前缀目录地址，如若嵌套较深则会书写麻烦，此时可以配置特定路径标识映射，以简化此路径。

同时，导入其他模块时默认在文件末尾增加js文件格式结尾，其他文件格式会无法找到，配置自动补全规则来增加其他文件格式补全

```js
module.exports = {
    resolve: {
      extensions: ['.js', '.jsx', '.vue', '.json', '...'],
      alias: {
        '@': resolveApp('src')
      }
    }
}
```

resolve：通过`extensions`配置自动补全的文件类型

resolve：通过`alias`配置简化路径的映射标识

#### externals 外部拓展

外部依赖的库文件如若较大且不会频繁变动，则可将库文件的导入交给外部，如使用外链形式走cdn加速、构建dll库或其他形式，通过html页面增加script脚本引入库文件

```js
module.exports = {
    externals: { // 不希望依赖打进包中，走外链cdn等
        // '$': 'Jquery',
        // react: 'React',
        // 'react-dom': 'ReactDOM',
        // 'prop-types': 'PropTypes',
        // moment: 'moment',
        // antd: 'antd',
        // classnames: 'classNames'
    }
}
```

#### module loader配置

##### 为什么要用loader

1.webpack默认只能处理js文件，而我们项目中不可避免要用到样式表css、图片imgage、字体font等资源，而这类的资源文件webpack并不认识，因此无法直接处理

2.js模块中我们会用到最新的ES6+语法，也可能会用到coffee script语法，或者严谨的typescript语法等，这些语法在浏览器环境中支持程度不一，或不支持直接解析。

基于以上种种原因，我们需要通过loader加载器来将这类资源文件转换为可以被识别并解析的标准js

##### loaders

- vue-loader

安装

先安装vue到项目依赖，再安装vue-loader、vue-template-compiler到开发依赖

vue-template-compiler 的作用是将vue的模板代码预编译为渲染函数，以避免在运行时编译开销

```js
npm i vue --save
npm i vue-loader vue-template-compiler --save-dev
```

使用

```js
const VueLoaderPlugin = require('vue-loader/lib/plugin')

module.exports = {
    module: {
      rules: [
        {
          test: /\.vue$/,
          use: 'vue-loader'
        }
      ]
    },
    plugins: [
        new VueLoaderPlugin()
    ]
}
```

安装完毕后rules内添加相应规则，检测vue结尾的文件，并使用vue-loader。同时要求单独导出vueLoaderPlugin，在plugins插件集合内实例化，以此来正确解析vue文件

- css-loader

对于样式表css的处理和其他vue、js处理不同，样式表css处理需要经过多步骤处理后才能正常使用，具体流程为：

1.使用postcss-loader将css文件内样式根据浏览器支持情况转换为多平台兼容写法，如使用autoprefixer、browserslist、caniuse-lite等依赖增加平台特性写法；

2.接着将转换后的样式表css交由css-loader处理import、url等行为，将引入和依赖的外部资源加载到当前样式表中；

3.处理完毕后将css交由style-loader，已style标签的形式注入html页面中

这里存在一个问题，开发环境时将css全部注入页面没有影响，但生产环境会是多文件大量内容，超出量后会导致首页加载变慢。因此需要在生产阶段将css单独抽离出文件，通过判断环境来加载相应配置,将module.exports导出内容变更为函数，接受环境标识

安装loader

```js
npm i postcss-loader css-loader style-loader --save-dev
```

安装插件plugin

```js
npm i mini-css-extract-plugin --save-dev
```

使用

```js
const MiniCssExtractPlugin = require('mini-css-extract-plugin')

module.exports = (isProduction) => {
    const cssFinalLoader = isProduction ? MiniCssExtractPlugin.loader : 'style-loader'
    return {
        module: {
          rules: [
            {
              test: /\.css$/,
              use: [
                cssFinalLoader, // 开发与生产使用不同loader
                {
                  loader: 'css-loader',
                  options: {
                    esModule: false, // css不使用esModule，直接输出
                    importLoaders: 1 // 使用本loader前使用1个其他处理器
                  }
                },
                'postcss-loader'
              ],
              sideEffects: true // 希望保留副作用
            }
          ]
        }
    }
}
```

  前边提到了postcss-loader会根据支持度情况自动转换css内容，postcss-loader作为转换的一个平台，会提供插槽的能力，但是转换的具体规则并不会去做，因此这个转换规则需要我们通过其他插件解决。这里我们使用一个预设的转换集合`postcss-preset-env`
  
安装

```js
npm i postcss-preset-env --save-dev
```

使用

postcss.config.js文件

```js
module.exports = {
  plugins: [
    require('postcss-preset-env')
  ]
}
```

- less-loader

less-loader与css-loader处理逻辑相似，区别为：处理前需要先使用less-loader将less文件处理为css文件，处理less文件依赖less包。

安装

```js
npm i less less-loader --save-dev
```

配置

```js
const MiniCssExtractPlugin = require('mini-css-extract-plugin')

module.exports = (isProduction) => {
    const cssFinalLoader = isProduction ? MiniCssExtractPlugin.loader : 'style-loader'
    return {
        module: {
          rules: [
            {
              test: /\.less$/,
              use: [
                cssFinalLoader,
                {
                  loader: 'css-loader',
                  options: {
                    importLoaders: 2
                  }
                },
                'postcss-loader',
                'less-loader'
              ]
            }
          ]
        }
    }
}
```

- 图片-loader

在webpack4版本中，图片类资源是通过file-loader和url-loader来配合实现。但webpack5版本中官方内置了asset来处理静态资源类。

配置

```js
module.exports = (isProduction) => {
    return {
        module: {
          rules: [
            {
              test: /\.(png|gif|jpe?g|svg)$/,
              type: 'asset', // webpack5使用内置静态资源模块，且不指定具体，根据以下规则使用
              generator: {
                filename: 'img/[name].[hash:6][ext]' // ext本身会附带点，放入img目录下
              },
              parser: {
                dataUrlCondition: {
                  maxSize: 10 * 1024 // 超过10kb的进行复制，不超过则直接使用base64
                }
              }
            }
          ]
        }
    }
}
```

- font 字体类loader

配置

与图片类资源处理相似，但去除了大小限制，因字体文件是整体，且无需额外处理，因此直接进行静态资源复制

```js
module.exports = (isProduction) => {
    return {
        module: {
          rules: [
            {
              test: /\.(ttf|woff2?|eot)$/,
              type: 'asset/resource', // 指定静态资源类复制
              generator: {
                filename: 'font/[name].[hash:6][ext]' // 放入font目录下
              }
            }
          ]
        }
    }
}      
```

- babel-loader

js模块的转换操作需要依赖babel-loader，与postcss-loader类似，babel-loader也可以理解为一个转换的平台，并不做具体转换规则，因此需要依赖`@babel/core`、`@babel/preset-env`、`@vue/cli-plugin-babel`插件来完成转换

安装

```js
npm i @babel/core @babel/preset-env @vue/cli-plugin-babel --save-dev
```

配置

```js
module.exports = (isProduction) => {
    return {
        module: {
          rules: [
            {
              test: /\.jsx?$/,
              exclude: /node_modules/, // 过滤掉node_modules目录，只使用而已
              use: 'babel-loader' // js、jsx使用bable-loader处理
            }
          ]
        }
    }
}
```

进行语法转换的时候需要排除掉node_modules目录


babel.config.js文件

```js
module.exports = {
  presets: [
    '@vue/cli-plugin-babel/preset',
    ['@babel/preset-env', {
      useBuiltIns: 'usage', // 用到什么填充什么，按需
      corejs: 3 // 默认是2版本
    }]
  ]
}
```

- eslint-loader

代码风格检查

安装

```js
npm i eslint eslint-loader --save-dev
```

配置

```js
npx eslint --init // 根据提示完成配置，详情见webpack打包-上
```

执行完毕后会安装相关依赖和生成`.eslintrc.js`配置文件

.eslintrc.js
```js
module.exports = {
  env: {
    browser: true,
    es2021: true
  },
  extends: [
    'plugin:vue/essential',
    'standard'
  ],
  parserOptions: {
    ecmaVersion: 12,
    sourceType: 'module'
  },
  plugins: [
    'vue'
  ],
  rules: {
  }
}

```

#### plugins 配置

基础功能内提供全局变量定义、页面模板、vue相关plugin等配置

```js
module.exports = (isProduction) => {
    return {
        plugins: [
          new DefinePlugin({
            BASE_URL: '"./"'
          }),
          new HtmlWebpackPlugin({
            title: 'vue2-webpack5-anted',
            template: 'public/index.html'
          }),
          new VueLoaderPlugin()
        ]
    }
}
```

#### 完整示例

```js
const { DefinePlugin } = require('webpack')
const HtmlWebpackPlugin = require('html-webpack-plugin')
const TerserWebpackPlugin = require('terser-webpack-plugin')
const MiniCssExtractPlugin = require('mini-css-extract-plugin')
const VueLoaderPlugin = require('vue-loader/lib/plugin')
const resolveApp = require('./paths')

module.exports = (isProduction) => {
  const cssFinalLoader = isProduction ? MiniCssExtractPlugin.loader : 'style-loader'

  return {
    entry: './src/main.js',
    output: {
      path: resolveApp('dist'),
      filename: 'js/[name].[hash:6].js',
      chunkFilename: 'js/[name].chunk.[hash:4].js'
    },
    resolve: {
      extensions: ['.js', '.jsx', '.vue', '.json', '...'],
      alias: {
        '@': resolveApp('src')
      }
    },
    externals: { // 不希望依赖打进包中，走外链cdn等
      // '$': 'Jquery',
      // react: 'React',
      // 'react-dom': 'ReactDOM',
      // 'prop-types': 'PropTypes',
      // moment: 'moment',
      // antd: 'antd',
      // classnames: 'classNames',
    },
    module: {
      rules: [
        // vue文件处理
        {
          test: /\.vue$/,
          use: 'vue-loader'
        },
        // less文件处理
        {
          test: /\.less$/,
          use: [
            cssFinalLoader,
            {
              loader: 'css-loader',
              options: {
                importLoaders: 2
              }
            },
            'postcss-loader',
            'less-loader'
          ]
        },
        // css文件处理
        {
          test: /\.css$/,
          use: [
            cssFinalLoader, // 最终要以style标签输出到页面
            {
              loader: 'css-loader',
              options: {
                esModule: false, // css不使用esModule，直接输出
                importLoaders: 1 // 使用本loader前使用1个其他处理器
              }
            },
            'postcss-loader'
          ],
          sideEffects: true // 希望保留副作用
        },
        // 图片类处理
        {
          test: /\.(png|gif|jpe?g|svg)$/,
          type: 'asset', // webpack5使用内置静态资源模块，且不指定具体，根据以下规则 使用
          generator: {
            filename: 'img/[name].[hash:6][ext]' // ext本身会附带 点，放入img目录下
          },
          parser: {
            dataUrlCondition: {
              maxSize: 10 * 1024 // 超过10kb的进行复制，不超过则直接使用base64
            }
          }
        },
        // 字体类处理
        {
          test: /\.(ttf|woff2?|eot)$/,
          type: 'asset/resource', // 指定静态资源类复制
          generator: {
            filename: 'font/[name].[hash:6][ext]' // 放入font目录下
          }
        },
        // 脚本类处理
        {
          test: /\.jsx?$/,
          exclude: /node_modules/, // 过滤掉node_modules目录，只使用而已
          use: 'babel-loader' // js、jsx使用bable-loader处理
        },
        {
          test: /\.(vue|js)$/,
          use: 'eslint-loader',
          exclude: /node_modules/,
          enforce: 'pre'
        }
      ]
    },
    plugins: [
      new DefinePlugin({
        BASE_URL: '"./"'
      }),
      new HtmlWebpackPlugin({
        title: '贾贵山-自建构建打包流程',
        template: 'public/index.html'
      }),
      new VueLoaderPlugin()
    ],
    optimization: {
      runtimeChunk: true, // 模块抽取，利用浏览器缓存
      minimizer: [
        new TerserWebpackPlugin({
          extractComments: false // 不要注释生成的文件
        })
      ]
    }
  }
}

```

### 开发阶段配置

因开发环境在通用配置基础上配置，因此直接展示完整示例，对相应配置进行说明

安装

```js
npm i webpack-merge webpack-dev-server --save-dev
```

配置

```js
const baseConfig = require('./webpack.common')
const { merge } = require('webpack-merge')

module.exports = merge(baseConfig(false), {
  mode: 'development',
  devtool: 'cheap-module-source-map',
  target: 'web',
  devServer: {
    hot: 'only',
    port: 3002, // 端口号，工作中从3001开始，因此增加1个到3002
    open: true, // 自动打开浏览器
    compress: true, // 开启gzip压缩
    historyApiFallback: true, // history路径在刷新出错时重定向开启
    proxy: { // 接口代理
      '/api': { // 统一api前缀都代理掉
        target: 'http://api.github.com', // 代理的目标地址
        changeOrigin: true, // 改变来源信息
        pathRewrite: { // 因前缀为自己增加，因此重写地址
          '/api': '' // 将前缀去掉
        }
      }
    }
  },
  optimization: {
    chunkIds: 'named'
  }
})
```

- 使用 `webpack-merge` 包进行配置合并
- mode: 指定当前模式为开发模式 `development`
- devtool: 指定生成source map，且规则模式为 `cheap-module-source-map`
- target: 指定目标为web平台
- devServer: 本地服务环境配置对象
    - hot: 开启热更新
    - port: 本地服务的端口号为3002，可任意更改
    - open: 服务启动后自动打开浏览器，默认为true
    - compress: 压缩模式
    - historyApiFallback: history模式访问时如路由不存在则重定向到首页
    - proxy: 请求代理
        - '/api': 代理的请求点缀
        - target: 将要代理到的目标地址
        - changeOrigin: 是否更改来源信息
        - pathRewrite: 重写配置
 
### 生产阶段配置

```js
const baseConfig = require('./webpack.common')
const { merge } = require('webpack-merge')
const CopyWebpackPlugin = require('copy-webpack-plugin')
const { CleanWebpackPlugin } = require('clean-webpack-plugin')
const MiniCssExtractPlugin = require('mini-css-extract-plugin')
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin')
const webpack = require('webpack')
const PurgeCSSPlugin = require('purgecss-webpack-plugin')
const resolveApp = require('./paths')
const glob = require('glob')
const CompressionPlugin = require('compression-webpack-plugin')
// var InlineChunkHtmlPlugin = require('inline-chunk-html-plugin');
// const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = merge(baseConfig(true), {
  mode: 'production',
  plugins: [
    new CleanWebpackPlugin(),
    new CopyWebpackPlugin({
      patterns: [{
        from: 'public',
        to: 'public',
        globOptions: {
          ignore: ['**/index.html']
        }
      }]
    }),
    new MiniCssExtractPlugin({ // css单独拆分为文件
      filename: 'css/[name].[hash:6].css'
    }),
    new webpack.optimize.ModuleConcatenationPlugin(), // 作用域提升，提升性能
    new PurgeCSSPlugin({
      paths: glob.sync(`${resolveApp('./src')}/**/*`, { nodir: true })
    }),
    new CompressionPlugin({
      test: /\.(css|js)$/i,
      algorithm: 'gzip'
    })
    // new InlineChunkHtmlPlugin(HtmlWebpackPlugin, [/runtime/]) // 将一定大小文件直接注入 html,
  ],
  optimization: {
    chunkIds: 'deterministic', // 文件名称尽可能短，也会是序号类型
    splitChunks: {
      chunks: 'all',
      minSize: 20000, // 拆分的每个包不小于20kb
      maxSize: 20000, // 体积大于设置的值的包要去拆分开包
      minChunks: 1, // 包如果要拆分，则必须要至少引用一次
      cacheGroups: {
        syVendors: {
          test: /[\\/]node_modules[\\/]/, // 对目录内文件进行单独打包拆分，且放入一个文件中 vender
          filename: 'js/[id]_verdor.js', // 最终名字
          priority: -10 // 都满足时候的优先级，越高月用
        }
      }
    },
    minimizer: [
      new CssMinimizerPlugin()
    ]
  }
})
```

生产环境依然采用合并基础配置方式

- mode: 此时采用 `production`模式
- plugins: 生产阶段使用的plugins列表
    - CleanWebpackPlugin 清除dist目录插件
    - CopyWebpackPlugin 静态资源目录复制插件
    - MiniCssExtractPlugin 单独拆分出css包
    - ModuleConcatenationPlugin 将js的作用域提升，减少作用域层级来提升性能
    - PurgeCSSPlugin 摇掉未使用的css
    - CompressionPlugin 采用gzip形式押错css和js
    - optimization 生产环境优化配置

### 构建入口配置

```js

const devConfig = require('./webpack.dev')
const prodConfig = require('./webpack.prod')
// const SpeedMeasurePlugin = require("speed-measure-webpack-plugin");

// const smp = new SpeedMeasurePlugin();

module.exports = (env) => {
  // const finalResult = env.production ? prodConfig : devConfig;
  // console.log('final---->', finalResult)
  // return smp.wrap(finalResult);
  return env.production ? prodConfig : devConfig
}

```

构建时减少繁琐修改配置文件的操作，在package.json内添加启动参数来选择不同环境配置


### ant-design-vue
```js
npm i --save ant-design-vue@1.7.2
```

官方有对其配置按需加载功能，但是在实际使用用过程中，很多没有使用的组件还是被打包进了项目，主要集中在@ant-design/icons、moment上，占用了很大的体积。

babel.config.js中：
```js
plugins: [
    <!--ant-design-vue按需加载配置-->
    [
      'import',
      { libraryName: 'ant-design-vue', libraryDirectory: 'es', style: 'css' },
      'ant-design-vue'
    ],
    <!--lodash按需加载配置-->
    'lodash'
  ]
```


### 项目构建

```js
npm install
npm run serve
npm run build
```

本地开发则启动 `npm run serve` 
构建生产环境则 `npm run build`
检查代码风格则 `npm run lint`
