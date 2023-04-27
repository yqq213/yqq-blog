---
title: 微前端qiankun在项目中的使用
date: 2022-09-05 21:09:55
categories: [微前端]
tags: [微前端, qiankun]
---

# 什么是微前端

> 微前端是一种多个团队通过独立发布功能的方式来共同构建现代化 web 应用的技术手段及方法策略

微前端架构具有以下几个核心价值：

- 技术栈无关

  主框架不限制接入应用的技术栈，微应用具备完全自主权

- 独立开发、独立部署

  微应用仓库独立，前后端可独立开发，部署完成后主框架自动完成同步更新

- 增量升级

  在面对各种复杂场景时，我们通常很难对一个已经存在的系统做全量的技术栈升级或重构，而微前端是一种非常好的实施渐进式重构的手段和策略

- 独立运行时

  每个微应用之间状态隔离，运行时状态不共享

> 微前端架构旨在解决单体应用在一个相对长的时间跨度下，由于参与的人员、团队的增多、变迁，从一个普通应用演变成一个巨石应用后，随之而来的应用不可维护的问题。这类问题在企业级 Web 应用中尤其常见。

*或许你会想到为什么不用iframe来实现微前端呢，多么方便啊*

是的，iframe确实是最接近微前端的实现方案，但是它也会存在一些问题，比如：

1. url 不同步。浏览器刷新 iframe url 状态丢失、后退前进按钮无法使用。
2. UI 不同步，DOM 结构不共享。想象一下屏幕右下角 1/4 的 iframe 里来一个带遮罩层的弹框，同时我们要求这个弹框要浏览器居中显示，还要浏览器 resize 时自动居中..
3. 全局上下文完全隔离，内存变量不共享。iframe 内外系统的通信、数据同步等需求，主应用的 cookie 要透传到根域名都不同的子应用中实现免登效果。
4. 慢。每次子应用进入都是一次浏览器上下文重建、资源重新加载的过程。



# qiankun实践

qiankun的实践具体可参考官方案例：[项目实践](https://qiankun.umijs.org/zh/guide/tutorial)

官网对使用方法阐述的很清晰，可以照葫芦画瓢



# 注意事项

1. **主应用与微应用之间如何传值**

   在挂载微应用时，主应用增加props，传入微应用所需要的信息

   ```javascript
   {
     resName:'备品备件',
     name:'zhoushan-spare-web',
     activeRule: '/zhoushan-spare-web/',
     entry: 'https:www.xxx.com',
     container: '#spare-web-zhoushan',
     props: {
     	name: '传入的值',
     }
   }
   ```

   微应用在mai n.js中通过mount生命周期里的props获取

   ```javascript
   export async function mount(props) {
     console.log('获取主应用传值',props)
     render(props);
   }
   ```

2. **设置主应用启动后默认进入的微应用**

   通过setDefaultMountApp(appLink)，  appLink为必填

   ```javascript
   import { setDefaultMountApp } from "qiankun"
   setDefaultMountApp('/vue2');
   ```

3. **一个应用如何既作为主应用又作为子应用**

   主应用和子应用可以同时存在，并不冲突，可以在main.js中同时导出qiankun生命周期实现子应用配置，也可以在main.js里注册qiankun实现主应用配置

   ```javascript
   import { registerMicroApps, start } from 'qiankun';
   let app = null
   // 子应用配置
   export async function bootstrap() {
     console.log('[vue] vue app bootstraped')
   }
   export async function mount(props) {
     console.log('[vue] props from main framework', props)
     render(props)
   }
   export async function unmount() {
     console.log('[vue] unmount-----')
     app.unmount()
     app = null
   }
   
   // 主应用配置
   registerMicroApps([
     {
       name: 'vue',
       entry: '//localhost:8080',
       container: '#container',
       activeRule: '/app-vue',
     },
     // ...
   ]);
   // 启动 qiankun
   start();
   ```

   

未完待续～