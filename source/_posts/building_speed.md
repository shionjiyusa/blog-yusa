---
title: "旧版本CRA打包构建速度优化"
url: building_speed.html
id: building_speed
categories:
  - 碎碎念
date: 2021-07-02 20:11:00
tags:
---

![gura](/img/post/gura.jpg)

公司项目由于使用的 react-scripts 版本太老，试了一下没办法一步升级到最新版本，每次打包都得花 15 分钟左右，实在是急死人。

最近实在忍不住了，尝试对打包速度进行了优化。走了一些弯路，但是最后打包时间缩短到了 1 分钟，还是非常欣慰的。

## 优化思路

首先弄清楚优化思路很重要，webpack 可以使用 SMP(speed-measure-webpack-plugin) 对打包时间进行记录，打包完成后会显示每个 loader 和插件运行耗费的时间，然后针对耗时高的 loader 和插件进行优化。

## 具体方法

分析了很久才发现打包耗时是因为 uglify 实在是太慢了，各大代码平台也提到了这个问题，本来想通过升级整体架构来解决，但是遇到的冲突太多了，不得不另想办法。

### 重点针对 uglify 进行优化

uglify 的优化思路其实很简单，多线程 + 缓存。

1. 使用 `webpack-parallel-uglify-plugin` 代替原有的 uglify 插件

安装插件前还要注意版本，太新的版本可能不兼容

>注意如果使用 config-override 进行配置，需要对原有的 uglify 插件进行覆盖

1. 使用 ParallelUglifyPlugin 的配置时要注意可能对代码造成的侵入性改动，建议关闭不必要的优化选项

### 针对 babel-loader 进行优化

babel 其实也是比较耗时的，尝试用 `esbuild-loader` 替换 `babel-loader` 之后发现，`esbuild-loader` 并不支持直接转换 ES6 代码，而项目本身的代码质量过低，没办法直接使用 `esbuild-loader` 进行构建，所以使用 `thread-loader` 开启 babel-loader 多线程。

## 配置参考

```js
// /config-overrides.js

const { injectBabelPlugin, getBabelLoader } = require('react-app-rewired');
const rewireLess = require('react-app-rewire-less');
const ParallelUglifyPlugin = require('webpack-parallel-uglify-plugin');

const path = require('path');
const { override, addWebpackAlias, disableEsLint } = require('customize-cra');

const conf = () => (config, env) => {
  // ...其他配置
  // 开启 babel 多线程
  const babelLoader = getBabelLoader(config.module.rules);
  const { options } = babelLoader;
  options.cacheDirectory = true;
  babelLoader.use = [
    { loader: 'thread-loader', options: { workers: 15, workerParallelJobs: 50, poolTimeout: 2000 } },
    {
      loader: babelLoader.loader,
      options,
    },
  ];
  delete babelLoader.loader;
  delete babelLoader.options;

  if (env === 'production') {
    // 开启 uglify 多线程
    config.plugins.splice(3, 1);
    config.plugins.push(
      new ParallelUglifyPlugin({
        parallel: true,
        // 本质上还是用了 UglifyJS，以下是传给 UglifyJS 的参数，优化的点在于开启了多进程
        uglifyJS: {
          output: {
            beautify: false, // 使用紧凑的输出
            comments: false, // 删除所有的注释
          },
          compress: {
            drop_console: true, // 删除所有 console
            collapse_vars: false, // 内嵌定义了但是只用到一次的变量
            reduce_vars: false, // 提取出出现多次，但是没有定义成变量去引用的静态值
            unused: false, // 提取未使用的局部变量
            drop_debugger: true, // 删除所有 debugger
            keep_fnames: true, // 保持函数名
          },
        },
        cacheDir: 'cache', // 开启缓存
      })
    );
  }
  return config;
};

module.exports = override(
  // ...其他配置
  conf(), // 将自定义配置组合进来
);
```

## 总结

提到优化，首先还是得分析问题所在，然后尝试不同的方向去解决问题，不能想当然套用别人的优化方式。
