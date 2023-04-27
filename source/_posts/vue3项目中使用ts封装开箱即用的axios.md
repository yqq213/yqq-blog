---
title: vue3项目中使用ts封装开箱即用的axios
date: 2022-09-06 22:08:53
categories: [TypeScript]
tags: [ts, axios, vue]
---

# 前言

axios目前用于处理前端项目里的ajax请求，每个项目里为了方便对axios进行操作，都会对axios进行封装，本文主要结合之前项目实践以vue3和typescript对axios完成统一封装，做到一个开箱即用的axios。封装后具有以下好处：

1. 使用时代码提示功能更丰富；
2. 灵活的拦截器；
3. 支持请求重试功能，以及自定义请求重试次数；
4. 支持扩展自定义请求头配置。



# 安装依赖

首先需要先安装依赖

```
npm i axios -S
```



# 基础封装request类

新建index.ts，首先实现一个最基础的版本：

```typescript
import axios from 'axios'
import type { AxiosInstance, AxiosRequestConfig, AxiosResponse, AxiosError } from "axios"

class Request {
  // axios实例
  instance: AxiosInstance
  // 基础配置（具体值视项目情况而定）
  baseConfig: AxiosRequestConfig = {
    baseURL: window._env_.baseURL + window._env_.URL_PREFIX,
    timeout: 2000,
    headers: {
      "Content-Type": "application/json",
      "X-GW-AccessKey": window._env_.accessKey,
    }
  }
  constructor(config: AxiosRequestConfig) {
    // 创建实例
    this.instance = axios.create(Object.assign(this.baseConfig, config))
  }
  request(config: AxiosRequestConfig) {
    return this.instance.request(config)
  }
}
export default Request
```

> 注意：`constructor`函数中的参数`config`是在api.ts中请求接口时自定义的一些配置，比如单独设置请求头里的`responseType: 'arraybuffer'`，通过合并基础配置与自定义配置实现请求头里的灵活配置

这里将其封装为一个类，而不是一个函数的原因是因为类可以创建多个实例，适用范围更广，封装性更强一些。



# 封装拦截器

拦截器主要包括请求拦截器和响应拦截器，请求拦截器可以在请求时添加token，响应拦截器可以根据错误码自定义处理等

```typescript
import { ElMsgToast } from "@enn/ency-design"
import type { AxiosInstance, AxiosRequestConfig, AxiosResponse, AxiosError } from "axios"
constructor(config: AxiosRequestConfig) {
  // 创建实例
  this.instance = axios.create(Object.assign(this.baseConfig, config))
  // 设置全局的请求次数，请求的间隙（可自定义）
  this.instance.defaults.retry = 1
  this.instance.defaults.retryDelay = 500
  // request 拦截器
  this.instance.interceptors.request.use(
    (config: AxiosRequestConfig) => {
      return config
    },
    (error: AxiosError) => {
      return Promise.reject(error)
    }
  )
  // response 拦截器
  this.instance.interceptors.response.use(
    (response: AxiosResponse) => {
      if (response?.data?.success == false) {
        ElMsgToast.error(response?.data.message)
      }
      return response.data;
    },
    (error: AxiosError) => {
      let config = error.config
      if (!config || !config.retry) return Promise.reject(error)
      // 设置变量追踪__retryCount的值
      config.__retryCount = config.__retryCount || 0
      // 判断是否超出最大重试次数
      if (config.__retryCount >= config.retry) {
        return Promise.reject(error)
      }
      config.__retryCount += 1
      // Create new promise to handle exponential backoff
      let backoff = new Promise(function (resolve) {
        setTimeout(function () {
          resolve()
        }, config.retryDelay || 1)
      })
      // Return the promise in which recalls axios to retry the request
      return backoff.then(function () {
        return service(config)
      })
    }
  )
}
```

在遇到网络问题导致请求失败时，会根据设置的请求重试次数以及请求间隔时间来处理请求重试，优化请求体验

# 封装请求方法

这里我们只对常用的get和post请求做下封装，put和delete请求可同理参考。

```typescript
  import qs from 'qs'	
  // 定义get方法
  public get<T = any>(
    url: string,
    data?: any,
    config?: AxiosRequestConfig
  ): Promise<BizResponse<T>> {  
    if (data && Object.keys(data).length) {
      return this.instance.get(`${url}?${qs.stringify(data, {indices: false})}`, config);
    } else {
      return this.instance.get(url, config);
    }
  }
  // 定义post方法
  public post<T = any>(
    url: string,
    data?: any,
    config?: AxiosRequestConfig
  ): Promise<BizResponse<T>> {
    return this.instance.post(url, data, config);
  }
```

此处get请求对参数data做了下判断，如果是对象形式，则将参数序列化。

`BizResponse`是通用接口返回结构，一般定义方式为：

```typescript
export interface BizResponse<T = Record<string, unknown> | Array<unknown>> {
  code: string;
  message: string;
  data: T & { pageNum?: number; pageSize?: number; total?: number };
  success: boolean;
}
```

一般到此就完成了axios的封装，完整代码附在文章最后

封装完成后，如何使用呢，一般我们在api.ts中对接口会做统一的管理，例如：

```typescript
import Request from '@/axios'
import { BizResponse } from '@/types'
export const exportAccidentPrevent: (data: string) => Promise<BizResponse> = (data: string) => {
  return request.get('/api/exportAccidentPrecautions', data, {responseType: 'arraybuffer'})
}
```

在业务组件中调用该api时如下：

```vue
<script setup lang="ts">
	onMounted(async () => {
    const res = await exportAccidentPrevent('accExport')
  })
</script>
```

完整封装axios代码如下：

```typescript
import axios from "axios"
import qs from 'qs'
import type { AxiosInstance, AxiosRequestConfig, AxiosResponse, AxiosError } from "axios"
import { ElMsgToast } from "@enn/ency-design"

interface BizResponse<T = Record<string, unknown> | Array<unknown>> {
  code: string;
  message: string;
  data: T & { pageNum?: number; pageSize?: number; total?: number };
  success: boolean;
}

class Request {
  // axios 实例
  instance: AxiosInstance
  // 基础配置
  baseConfig: AxiosRequestConfig = {
    baseURL: window._env_.baseURL + window._env_.URL_PREFIX,
    timeout: 10000,
    headers: {
      "Content-Type": "application/json",
      "X-GW-AccessKey": window._env_.accessKey,
    }
  }
  constructor(config: AxiosRequestConfig) {
    // 创建实例
    this.instance = axios.create(Object.assign(this.baseConfig, config))
    // 设置全局的请求次数，请求的间隙（可自定义）
    this.instance.defaults.retry = 1
    this.instance.defaults.retryDelay = 500
    // request 拦截器
    this.instance.interceptors.request.use(
      (config: AxiosRequestConfig) => {
        return config
      },
      (error: AxiosError) => {
        return Promise.reject(error)
      }
    )
    // response 拦截器
    this.instance.interceptors.response.use(
      (response: AxiosResponse) => {
        if (response?.data?.success == false) {
          ElMsgToast.error(response?.data.message)
        }
        return response.data;
      },
      (error: AxiosError) => {
        let config = error.config
        if (!config || !config.retry) return Promise.reject(error)
        // 设置变量追踪__retryCount的值
        config.__retryCount = config.__retryCount || 0
        // 判断是否超出最大重试次数
        if (config.__retryCount >= config.retry) {
          return Promise.reject(error)
        }
        config.__retryCount += 1
        // Create new promise to handle exponential backoff
        let backoff = new Promise(function (resolve) {
          setTimeout(function () {
            resolve()
          }, config.retryDelay || 1)
        })
        // Return the promise in which recalls axios to retry the request
        return backoff.then(function () {
          return service(config)
        })
      }
    )
  }
  // 定义get方法
  public get<T = any>(
    url: string,
    data?: any,
    config?: AxiosRequestConfig
  ): Promise<BizResponse<T>> {  
    if (data && Object.keys(data).length) {
      return this.instance.get(`${url}?${qs.stringify(data, {indices: false})}`, config);
    } else {
      return this.instance.get(url, config);
    }
  }
  // 定义post方法
  public post<T = any>(
    url: string,
    data?: any,
    config?: AxiosRequestConfig
  ): Promise<BizResponse<T>> {
    return this.instance.post(url, data, config);
  }
}

export default new Request({})
```

