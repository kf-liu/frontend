<!-- vscode-markdown-toc -->
* 1. [文件结构](#)
* 2. [收口处 · `useRequest.ts`](#useRequest.ts)
	* 2.1. [📖 源码](#-1)
	* 2.2. [🌍 整体解读](#-1)
* 3. [出入参与实例化 · `useRequestImplement.ts`](#useRequestImplement.ts)
	* 3.1. [📖 源码](#-1)
	* 3.2. [✨ 核心实例 · `Fetch`实例`fetchInstance`](#FetchfetchInstance)
	* 3.3. [入参配置](#-1)
	* 3.4. [`fetchInstance`的生命周期管理](#fetchInstance)
	* 3.5. [出参处理](#-1)
* 4. [幕后大佬 `Fetch.ts`](#Fetch.ts)
	* 4.1. [📖 源码](#-1)
	* 4.2. [🌍 整体解读](#-1)
	* 4.3. [构造 · `constructor`](#constructor)
	* 4.4. [插件处理 · `runPluginHandler`](#runPluginHandler)
	* 4.5. [✨ 核心方法 · `run` & `runAsync`](#runrunAsync)
		* 4.5.1. [📍第一部分, 调用插件`onBefore`, 判断是否立即停止🤚](#onBefore)
		* 4.5.2. [📍第二部分, 判断是否立即返回🦶](#-1)
		* 4.5.3. [📍第三部分, 运行`useRequest`中用户入参`options`中传入的`onBefore`方法.](#useRequestoptionsonBefore.)
		* 4.5.4. [📍第四部分,  真正的请求——`try{}catch{} `](#trycatch)
		* 4.5.5. [🌍 关于`runAsync`, 小叨几句](#runAsync)
	* 4.6. [取消 · `cancel`](#cancel)
	* 4.7. [更新 · `refresh` & `refreshAsync`](#refreshrefreshAsync)
	* 4.8. [`mutate`](#mutate)

<!-- vscode-markdown-toc-config
	numbering=true
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->
# `ahooks/useRequest` 源码解读
`useRequest`作为`ahooks`中最重要的成员之一, 在实际开发中的使用频率还是很高的, 功能也很全面, 值得一读.



##  1. <a name=''></a>文件结构
`useRequest`核心代码位于项目`packages/hooks/src/useRequest/src/`, 其下文件目录:
```
src
├── Fetch.ts
├── types.ts
├── useRequest.ts
├── useRequestImplement.ts
├── plugins
│   ├── useAutoRunPlugin.ts
│   ├── useCachePlugin.ts
│   ├── useDebouncePlugin.ts
│   ├── useLoadingDelayPlugin.ts
│   ├── usePollingPlugin.ts
│   ├── useRefreshOnWindowFocusPlugin.ts
│   ├── useRetryPlugin.ts
│   └── useThrottlePlugin.ts
└── utils
    ├── cache.ts
    ├── cachePromise.ts
    ├── cacheSubscribe.ts
    ├── isDocumentVisible.ts
    ├── isOnline.ts
    ├── limit.ts
    ├── subscribeFocus.ts
    └── subscribeReVisible.ts
```

##  2. <a name='useRequest.ts'></a>收口处 · `useRequest.ts`
自顶向下的看, 自然是从`useRequest.ts入`手, 但显然, 这只是一个隐藏了核心逻辑的规范化收口文件 ⬇️
###  2.1. <a name='-1'></a>📖 源码
```
import useAutoRunPlugin from './plugins/useAutoRunPlugin';
import useCachePlugin from './plugins/useCachePlugin';
import useDebouncePlugin from './plugins/useDebouncePlugin';
import useLoadingDelayPlugin from './plugins/useLoadingDelayPlugin';
import usePollingPlugin from './plugins/usePollingPlugin';
import useRefreshOnWindowFocusPlugin from './plugins/useRefreshOnWindowFocusPlugin';
import useRetryPlugin from './plugins/useRetryPlugin';
import useThrottlePlugin from './plugins/useThrottlePlugin';
import type { Options, Plugin, Service } from './types';
import useRequestImplement from './useRequestImplement';

function useRequest<TData, TParams extends any[]>(
  service: Service<TData, TParams>,
  options?: Options<TData, TParams>,
  plugins?: Plugin<TData, TParams>[],
) {
  return useRequestImplement<TData, TParams>(service, options, [
    ...(plugins || []),
    useDebouncePlugin,
    useLoadingDelayPlugin,
    usePollingPlugin,
    useRefreshOnWindowFocusPlugin,
    useThrottlePlugin,
    useAutoRunPlugin,
    useCachePlugin,
    useRetryPlugin,
  ] as Plugin<TData, TParams>[]);
}

export default useRequest;
```

###  2.2. <a name='-1'></a>🌍 整体解读
抛开`import`、`type`, 这三十行代码实际上只有一句话:
```
function useRequest(service, options, plugins) {
  return useRequestImplement(service, options, [
    ...(plugins || []),
    // 默认插件们
  ];
}

export default useRequest;
```
可见`useRequest`方法实际就只是返回了它隔壁`useRequestImplement`, 同时多带了一些默认插件. 插件具体功能暂且按下不表, 还是先看看更重要的`useRequestImplement.ts`. 

##  3. <a name='useRequestImplement.ts'></a>出入参与实例化 · `useRequestImplement.ts`
先上源码.
###  3.1. <a name='-1'></a>📖 源码
```
import useCreation from '../../useCreation';
import useLatest from '../../useLatest';
import useMemoizedFn from '../../useMemoizedFn';
import useMount from '../../useMount';
import useUnmount from '../../useUnmount';
import useUpdate from '../../useUpdate';

import Fetch from './Fetch';
import type { Options, Plugin, Result, Service } from './types';

function useRequestImplement<TData, TParams extends any[]>(
  service: Service<TData, TParams>,
  options: Options<TData, TParams> = {},
  plugins: Plugin<TData, TParams>[] = [],
) {
  const { manual = false, ...rest } = options;

  const fetchOptions = {
    manual,
    ...rest,
  };

  const serviceRef = useLatest(service);

  const update = useUpdate();

  const fetchInstance = useCreation(() => {
    const initState = plugins.map((p) => p?.onInit?.(fetchOptions)).filter(Boolean);

    return new Fetch<TData, TParams>(
      serviceRef,
      fetchOptions,
      update,
      Object.assign({}, ...initState),
    );
  }, []);
  fetchInstance.options = fetchOptions;
  // run all plugins hooks
  fetchInstance.pluginImpls = plugins.map((p) => p(fetchInstance, fetchOptions));

  useMount(() => {
    if (!manual) {
      // useCachePlugin can set fetchInstance.state.params from cache when init
      const params = fetchInstance.state.params || options.defaultParams || [];
      // @ts-ignore
      fetchInstance.run(...params);
    }
  });

  useUnmount(() => {
    fetchInstance.cancel();
  });

  return {
    loading: fetchInstance.state.loading,
    data: fetchInstance.state.data,
    error: fetchInstance.state.error,
    params: fetchInstance.state.params || [],
    cancel: useMemoizedFn(fetchInstance.cancel.bind(fetchInstance)),
    refresh: useMemoizedFn(fetchInstance.refresh.bind(fetchInstance)),
    refreshAsync: useMemoizedFn(fetchInstance.refreshAsync.bind(fetchInstance)),
    run: useMemoizedFn(fetchInstance.run.bind(fetchInstance)),
    runAsync: useMemoizedFn(fetchInstance.runAsync.bind(fetchInstance)),
    mutate: useMemoizedFn(fetchInstance.mutate.bind(fetchInstance)),
  } as Result<TData, TParams>;
}

export default useRequestImplement;
```
###  3.2. <a name='FetchfetchInstance'></a>✨ 核心实例 · `Fetch`实例`fetchInstance`
可见, `useRequestImplement`中的功能为:
使用`useCreation`实例化一个`Fetch`类: `fetchInstance`, 并对出入参配置及`Fetch`实例生命周期相关逻辑进行简单处理. 由此已可以推知核心逻辑应当在`Fetch.ts`中.
实例化代码:
```
  const fetchInstance = useCreation(() => {
    const initState = plugins.map((p) => p?.onInit?.(fetchOptions)).filter(Boolean);

    return new Fetch<TData, TParams>( // new一个实例
      serviceRef,
      fetchOptions,
      update,
      Object.assign({}, ...initState),
    );
  }, []);
```
而其他代码都围绕这一实例展开:
###  3.3. <a name='-1'></a>入参配置
入参, 配置实例. `useRequestImplement`组件共三项入参: `service`, `options`, `plugins`. 

- 将组件入参中的`service`装进`useLatest`以总是调用最新值, 并将返回的`serviceRef`作为入参在实例化`Fetch`时传入
- 实例化后, 配置`fetchInstance`的`options`为组件入参中的`options`, 同时默认设置`manul`为`false`
- 实例化后, 配置`fetchInstance`的`pluginImpls`为组件入参中的`plugins`, 并让所有插件跑起来
###  3.4. <a name='fetchInstance'></a>`fetchInstance`的生命周期管理
实例化后, 生命周期管理
- 在`useMount`中, 如果`manul = false`即非手动调用, 则在此调用`fetchInstance.run(...params)`唤醒`fetachInstance`, 发起请求
- 在`useUnmount`中调用`fetchInstance.cancel()`, 停止可能有未完成的请求
###  3.5. <a name='-1'></a>出参处理
出参. 所有出参都从`FetchInstance`中返回, 具体可以分为返回值和方法.
- 返回值: 如`loading`, `data`, `error`, `params`. 直接从`fetchInstance.state`中获取并返回.
- 六个方法: 有`cancel`, `refresh`, `refreshAsync`, `run`, `runAsync`, `mutate`. 则为`fn: useMemoizedFn(fetchInstance.fn.bind(fetchInstance))`, 使用`class`的写法绑定后通过`useMemoizedFn`返回以确保函数地址不变. 

至此, `useRequestImplement.ts` 的主要内容就读完了, 接下来进入真正的拉取请求环节——`Fetch`.

##  4. <a name='Fetch.ts'></a>幕后大佬 `Fetch.ts`
`useRequest.ts`, `useRequestImplement.ts`, `Fetch.ts`三个文件分别有30、70和170行…… 因此…… 不想看的就往下多滑两下, 后面还会逐段粘贴.
###  4.1. <a name='-1'></a>📖 源码
```
/* eslint-disable @typescript-eslint/no-parameter-properties */
import { isFunction } from '../../utils';
import type { MutableRefObject } from 'react';
import type { FetchState, Options, PluginReturn, Service, Subscribe } from './types';

export default class Fetch<TData, TParams extends any[]> {
  pluginImpls: PluginReturn<TData, TParams>[];

  count: number = 0;

  state: FetchState<TData, TParams> = {
    loading: false,
    params: undefined,
    data: undefined,
    error: undefined,
  };

  constructor(
    public serviceRef: MutableRefObject<Service<TData, TParams>>,
    public options: Options<TData, TParams>,
    public subscribe: Subscribe,
    public initState: Partial<FetchState<TData, TParams>> = {},
  ) {
    this.state = {
      ...this.state,
      loading: !options.manual,
      ...initState,
    };
  }

  setState(s: Partial<FetchState<TData, TParams>> = {}) {
    this.state = {
      ...this.state,
      ...s,
    };
    this.subscribe();
  }

  runPluginHandler(event: keyof PluginReturn<TData, TParams>, ...rest: any[]) {
    // @ts-ignore
    const r = this.pluginImpls.map((i) => i[event]?.(...rest)).filter(Boolean);
    return Object.assign({}, ...r);
  }

  async runAsync(...params: TParams): Promise<TData> {
    this.count += 1;
    const currentCount = this.count;

    const {
      stopNow = false,
      returnNow = false,
      ...state
    } = this.runPluginHandler('onBefore', params);

    // stop request
    if (stopNow) {
      return new Promise(() => {});
    }

    this.setState({
      loading: true,
      params,
      ...state,
    });

    // return now
    if (returnNow) {
      return Promise.resolve(state.data);
    }

    this.options.onBefore?.(params);

    try {
      // replace service
      let { servicePromise } = this.runPluginHandler('onRequest', this.serviceRef.current, params);

      if (!servicePromise) {
        servicePromise = this.serviceRef.current(...params);
      }

      const res = await servicePromise;

      if (currentCount !== this.count) {
        // prevent run.then when request is canceled
        return new Promise(() => {});
      }

      // const formattedResult = this.options.formatResultRef.current ? this.options.formatResultRef.current(res) : res;

      this.setState({
        data: res,
        error: undefined,
        loading: false,
      });

      this.options.onSuccess?.(res, params);
      this.runPluginHandler('onSuccess', res, params);

      this.options.onFinally?.(params, res, undefined);

      if (currentCount === this.count) {
        this.runPluginHandler('onFinally', params, res, undefined);
      }

      return res;
    } catch (error) {
      if (currentCount !== this.count) {
        // prevent run.then when request is canceled
        return new Promise(() => {});
      }

      this.setState({
        error,
        loading: false,
      });

      this.options.onError?.(error, params);
      this.runPluginHandler('onError', error, params);

      this.options.onFinally?.(params, undefined, error);

      if (currentCount === this.count) {
        this.runPluginHandler('onFinally', params, undefined, error);
      }

      throw error;
    }
  }

  run(...params: TParams) {
    this.runAsync(...params).catch((error) => {
      if (!this.options.onError) {
        console.error(error);
      }
    });
  }

  cancel() {
    this.count += 1;
    this.setState({
      loading: false,
    });

    this.runPluginHandler('onCancel');
  }

  refresh() {
    // @ts-ignore
    this.run(...(this.state.params || []));
  }

  refreshAsync() {
    // @ts-ignore
    return this.runAsync(...(this.state.params || []));
  }

  mutate(data?: TData | ((oldData?: TData) => TData | undefined)) {
    let targetData: TData | undefined;
    if (isFunction(data)) {
      // @ts-ignore
      targetData = data(this.state.data);
    } else {
      targetData = data;
    }

    this.runPluginHandler('onMutate', targetData);

    this.setState({
      data: targetData,
    });
  }
}
```
###  4.2. <a name='-1'></a>🌍 整体解读
首先是一个好消息:
```
import { isFunction } from '../../utils';
import type { MutableRefObject } from 'react';
import type { FetchState, Options, PluginReturn, Service, Subscribe } from './types';
```
`Fetch.ts`的所有`import`都在这里 ⬆️ , 显然这就是`ahooks/useRequest`的最里一层套娃了, 也是核心代码所在.

值得一提的是, 这和前两个文件不同, 是用`class`而非`function`. 浏览整个类的结构, 可以说是相当的清晰. 
- 状态管理: 几个很好理解的`state`和`setState`函数, 不另外解读了
- 构造方法: `constructor` 
- 唯一的辅助工具(插件处理): `runPluginHandler` 
- 核心部分(请求处理): `async`, `runAsync` 
- 上文六个返回方法中, 除`async`, `runAsync`外的五个
​
![](https://pic2.zhimg.com/80/v2-2729bc93dafb3b97536c56d6933763fd_1440w.jpg)

接来下, 就从`constructor`开始阅读. 
###  4.3. <a name='constructor'></a>构造 · `constructor`
```
  constructor(
    public serviceRef: MutableRefObject<Service<TData, TParams>>, // 第一个
    public options: Options<TData, TParams>, // 第二个
    public subscribe: Subscribe, // 第三个
    public initState: Partial<FetchState<TData, TParams>> = {}, // 第四个
  ) {
    this.state = {
      ...this.state, // 原来的参数不变
      loading: !options.manual, // 如果非手动触发, 那么现在就开始loading了
      ...initState, // 存入插件的初始化参数
    };
  }
```
⬆️ 上面四个入参对应`useRequestImplement`中实例化时的四个入参 ⬇️
```
  const fetchInstance = useCreation(() => {
    const initState = plugins.map((p) => p?.onInit?.(fetchOptions)).filter(Boolean); // 捞一捞插件的初始化参数

    return new Fetch<TData, TParams>(
      serviceRef, // 第一个
      fetchOptions, // 第二个
      update, // 第三个
      Object.assign({}, ...initState), // 第四个
    );
  }, []);
```
整体来看, 四个入参都被直接作为了`public`成员, 因而如`serviceRef`, `options`中`manual`以外的值, 虽然在`constructor`中没有做处理或保存, 但其实已经成为了`Fetch`的公共成员, 可以在其他成员函数中通过`this`直接调用. 

其实真正做的处理, 只有一个`loading`状态根据是否手动参数`manual`更新. 其他`initState`都是插件的初始化参数. 
###  4.4. <a name='runPluginHandler'></a>插件处理 · `runPluginHandler`
```
  runPluginHandler(event: keyof PluginReturn<TData, TParams>, ...rest: any[]) {
    // @ts-ignore
    const r = this.pluginImpls.map((i) => i[event]?.(...rest)).filter(Boolean);
    return Object.assign({}, ...r);
  }
```
这一方法将遍历`pluginImpls`所有插件, 执行其`event`指定的事件方法, 遍历后合并返回所有插件的返回.
###  4.5. <a name='runrunAsync'></a>✨ 核心方法 · `run` & `runAsync`
虽然`run`看起来也很核心, 但实际上不论是`run`还是`runAsync`, 实际都是调用`runAsync`, 看看`run`的源码就知道了 ⬇️
```
// run
  run(...params: TParams) {
    this.runAsync(...params).catch((error) => {
      if (!this.options.onError) {
        console.error(error);
      }
    });
  }
```
所以还是来看`runAsync` ⬇️ 
```
// runAsync
  async runAsync(...params: TParams): Promise<TData> {
    this.count += 1;
    const currentCount = this.count;

    const {
      stopNow = false,
      returnNow = false,
      ...state
    } = this.runPluginHandler('onBefore', params);

    // stop request
    if (stopNow) {
      return new Promise(() => {});
    }

    this.setState({
      loading: true,
      params,
      ...state,
    });

    // return now
    if (returnNow) {
      return Promise.resolve(state.data);
    }

    this.options.onBefore?.(params);

    try {
      // replace service
      let { servicePromise } = this.runPluginHandler('onRequest', this.serviceRef.current, params);

      if (!servicePromise) {
        servicePromise = this.serviceRef.current(...params);
      }

      const res = await servicePromise;

      if (currentCount !== this.count) {
        // prevent run.then when request is canceled
        return new Promise(() => {});
      }

      // const formattedResult = this.options.formatResultRef.current ? this.options.formatResultRef.current(res) : res;

      this.setState({
        data: res,
        error: undefined,
        loading: false,
      });

      this.options.onSuccess?.(res, params);
      this.runPluginHandler('onSuccess', res, params);

      this.options.onFinally?.(params, res, undefined);

      if (currentCount === this.count) {
        this.runPluginHandler('onFinally', params, res, undefined);
      }

      return res;
    } catch (error) {
      if (currentCount !== this.count) {
        // prevent run.then when request is canceled
        return new Promise(() => {});
      }

      this.setState({
        error,
        loading: false,
      });

      this.options.onError?.(error, params);
      this.runPluginHandler('onError', error, params);

      this.options.onFinally?.(params, undefined, error);

      if (currentCount === this.count) {
        this.runPluginHandler('onFinally', params, undefined, error);
      }

      throw error;
    }
  }
```
有点长, 需要分几个部分来读.
####  4.5.1. <a name='onBefore'></a>📍第一部分, 调用插件`onBefore`, 判断是否立即停止🤚
```
    this.count += 1; // 设置计数器 + 1
    const currentCount = this.count; // 设置当前是几号请求

    const {
      stopNow = false,
      returnNow = false,
      ...state
    } = this.runPluginHandler('onBefore', params);

    // stop request
    if (stopNow) {
      return new Promise(() => {});
    }
```
- 此时首先处理了`count`, 用于区分每一次请求, 即哪怕没开始就停止了, 也要算作一次请求, 避免混乱.
- 第二步, 运行插件的`onBefore`方法, 更新方法中的状态`state`并捞出`stopNow`, `returnNow`备用.
- 判断是否`stopNow`, 如果是, 那么就返回空对象, 请求结束了.

####  4.5.2. <a name='-1'></a>📍第二部分, 判断是否立即返回🦶
```
    this.setState({
      loading: true,
      params,
      ...state,
    });

    // return now
    if (returnNow) {
      return Promise.resolve(state.data);
    }
```
- 更新`Fetch`类的状态`state`, 其中`loading: true`, 请求开始, 同时将请求参数放入类状态中. 
- 判断上一部分捞回的`returnNow`, 如果是, 那么就立即返回方法中的状态中的`data`, 请求结束了. 

####  4.5.3. <a name='useRequestoptionsonBefore.'></a>📍第三部分, 运行`useRequest`中用户入参`options`中传入的`onBefore`方法.
```
    this.options.onBefore?.(params);
```
####  4.5.4. <a name='trycatch'></a>📍第四部分,  真正的请求——`try{}catch{} `
注意,  这一部分所有内容都被包在`try{}catch{}`中. 
##### 📍📍`try`的部分 
此时开始进入`onRequest`状态了, 意味着要调用所有插件中的`onRequest`方法更新服务, 然后发起请求 ⬇️
```
      // replace service
      let { servicePromise } = this.runPluginHandler('onRequest', this.serviceRef.current, params);

      if (!servicePromise) {
        servicePromise = this.serviceRef.current(...params);
      }

      const res = await servicePromise; // 发起请求!!!
```
此处真正的作用是更新`service`, 如果插件有返回新的服务, 则调用新的服务, 否则才使用原本`useRequestImplement`中的服务.

请求完了…… 但是如果此时请求已经被过期(即被取消或覆盖), 那么就只好返回空对象了 ⬇️
```
      if (currentCount !== this.count) {
        // prevent run.then when request is canceled
        return new Promise(() => {});
      }
```
更新参数 ⬇️
```
      // const formattedResult = this.options.formatResultRef.current ? this.options.formatResultRef.current(res) : res;

      this.setState({
        data: res,
        error: undefined,
        loading: false,
      });
```
有一个被注释掉了的`format`(?), 然后将返回结果赋值给`data`, 同时设置`loading: false`, 意味着请求已经结束.

最后就是成功后的调用和请求结束的调用啦 ⬇️
```
      this.options.onSuccess?.(res, params);
      this.runPluginHandler('onSuccess', res, params);

      this.options.onFinally?.(params, res, undefined);

      if (currentCount === this.count) {
        this.runPluginHandler('onFinally', params, res, undefined);
      }

      return res;
```
按`options.onSuccess`, 所有插件的`onSuccess`,  `options`. `onFinally`, 所有插件的`onFinally`依次调用. 当然, 如果当前请求已经过期, 那么最后一项插件的`onFinally`事件则不会再调用.

至此, 就是一切顺利, 返回`res`啦. 
##### 📍📍`catch`的部分
既然出问题了…… 那么先来判断一把请求是不是已经过期了. be like: pre已经结束了, ppt就算炸了也没有关系.
```
      if (currentCount !== this.count) {
        // prevent run.then when request is canceled
        return new Promise(() => {});
      }
```
如果请求已经过期, 那么返回空对象即可.

否则, 只能用`loading: false`宣告请求结束, 然后将`error`存入类状态中. ⬇️
```
      this.setState({
        error,
        loading: false,
      }); 
```
最后, 按和`try`结尾一模一样的逻辑依次调用`onError`, `onFinally`. 
```
      this.options.onError?.(error, params);
      this.runPluginHandler('onError', error, params);

      this.options.onFinally?.(params, undefined, error);

      if (currentCount === this.count) {
        this.runPluginHandler('onFinally', params, undefined, error);
      }

      throw error; 
```
最后抛出`error`(暗爽)!
####  4.5.5. <a name='runAsync'></a>🌍 关于`runAsync`, 小叨几句
`runAsync`的句子比较长, 但总体而言内容不多, 无非几步: `onBefore`, 检查是否停止, 检查是否立即返回, `try`请求…… 请求完, 检查请求是否过期, `onSuccess / onError`, `onFinally`. 

其中有四个`on`开头的事件, 而每到相应事件阶段, 都会分别调用用户从`useRequest`的`options`中传入的事件方法 和 所有插件中的事件方法. 从中不难注意到, 请求前的`onBefore`事件时是先遍历调用插件中的事件后调用用户传入的事件, 请求后的三个事件反之. 个人对此其实不太了解, 粗浅的理解了一下, 认为用户的事件在此总是比插件距离请求本身更近, 即用户具有最真实材料(即请求本身)的最终决定和第一个使用的权利, 插件的权限次之. 将事件跨度再拉大一些, 其实也是让用户的事件方法距离`useRequest`的出入参更远——试想, 刚拿到入参就做`onBefore`处理, 为什么不先处理了再给入参呢? 

好了, 核心请求逻辑都在上面. 接下来看看返回给用户的六个方法. 当然, 虽说有六个, 实则接下来要聊的只有四个, 因为其中`run`和`runAsync`已经读过了.
###  4.6. <a name='cancel'></a>取消 · `cancel`
```
  cancel() {
    this.count += 1;
    this.setState({
      loading: false,
    });

    this.runPluginHandler('onCancel');
  }
```
这里逻辑很简单, `计数器 + 1`, `loading: false`就强制将请求置为停止状态了, 当真正请求回来时对比计数器, 就会发现已经过期了~
###  4.7. <a name='refreshrefreshAsync'></a>更新 · `refresh` & `refreshAsync`
```
  refresh() {
    // @ts-ignore
    this.run(...(this.state.params || []));
  }

  refreshAsync() {
    // @ts-ignore
    return this.runAsync(...(this.state.params || []));
  } 
```
其实也就是重新`run`啦.
###  4.8. <a name='mutate'></a>`mutate`
```
  mutate(data?: TData | ((oldData?: TData) => TData | undefined)) {
    let targetData: TData | undefined;
    if (isFunction(data)) {
      // @ts-ignore
      targetData = data(this.state.data);
    } else {
      targetData = data;
    }

    this.runPluginHandler('onMutate', targetData);

    this.setState({
      data: targetData,
    });
  }
```
`mutate`用于手动修改`data`. 这一段逻辑也比较简单, 共三小步:
- 判断入参`data`是否是方法后进行调用或复制
- 遍历调用插件中的`onMutate`方法
- 更新类状态中的`data`为最新的值

目前, `useRequest`中的核心源码已经读完了, 能看到都是非常核心的功能模块, 而许多实用特性实际都在**插件**中. 这篇暂时不做展开, 下次一定.

---

`20220731` 次日更新:

`✨Start`安排, 插件马上就写.

first published on [知乎专栏](https://zhuanlan.zhihu.com/p/547965777), `20220730`.
