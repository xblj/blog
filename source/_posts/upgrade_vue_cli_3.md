---
title: 升级vue-cli3.0
date: 2018-12-21 00:54:25
tags: vue javascript
category:
  - javascript
  - vue
---

# vue-cli 升级到 3.x 问题集

从 2.x 升级到 3.x 前后差不多用了近两周的时间，这过程中遇到很多的问题，一个个问题解决下来还是很有收获的，在此记录下升级过程中遇到的各种坑。

## 自定义块报错

项目中有很多全局使用的组件，为了便于使用将使用文档写在一个自定义块`docs`中，如下图所示：

{% asset_img docs.png %}

在 2.x 中没有任何问题，，但是升级到 3.x 后会报错，无法解析，如下：

{% asset_img docs_err.png %}

从报错信息中可以看出是缺少对应的 loader。那我们就找个 loader 呗，我们可以把`docs`块里面的作为纯文本来加载，所以可以使用`raw-loader`。接下来又有问题了，我们咋向配置中加入新的 loader 呢。

1. 在项目根目录，新建`vue.config.js`文件；
2. 填写内容：

   ```javascript
   module.exports = {
     chainWebpack: config => {
       config.module
         .rule('docs')
         // 注意，这里很重要，我在这卡了好几天
         .resourceQuery(/blockType=docs/)
         .use('raw')
         .loader('raw-loader')
         .end();
     },
   };
   ```

   > vue-cli 3.0 使用的是[webpack-chain](https://github.com/neutrinojs/webpack-chain)，多看看文档，配置起来也不是太难，其实还是有点难。

3. 再次运行`npm run serve`,运行结果如下：

   {% asset_img success.png %}

貌似成功了，但是真的成功了吗？

## 自定义块打包问题

通过上面我们解决了开发时报错的问题，但是我们`npm run build`之后在`dist`目录搜索，可以看到我们的文档全部被打包进代码：

{% asset_img dist.png %}

为了解决这问题，我找了各种 loader，最后都没有能成功解决问题的，如何有谁解决了的，还请告知一声。

本来打算自己写一个 loader 的看了几天的文档，能解决一些问题，但是还是不够完善，我就先不放出来了，后面想到好的方式再看。最后写了一个简单粗暴的:

```javascript
// loader.js
module.exports = function(source) {
  return ``;
};
```

修改`vue.config.js`

```javascript
// vue.config.js
module.exports = {
  chainWebpack: config => {
    config.module
      .rule('docs')
      // 注意，这里很重要，我在这卡了好几天
      .resourceQuery(/blockType=docs/)
      .use('raw')
      .loader('./loader.js')
      .end();
  },
};
```

再次`build`后，打包后的代码不再有`docs`块中的内容。完美解决，其实并不完美，后面会再写一个 loader 抽取所有`docs`块到一个单独的文件中，这个只是一个临时方案。

虽然看似解决了问题，但是实际上还是有问题。在打包后的代码中，我们能看到多了很多的空函数：

{% asset_img func.png %}

这个问题，我还没有解决，但是相比吧整个`docs`块里面的内容都打包进去，还算能接受。
