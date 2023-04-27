---
title: gzip压缩
date: 2021-12-10 20:10:03
categories: [WEB性能优化]
tags: [性能优化, nginx, gzip]
---

# gzip简介

在讨论性能优化的时候，你可能会经常听到一个叫gzip压缩的名词，这篇文章就对gzip做一个详细的介绍。

gzip就是GNUzip的缩写，也是一个文件压缩程序，可以将文件压缩进后缀为.gz的压缩包。而我们前端所讲的gzip压缩优化，就是通过gzip这个压缩程序，对资源进行压缩，从而降低请求资源的文件大小。

gzip压缩优化应用非常广泛，基本上你打开任何一个网站，看它们的html，js，css文件都是经过gzip压缩的（即使js，css这类文件经过了混淆压缩之后，gzip仍然可以明显的优化文件体积。）

> Tips：通常gzip对纯文本内容可压缩到原大小的40%。但png、gif、jpg、jpeg这类图片文件并不推荐使用gzip压缩（svg是个例外），首先经过压缩后的图片文件gzip能压缩的空间很小，对于图片这种，可以采用webp的形式，专属于图片的压缩。

比如在调试工具network中，Response Headers中找到 **content-encoding: gzip** 键值对，这就表示了这个文件是启用了gzip压缩的。

<img src="gzip_demo.png">

<br>

# 开启gzip压缩

gzip压缩一般分为两种：

1. 一种是前端直接打包，在构建的过程中生成gzip文件，nginx直接拿压缩后的文件返给浏览器，浏览器拿到压缩文件后，会自动解压，这种也可以叫nginx静态压缩
   - 优点：服务端运行时可快速响应压缩文件，节省nginx性能
   - 缺点：会增加构建时间和资源上传时间（源文件 + 压缩后的文件）
2. 另一种是nginx动态压缩，nginx拿到资源后，直接压缩，再返回给浏览器
   - 优点：不需要前端参与，直接nginx处理
   - 缺点：比较消耗nginx的CPU



***前端开启gzip***

主要利用打包工具的插件来实现，webpack使用[CompressionWebpackPlugin](https://webpack.js.org/plugins/compression-webpack-plugin/)插件做压缩，vite使用 [vite-plugin-compression](https://github.com/anncwb/vite-plugin-compression) 插件做压缩

Webpack具体实现如下

```javascript
new CompressionPlugin({
  filename: '[path][base].gz',  
  algorithm: 'gzip',  
  test: /\.js$|\.css$|\.html$/,  
  threshold: 10240,  
  minRatio: 0.8, 
})
```

vite具体实现如下

```javascript
import compressPlugin from 'vite-plugin-compression';
import type { Plugin } from 'vite';
function configCompressPlugin(
  compress: 'gzip' | 'brotli' | 'none',
  deleteOriginFile = false
): Plugin | Plugin[] {
  const compressList = compress.split(',');

  const plugins: Plugin[] = [];
	// gzip压缩
  if (compressList.includes('gzip')) {
    plugins.push(
      compressPlugin({
        ext: '.gz',
        threshold: 1024,
        deleteOriginFile,
      })
    );
  }
	// br压缩
  if (compressList.includes('brotli')) {
    plugins.push(
      compressPlugin({
        ext: '.br',
        algorithm: 'brotliCompress',
        threshold: 1024,
        deleteOriginFile,
      })
    );
  }
  return plugins;
}
```

另外还需要在nginx开启静态压缩

```nginx
server {
  gzip on;
  gzip_static on;
}
```

配置完后，可以看到请求response headers里的Content-Encoding:gzip字段

<br>

***服务端开启gzip***

直接在nginx中配置

```nginx
server {
  gzip on; # 开启gzip
  gzip_min_length 1k; # 设置允许压缩的页面最小字节数
  gzip_buffers 4 16k; # 设置用于处理请求压缩的缓冲区数量和大小
  gzip_http_version 1.0; # 压缩版本
  gzip_comp_level 2; # 设置压缩比率，0-9，比率越低，处理速度快，传输速度慢
  gzip_types text/plain application/javascript text/css application/xml; # 设置压缩类型
  gzip_vary on; # 开启后，如果response headers里有Accept-Encoding:gzip，表示浏览器支持gzip压缩
}
```

tips：配置完成后记得重启nginx服务！！！

<br>

# 补充内容：brotli压缩

br压缩也是目前非常流行的压缩方式，在常用网站中也会经常看到，对应的Response Headers中 **content-encoding: br** ，

比如打开百度网页，查看network，如下：

<img src="br_demo.png">

<br>

> 需要注意的是，brotli 压缩只能在 https 中生效，因为 在 http 请求中 request header 里的 Accept-Encoding: gzip, deflate 是没有 br 的。
