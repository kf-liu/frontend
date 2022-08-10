- [`ahooks useRequest` 8项默认`plugins`源码解读](#ahooks-userequest-8项默认plugins源码解读)
  - [插件在核心源码中的调用](#插件在核心源码中的调用)
    - [step1. 默认插件处理](#step1-默认插件处理)
    - [step2. 初始化与实例化](#step2-初始化与实例化)
      - [step2-1.](#step2-1)
      - [step2-2.](#step2-2)
    - [step3. 实例运行与事件触发](#step3-实例运行与事件触发)
  - [防抖 · useDebouncePlugin](#防抖--usedebounceplugin)
    - [源码](#源码)
    - [整体解读](#整体解读)
    - [核心思路](#核心思路)
    - [防抖替换](#防抖替换)
    - [副作用处理](#副作用处理)
    - [`onCancel`事件](#oncancel事件)
  - [防闪烁 · `useLoadingDelayPlugin`](#防闪烁--useloadingdelayplugin)
    - [源码](#源码-1)
    - [整体解读](#整体解读-1)
    - [核心动作 · `setTimeout`](#核心动作--settimeout)
    - [善后事宜](#善后事宜)
  - [轮询 · `usePollingPlugin`](#轮询--usepollingplugin)
    - [源码](#源码-2)
    - [整体解读](#整体解读-2)
    - [主链路 · `pollingInterval `](#主链路--pollinginterval-)
    - [页面隐藏与重新 · `pollingWhenHidden`](#页面隐藏与重新--pollingwhenhidden)
    - [持续错误重试次数 · `pollingErrorRetryCount`](#持续错误重试次数--pollingerrorretrycount)
  - [屏幕聚焦重新请求 · `useRefreshOnWindowFocusPlugin`](#屏幕聚焦重新请求--userefreshonwindowfocusplugin)
    - [源码](#源码-3)
    - [整体解读](#整体解读-3)
    - [重新聚焦 · `subscribeFocus`](#重新聚焦--subscribefocus)
    - [间隔时间 · `limit`](#间隔时间--limit)
    - [整体逻辑](#整体逻辑)
  - [节流 · `useThrottlePlugin`](#节流--usethrottleplugin)
    - [防抖 VS 节流](#防抖-vs-节流)
    - [源码](#源码-4)
  - [`useAutoRunPlugin`](#useautorunplugin)
    - [源码](#源码-5)
    - [入参](#入参)
    - [核心变量](#核心变量)
    - [自动化与初始化](#自动化与初始化)
    - [整体逻辑](#整体逻辑-1)
  - [缓存 & SWR · `useCachePlugin`](#缓存--swr--usecacheplugin)
    - [源码](#源码-6)
    - [缓存 · `utils/cache`](#缓存--utilscache)
    - [请求缓存 · `utils/cachePromise`](#请求缓存--utilscachepromise)
    - [预定缓存 · `utils/cacheSubscribe `](#预定缓存--utilscachesubscribe-)

# `ahooks useRequest` 8项默认`plugins`源码解读
上次已经聊过`ahooks/useRequest`的核心源码了, 这次就来专门看看插件, 毕竟不少功能特性都在插件中实现, 并没有集成到主链路中.
附: 核心源码解读指路 ➡️ [`ahooks/useRequest` 源码解读](../)

首先来review一下插件在核心源码中的调用链路.
## 插件在核心源码中的调用
从`useRequest`到`useRequestImplement`, `plugins`入参一路随行, 直到实例化`Fetch`时才开始进行初始化、实例化, 然后在`Fetch`内的各个生命周期中被使用. 
### step1. 默认插件处理
从useRequest到useRequestImplement, 加入默认插件:
```
    useDebouncePlugin,
    useLoadingDelayPlugin,
    usePollingPlugin,
    useRefreshOnWindowFocusPlugin,
    useThrottlePlugin,
    useAutoRunPlugin,
    useCachePlugin,
    useRetryPlugin,
```
这也是我们这次要读的插件.
### step2. 初始化与实例化
`useRequestImplement`中, 初始化.

先顺路看一眼参数的处理. 其中`options`就是`useRequest`中原封不动传入的第一个参数`options`. 这段代码是在`useRequestImplement`的开头, 用于添加默认非手动请求的配置`manual = false`, 其他保持不变, 并存为新的配置项`fetchOptions`.

需要注意, 之后所有配置项都使用更新了默认配置的`fetchOptions`, 不再提及`options`.
```
  const { manual = false, ...rest } = options;

  const fetchOptions = {
    manual,
    ...rest,
  };
```
#### step2-1.
`useRequestImplement`中, 实例化`Fetch`前, `onInit`调用.`
```
  const fetchInstance = useCreation(() => {
    // 注意下面这一行
    const initState = plugins.map((p) => p?.onInit?.(fetchOptions)).filter(Boolean);

    return new Fetch<TData, TParams>(
      serviceRef,
      fetchOptions,
      update,
      Object.assign({}, ...initState),
    );
  }, []);
```
注意, 这里的`onInit`是一个静态方法, 是在插件实例化之前直接调用的, 之后也会看到它在函数体内的写法也会有所不同. 从`Plugin`的`type`中可见一斑. ⬇️`
```
export type Plugin<TData, TParams extends any[]> = {
  (fetchInstance: Fetch<TData, TParams>, options: Options<TData, TParams>): PluginReturn<
    TData,
    TParams
  >;
  onInit?: (options: Options<TData, TParams>) => Partial<FetchState<TData, TParams>>;
};
```
#### step2-2.
`useRequestImplement`中, 实例化`Fetch`后, 对插件进行遍历调用, 并将实例传入`fetchInstance.pluginImpls`.
```
  // run all plugins hooks
  fetchInstance.pluginImpls = plugins.map((p) => p(fetchInstance, fetchOptions));
```
可见, 插件的入参`fetchOptions`实则与`useRequest`中的`options`同一类型, 也是从`useRequest`中获取过来.

可以对比上文`type Plugin中的options: Options<TData, TParams>)`和下面`useRequest`的函数签名`options?: Options<TData, TParams>`. 
```
function useRequest<TData, TParams extends any[]>(service: Service<TData, TParams>, options?: Options<TData, TParams>, plugins?: Plugin<TData, TParams>[]): Result<TData, TParams>
```
### step3. 实例运行与事件触发
此时插件已经在`fetchInstance`中跑起来了, 它可以根据自身逻辑对`fetchInstance`的方法和参数进行取用, 也会被每一次`fetchInstance`请求前后的事件触发.

前者需要等会儿阅读每个插件中的具体逻辑才能知晓, 后者则在`Fetch`的核心逻辑中已经提及过, 有`onBefore`, `onRequest`, `onSuccess`, `onError`, `onFinally`, `onCancel`, `onMutate`七种事件类型. 

加上前面的`onInit`, 共八种事件类型. 

接下来, 就按默认插件的顺序瞧一瞧看一看.
## 防抖 · useDebouncePlugin 
### 源码
不长
```
import type { DebouncedFunc, DebounceSettings } from 'lodash';
import debounce from 'lodash/debounce';
import { useEffect, useMemo, useRef } from 'react';
import type { Plugin } from '../types';

const useDebouncePlugin: Plugin<any, any[]> = (
  fetchInstance,
  { debounceWait, debounceLeading, debounceTrailing, debounceMaxWait },
) => {
  const debouncedRef = useRef<DebouncedFunc<any>>();

  const options = useMemo(() => {
    const ret: DebounceSettings = {};
    if (debounceLeading !== undefined) {
      ret.leading = debounceLeading;
    }
    if (debounceTrailing !== undefined) {
      ret.trailing = debounceTrailing;
    }
    if (debounceMaxWait !== undefined) {
      ret.maxWait = debounceMaxWait;
    }
    return ret;
  }, [debounceLeading, debounceTrailing, debounceMaxWait]);

  useEffect(() => {
    if (debounceWait) {
      const _originRunAsync = fetchInstance.runAsync.bind(fetchInstance);

      debouncedRef.current = debounce(
        (callback) => {
          callback();
        },
        debounceWait,
        options,
      );

      // debounce runAsync should be promise
      // https://github.com/lodash/lodash/issues/4400#issuecomment-834800398
      fetchInstance.runAsync = (...args) => {
        return new Promise((resolve, reject) => {
          debouncedRef.current?.(() => {
            _originRunAsync(...args)
              .then(resolve)
              .catch(reject);
          });
        });
      };

      return () => {
        debouncedRef.current?.cancel();
        fetchInstance.runAsync = _originRunAsync;
      };
    }
  }, [debounceWait, options]);

  if (!debounceWait) {
    return {};
  }

  return {
    onCancel: () => {
      debouncedRef.current?.cancel();
    },
  };
};

export default useDebouncePlugin;
```
### 整体解读  


由开头两行可知, 其防抖能力是基于`'lodash/debounce'`进行封装. 几个入参实则也都用于传入`'lodash/debounce'`, 功能如下, 简单明确.

| 参数             | 说明                                           | 类型    | 默认值 |
| ---------------- | ---------------------------------------------- | ------- | ------ |
| debounceWait     | 防抖等待时间, 单位为毫秒，设置后，进入防抖模式 | number  | -      |
| debounceLeading  | 在延迟开始前执行调用                           | boolean | false  |
| debounceTrailing | 在延迟结束后执行调用                           | boolean | true   |
| debounceMaxWait  | 允许被延迟的最大值                             | number  | -      |

函数体内分三个部分:

1. 配置: 根据函数后三个入参对lodash/debounce入参options进行配置
1. 核心功能: 若有debounceWait入参, 则开启防抖, 这也是本插件的核心部分, 均包裹在一个依赖项为[debounceWait, options]的useEffect中
1. 返回: 若开启防抖, 则返回onCancel方法, 否则返回空对象
可见, 这一函数的核心参数为debounceWait, 其不仅是lodash/debounce的入参, 也是否开启防抖的标志.
这里分析其核心功能.
### 核心思路
这一函数中最常出现的就是`debouncedRef`, 在函数开头就用`useRef`进行了定义, 但里面空空如也. `
```
  const debouncedRef = useRef<DebouncedFunc<any>>();
```
而在核心功能部分中, 对其current进行了赋值, 读懂它很重要.`
```
      debouncedRef.current = debounce(
        (callback) => {
          callback();
        },
        debounceWait,
        options,
      );
```
可以看到, 这个是一个防抖请求, 入参为一个回调函数, 出参为该回调函数的调用. 简单来说, 这就是一个防抖的函数模版, 不论放入什么请求, 都会带着`debounceWait`和`options`参数以防抖的形式拉取该请求. 这意味着, 如果将我们的`runAsnyc`传入`debouncedRef.current`, 则可以让它以防抖的方式被调用.

核心思路🈶️. 至于其他的代码, 都是具体实现了.

### 防抖替换
既然要修改`runAsnyc`, 在上面`debouncedRef.current`之前, 我们先将原本的`runAsnyc`定义为`_originRunAsync`, 以便在修改后还可以调用原来的函数. 
```
      const _originRunAsync = fetchInstance.runAsync.bind(fetchInstance);
```
然后就是紧接着上面debouncedRef.current的,  替换掉原本`fetchInstance.runAsync`. 
```
      // debounce runAsync should be promise
      // https://github.com/lodash/lodash/issues/4400#issuecomment-834800398
      fetchInstance.runAsync = (...args) => {
        return new Promise((resolve, reject) => {
          debouncedRef.current?.(() => {
            _originRunAsync(...args)
              .then(resolve)
              .catch(reject);
          });
        });
      };
```
并将`fetchInstance.runAsync`设为了一个新的`Promise`函数, 返回防抖后的原本的它(`_originRunAsync`). 于是现在的它, 就是防抖的它了.
### 副作用处理
当然, 我们也要考虑副作用, 在`useEffect`的回调函数中调用`cancel()`并取消防抖, 换回原来的`runAsync`. 
```
      return () => {
        debouncedRef.current?.cancel();
        fetchInstance.runAsync = _originRunAsync;
      };
```
### `onCancel`事件
防抖插件中只有这一个事件, 也是因为请求本身实际也只涉及这一个事件. 其他事件在`runAsnyc`中就可以做完整的处理(如请求前后事件), 或状态与请求本身无关(如`mutate`只需要改变`Fetch`中的类状态即可). 而`onCancel`是通过改变类状态来影响`runAsnyc`中的判断, 横跨了两者. 
因此, 如果开启了防抖, 那么返回相应的取消请求事件.
```
  if (!debounceWait) {
    return {};
  }

  return {
    onCancel: () => {
      debouncedRef.current?.cancel();
    },
  };
```
## 防闪烁 · `useLoadingDelayPlugin` 
### 源码
```
import { useRef } from 'react';
import type { Plugin, Timeout } from '../types';

const useLoadingDelayPlugin: Plugin<any, any[]> = (fetchInstance, { loadingDelay }) => {
  const timerRef = useRef<Timeout>();

  if (!loadingDelay) {
    return {};
  }

  const cancelTimeout = () => {
    if (timerRef.current) {
      clearTimeout(timerRef.current);
    }
  };

  return {
    onBefore: () => {
      cancelTimeout();

      timerRef.current = setTimeout(() => {
        fetchInstance.setState({
          loading: true,
        });
      }, loadingDelay);

      return {
        loading: false,
      };
    },
    onFinally: () => {
      cancelTimeout();
    },
    onCancel: () => {
      cancelTimeout();
    },
  };
};

export default useLoadingDelayPlugin;
```
### 整体解读
这是一个防止因为请求返回的太快, `loading = true`的时间太短, 而导致页面的`loading`圈圈一闪而过(即闪烁)的插件, 只有一个入参.
| 参数         | 说明                              | 类型   | 默认值 |
| ------------ | --------------------------------- | ------ | ------ |
| loadingDelay | 设置 loading 变成 true 的延迟时间 | number | 0      |

显然, 这里要做的, 就是让`loading`状态慢一点更新为`true`, 如果在慢一点的那会儿就请求完了, 正好就不用`loading`了. 因此, 它影响的应该是`onBefore`事件时的状态更新——先不让`loading`状态更新, 并设置一个计时器, 若在计时器结束后请求仍然没有结束, 再设置`loading: true`.

当然, 如果没有传入`loadingDelay`, 则无需防闪烁, 返回空对象跑路. 

同样, 在函数的开头也定义了一个最新的计时器`timerRef` ⬇️ 
```
  const timerRef = useRef<Timeout>();
```
### 核心动作 · `setTimeout` 
```
    onBefore: () => {
      cancelTimeout();

      timerRef.current = setTimeout(() => {
        fetchInstance.setState({
          loading: true,
        });
      }, loadingDelay);

      return {
        loading: false,
      };
    },
```
在`onBefore`事件被触发时, 插件将先被调用该事件, 其返回值也将和默认的`loading: true`一同被放入类状态. 此时插件只需要返回`loading: false`即可覆盖默认设置, `loading`状态仍为`false`. 
因此`useLoadingDelayPlugin`在`onBefore`事件中返回的对象为`{ loading: false }`. 
```
// Fetch.runAsnyc 在onBefore阶段的源码
// 忘了的话可以看一看
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
```
当然, 返回前还是要先把计时器开好. 即先重置(取消原有的定时), 并开启新的计时, 结束后调用`fetchInstance.setState({ loading: true });`设置`loading`状态即可.
### 善后事宜
如果请求完成或取消了, 那么即便计时到了也不应该再调用状态更新, 所以在`onFinally,` `onCancel`事件触发时, 需要将计时器取消.
## 轮询 · `usePollingPlugin`
### 源码
```
import { useRef } from 'react';
import useUpdateEffect from '../../../useUpdateEffect';
import type { Plugin, Timeout } from '../types';
import isDocumentVisible from '../utils/isDocumentVisible';
import subscribeReVisible from '../utils/subscribeReVisible';

const usePollingPlugin: Plugin<any, any[]> = (
  fetchInstance,
  { pollingInterval, pollingWhenHidden = true, pollingErrorRetryCount = -1 },
) => {
  const timerRef = useRef<Timeout>();
  const unsubscribeRef = useRef<() => void>();
  const countRef = useRef<number>(0);

  const stopPolling = () => {
    if (timerRef.current) {
      clearTimeout(timerRef.current);
    }
    unsubscribeRef.current?.();
  };

  useUpdateEffect(() => {
    if (!pollingInterval) {
      stopPolling();
    }
  }, [pollingInterval]);

  if (!pollingInterval) {
    return {};
  }

  return {
    onBefore: () => {
      stopPolling();
    },
    onError: () => {
      countRef.current += 1;
    },
    onSuccess: () => {
      countRef.current = 0;
    },
    onFinally: () => {
      if (
        pollingErrorRetryCount === -1 ||
        // When an error occurs, the request is not repeated after pollingErrorRetryCount retries
        (pollingErrorRetryCount !== -1 && countRef.current <= pollingErrorRetryCount)
      ) {
        // if pollingWhenHidden = false && document is hidden, then stop polling and subscribe revisible
        if (!pollingWhenHidden && !isDocumentVisible()) {
          unsubscribeRef.current = subscribeReVisible(() => {
            fetchInstance.refresh();
          });
          return;
        }

        timerRef.current = setTimeout(() => {
          fetchInstance.refresh();
        }, pollingInterval);
      } else {
        countRef.current = 0;
      }
    },
    onCancel: () => {
      stopPolling();
    },
  };
};

export default usePollingPlugin;
```
### 整体解读
简而言之, 其重点在于多久轮询一次, 然后每次请求结束后(即`onFinally`)设个定时器重新发起请求, 便可以循环往复. 当然, 也可以加一些其他的配置, 比如: 总是连续请求失败是不是就不查了? 或者页面都隐藏了还要查吗? 于是就有了以下入参.
| 参数                   | 说明                                                               | 类型                               | 默认值  |
| ---------------------- | ------------------------------------------------------------------ | ---------------------------------- | ------- |
| pollingInterval        | 轮询间隔，单位为毫秒。如果值大于 0，则启动轮询模式。               | number                             | 0       |
| pollingWhenHidden      | 在页面隐藏时，是否继续轮询。如果设置为 false，在页面隐藏时会暂时停 | 轮询，页面重新显示时继续上次轮询。 | boolean | true |
| pollingErrorRetryCount | 轮询错误重试次数。如果设置为 -1，则无限次                          | number                             | -1      |

当然, 还有些老套路, 比如没有`pollingInterval`入参则不轮询, 此时就直接返回空对象了.
### 主链路 · `pollingInterval `
一个最新的计时器:
```
  const timerRef = useRef<Timeout>();
```
计时器重置备用(注意看下面注释的标号):`
```
// 1
  const stopPolling = () => {
    if (timerRef.current) {
      clearTimeout(timerRef.current);
    }
    unsubscribeRef.current?.();
  };

// ……

// 2
  useUpdateEffect(() => {
    if (!pollingInterval) {
      stopPolling();
    }
  }, [pollingInterval]);

// ……

// 3
    onBefore: () => {
      stopPolling();
    },

// ……

// 4
    onCancel: () => {
      stopPolling();
    },
```
提前定义好停止轮询的方法 在里面`clearTimeout`

若入参更新为`0`, 就不用轮询了. 这里的`useUpdateEffect`是只在依赖更新时执行的`useEffect`. 

在`onBefore`事件中停止之前的轮询, 准备开启新一轮
如果取消了自然也就不用轮询了
一切就绪, 可以来到`onFinally`事件触发的现场了. 核心代码:
```
        timerRef.current = setTimeout(() => {
          fetchInstance.refresh();
        }, pollingInterval); 
```
计时器轮询时间一到, 就调用`fetchInstance.refresh();`重新发起请求.

至此, 都只用到了`pollingInterval`一个参数, 接下来再来看看其他功能.
### 页面隐藏与重新 · `pollingWhenHidden`
`pollingWhenHidden`默认为`true`, 一直轮询. 但也可以手动设置为`false`, 当页面隐藏时暂停轮询, 页面重现时重启轮询.

这里显然需要依赖两个工具:
- 对窗口是否隐藏的判断import isDocumentVisible from '../utils/isDocumentVisible'; 
- 窗口出现的回调import subscribeReVisible from '../utils/subscribeReVisible'; 

两个源码都不长, 可以简单过目1 ⬇️ 

`isDocumentVisible.ts`
```
import isBrowser from '../../../utils/isBrowser';

export default function isDocumentVisible(): boolean {
  if (isBrowser) {
    return document.visibilityState !== 'hidden';
  }
  return true;
}
```
`subscribeReVisible.ts` 
```
import isBrowser from '../../../utils/isBrowser';
import isDocumentVisible from './isDocumentVisible';

const listeners: any[] = [];

function subscribe(listener: () => void) {
  listeners.push(listener);
  return function unsubscribe() {
    const index = listeners.indexOf(listener);
    listeners.splice(index, 1);
  };
}

if (isBrowser) {
  const revalidate = () => {
    if (!isDocumentVisible()) return;
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i];
      listener();
    }
  };
  window.addEventListener('visibilitychange', revalidate, false);
}

export default subscribe;
```
雷打不动, 我们先`useRef`来存储一个最新的页面可见时的调用函数. 
```
  const unsubscribeRef = useRef<() => void>();
```
有了它们, 就可以在`onFinally`中先对页面是否隐藏进行判断, 如有必要(页面不可见, 要等可见后再刷新), 就使用将`fetchInstance.refresh();`重新请求方法放入页面可见后的回调`subscribeReVisible`中, 存储到`unsubscribeRef.current`. 
```
        // if pollingWhenHidden = false && document is hidden, then stop polling and subscribe revisible
        if (!pollingWhenHidden && !isDocumentVisible()) {
          unsubscribeRef.current = subscribeReVisible(() => {
            fetchInstance.refresh();
          });
          return;
        } 
```
于是, 如果页面重新可见, 便会自动进行调用`fetchInstance.refresh();`了.

但还没完, 还有一处值得注意 ⬇️
```
  const stopPolling = () => {
    if (timerRef.current) {
      clearTimeout(timerRef.current);
    }
    unsubscribeRef.current?.();
  };
```
在停止轮询`stopPolling`方法被调用时, 则会将其置空. (这里个人理解得不甚清晰, 就不具体阐述了, 欢迎探讨)
### 持续错误重试次数 · `pollingErrorRetryCount` 
此功能的实现清晰简明, 使用一个总是最新值的计数器做累加、清零和比较即可.
```
  const countRef = useRef<number>(0); 
```
即: 每当出现便加一, 成功便清零, 每次准备轮询前做一次比较.
```
// 1
    onError: () => {
      countRef.current += 1;
    },
// 2
    onSuccess: () => {
      countRef.current = 0;
    },
// 3
    onFinally: () => {
      if (
        // 3-1
        pollingErrorRetryCount === -1 ||
        // 3-2
        // When an error occurs, the request is not repeated after pollingErrorRetryCount retries
        (pollingErrorRetryCount !== -1 && countRef.current <= pollingErrorRetryCount)
      ) {

        // ……

      } else {
        countRef.current = 0;
      }
    },
```
1. 出现错误: 加一
1. 成功: 清零
1. 轮询前: 比较
    1. 是否不设置错误次数上限, 若为-1则直接准备轮询 
    1. 当前错误次数是否超出上限, 是则不轮询并将错误计数器重置为0, 否则继续轮询
## 屏幕聚焦重新请求 · `useRefreshOnWindowFocusPlugin`
有了上面的轮询为基础, 这一插件的实现难度好像不值一提, 甚至乍一看起来没有必要分为两个插件.

——但是为什么还需要一个这样的插件呢?

个人理解, 一方面是单一职责原则, 故不能耦合. 另一方面是其具体调用逻辑实则是不同的, 具体看看源码就知道了.

——而且, 其中的逻辑也轮询也是有所不同的.

### 源码
```
import { useEffect, useRef } from 'react';
import useUnmount from '../../../useUnmount';
import type { Plugin } from '../types';
import limit from '../utils/limit';
import subscribeFocus from '../utils/subscribeFocus';

const useRefreshOnWindowFocusPlugin: Plugin<any, any[]> = (
  fetchInstance,
  { refreshOnWindowFocus, focusTimespan = 5000 },
) => {
  const unsubscribeRef = useRef<() => void>();

  const stopSubscribe = () => {
    unsubscribeRef.current?.();
  };

  useEffect(() => {
    if (refreshOnWindowFocus) {
      const limitRefresh = limit(fetchInstance.refresh.bind(fetchInstance), focusTimespan);
      unsubscribeRef.current = subscribeFocus(() => {
        limitRefresh();
      });
    }
    return () => {
      stopSubscribe();
    };
  }, [refreshOnWindowFocus, focusTimespan]);

  useUnmount(() => {
    stopSubscribe();
  });

  return {};
};

export default useRefreshOnWindowFocusPlugin;
```
### 整体解读
轮询中的页面是否隐藏是通过`subscribeReVisible`监听`visibilitychange`事件, 而屏幕聚焦重新请求是通过`subscribeFocus`监听`visibilitychange`和`focus`事件, 即轮询只在页面是否可见发生变化时, 才会关注, 而屏幕聚焦重新请求不仅关注页面是否可见, 也关注页面是否聚焦. 自然, 轮询中就不便直接复用屏幕聚焦插件了.

好的, 来看看参数.
| 参数                 | 说明                                         | 类型    | 默认值 |
| -------------------- | -------------------------------------------- | ------- | ------ |
| refreshOnWindowFocus | 在屏幕重新获取焦点或重新显示时，重新发起请求 | boolean | false  |
| focusTimespan        | 重新请求间隔，单位为毫秒                     | number  | 5000   |

必填参数只有一个, `refreshOnWindowFocus`. 

这里的整体逻辑也是比较简单, 监听以上两个入参的变化, 并在`refreshOnWindowFocus`为`true`时, 给屏幕重新聚焦事件传入一个回调——当间隔时间在`focusTimespan`内时, 不做请求, 否则调用`refresh`方法即可. 
这里的涉及两个依赖工具, 一是方才反复提到的`subscribeFocus`回调函数, 一是判断间隔时间的函数`limit`.
### 重新聚焦 · `subscribeFocus`
那么先扫一眼`subscribeFocus`, 它和前文的`subscribeReVisible`实现方法一致, 只是多监听了一个窗口事件.
```
// from swr
import isBrowser from '../../../utils/isBrowser';
import isDocumentVisible from './isDocumentVisible';
import isOnline from './isOnline';

const listeners: any[] = [];

function subscribe(listener: () => void) {
  listeners.push(listener);
  return function unsubscribe() {
    const index = listeners.indexOf(listener);
    listeners.splice(index, 1);
  };
}

if (isBrowser) {
  const revalidate = () => {
    if (!isDocumentVisible() || !isOnline()) return;
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i];
      listener();
    }
  };
  window.addEventListener('visibilitychange', revalidate, false);
  window.addEventListener('focus', revalidate, false);
}

export default subscribe;
```
### 间隔时间 · `limit` 
然后主要来看看`limit`. ``
```
export default function limit(fn: any, timespan: number) {
  let pending = false;
  return (...args: any[]) => {
    if (pending) return;
    pending = true;
    fn(...args);
    setTimeout(() => {
      pending = false;
    }, timespan);
  };
}
```
其入参为一个函数`fn`和时间间隔`timespan`, 内部主要使用了一个`pending`变量以判断当前时间间隔状态.

每次调用`limit`函数, 都将先判断与上一次调用之间的时间间隔, 若还在`timespan`内, 将直接返回, 不再继续执行, 否则将`pending`标志设为`true`表示还未过时间间隔, 正常调用`fn`函数,  随后调用一个计时器, 在指定的`timespan`事件后, 再将`pending`标志设为`false`, 表示时间间隔已过. 
由这里可见, 此处的时间间隔是`fn`函数两次调用之间的时间间隔.

向上一层, 则是`limit(fetchInstance.refresh.bind(fetchInstance), focusTimespan);`, 即两次`refresh`之间的调用间隔.

再向上一层, 则是`subscribeFocus(() => { limitRefresh(); });`的时间间隔.

可推知, timespan参数的时间间隔, 应当是两次屏幕聚焦事件之间的时间间隔.
不过此处官方文档的表述为:

> 你可以点击浏览器外部，再点击当前页面来体验效果（或者隐藏当前页面，重新展示），如果和上一次请求间隔大于 5000ms，则会重新请求一次。

此处我推测的两次屏幕聚焦事件之间的时间间隔和 文档中的上一次请求间隔应当还是些许差别,  欢迎探讨.
### 整体逻辑
其实这一插件的高光时刻是在`limit`函数的, 不过还是要盘盘整体的逻辑.
生命周期.
开始:
```
   const unsubscribeRef = useRef<() => void>();
```
结束:
```
  const stopSubscribe = () => {
    unsubscribeRef.current?.();
  };

  useEffect(() => {

    // ……

    return () => {
      stopSubscribe();
    };
  }, [refreshOnWindowFocus, focusTimespan]);

  useUnmount(() => {
    stopSubscribe();
  });
```
核心逻辑:
```
    if (refreshOnWindowFocus) {
      const limitRefresh = limit(fetchInstance.refresh.bind(fetchInstance), focusTimespan);
      unsubscribeRef.current = subscribeFocus(() => {
        limitRefresh();
      });
    }
```
很短, 了解`limit`和`subscribeFocus`之后, 也很好理解了.

如果开启屏幕聚焦重新请求, 则将刷新方法`refresh`和时间间隔参数`timespan`放入`limit`中, 在屏幕聚焦时, 发起`limit`返回的`limitRefresh`请求即可.
## 节流 · `useThrottlePlugin` 
这个函数乍一看和防抖几乎一模一样, 且是以`'lodash/throttle'`为基础进行封装.
### 防抖 VS 节流
可是它们区别在哪呢, 看看lodash的官方文档怎么说. 

防抖:
> 创建一个 debounced（防抖动）函数，该函数会从上一次被调用后，延迟wait毫秒后调用func方法。

节流:
> 创建一个节流函数，在 wait 秒内最多执行func一次的函数。

此外, 文档中还提供了[David Corbacho's article](https://css-tricks.com/debouncing-throttling-explained-examples/)来让大家了解`_.throttle`与`_.debounce`的区别. 文章结尾的总结为:
> debounce: Grouping a sudden burst of events (like keystrokes) into a single one.
> throttle: Guaranteeing a constant flow of executions every X milliseconds. Like checking every 200ms your scroll position to trigger a CSS animation.

防抖是将特定时间内反复触发的事件合并为一个, 节流则是每隔恒定的特定时间执行一次.
### 源码
```
import type { DebouncedFunc, ThrottleSettings } from 'lodash';
import throttle from 'lodash/throttle';
import { useEffect, useRef } from 'react';
import type { Plugin } from '../types';

const useThrottlePlugin: Plugin<any, any[]> = (
  fetchInstance,
  { throttleWait, throttleLeading, throttleTrailing },
) => {
  const throttledRef = useRef<DebouncedFunc<any>>();

  const options: ThrottleSettings = {};
  if (throttleLeading !== undefined) {
    options.leading = throttleLeading;
  }
  if (throttleTrailing !== undefined) {
    options.trailing = throttleTrailing;
  }

  useEffect(() => {
    if (throttleWait) {
      const _originRunAsync = fetchInstance.runAsync.bind(fetchInstance);

      throttledRef.current = throttle(
        (callback) => {
          callback();
        },
        throttleWait,
        options,
      );

      // throttle runAsync should be promise
      // https://github.com/lodash/lodash/issues/4400#issuecomment-834800398
      fetchInstance.runAsync = (...args) => {
        return new Promise((resolve, reject) => {
          throttledRef.current?.(() => {
            _originRunAsync(...args)
              .then(resolve)
              .catch(reject);
          });
        });
      };

      return () => {
        fetchInstance.runAsync = _originRunAsync;
        throttledRef.current?.cancel();
      };
    }
  }, [throttleWait, throttleLeading, throttleTrailing]);

  if (!throttleWait) {
    return {};
  }

  return {
    onCancel: () => {
      throttledRef.current?.cancel();
    },
  };
};

export default useThrottlePlugin;
```
可以看看, 节流和防抖的源码真的可以说是一模一样, 就不读了. 下一个.
## `useAutoRunPlugin` 
没专门中文名的插件, 因为功能比较重要和基本. 看看源码, 就会发现都是熟悉的入参~
### 源码
```
import { useRef } from 'react';
import useUpdateEffect from '../../../useUpdateEffect';
import type { Plugin } from '../types';

// support refreshDeps & ready
const useAutoRunPlugin: Plugin<any, any[]> = (
  fetchInstance,
  { manual, ready = true, defaultParams = [], refreshDeps = [], refreshDepsAction },
) => {
  const hasAutoRun = useRef(false);
  hasAutoRun.current = false;

  useUpdateEffect(() => {
    if (!manual && ready) {
      hasAutoRun.current = true;
      fetchInstance.run(...defaultParams);
    }
  }, [ready]);

  useUpdateEffect(() => {
    if (hasAutoRun.current) {
      return;
    }
    if (!manual) {
      hasAutoRun.current = true;
      if (refreshDepsAction) {
        refreshDepsAction();
      } else {
        fetchInstance.refresh();
      }
    }
  }, [...refreshDeps]);

  return {
    onBefore: () => {
      if (!ready) {
        return {
          stopNow: true,
        };
      }
    },
  };
};

useAutoRunPlugin.onInit = ({ ready = true, manual }) => {
  return {
    loading: !manual && ready,
  };
};

export default useAutoRunPlugin;
```
### 入参
首先来看到入参:
```
  { manual, ready = true, defaultParams = [], refreshDeps = [], refreshDepsAction },
```
不难推测, 这一插件需要处理的是非手动请求(自动请求)和默认请求参数及其依赖.
其中一部分参数的作用, 在官方文档的基础用法中:
| 参数          | 说明                                                                                                  | 类型    | 默认值 |
| ------------- | ----------------------------------------------------------------------------------------------------- | ------- | ------ |
| manual        | 默认 false。 即在初始化时自动执行 service。如果设置为 true，则需要手动调用 run 或 runAsync 触发执行。 | boolean | false  |
| defaultParams | 首次默认执行时，传递给 service 的参数                                                                 | TParams | -      |
另外还有Ready:
| 参数  | 说明                 | 类型    | 默认值 |
| ----- | -------------------- | ------- | ------ |
| ready | 当前请求是否准备好了 | boolean | true   |
刷新依赖:
| 参数        | 说明                                                              | 类型  | 默认值 |
| ----------- | ----------------------------------------------------------------- | ----- | ------ |
| refreshDeps | 依赖数组，当数组内容变化后，发起请求。同 useEffect 的第二个参数。 | any[] | []     |
以及官方文档中无处查找的…… `refreshDepsAction`. (有意思起来了)
### 核心变量
```
  const hasAutoRun = useRef(false);
  hasAutoRun.current = false;
```
这个套路已经很熟悉了, 但还是要先看看这个默认值为`false`的`hasAutoRun.current`——它记录着是否已经发起过请求, 当然要记住它, 后面要用. 
### 自动化与初始化
通过这几个参数, 可以推知发起自动请求需要两个条件——`!manual && ready`. 
一般而言, `manual`参数不会变化——或者说从定义上而言就不应该变化也没必要. 而`ready`参数如果有使用到就应当是会变化, 且一般是由`false`变为`true`的, 否则默认就是`true`(准备好了). 
由于`useRequest`一开始被注册时, 插件就跑起来了. 那么在一开始初始化(`onInit`事件触发)时, 当前插件就可以拿到`manual`和`ready`两个参数, 并在如果要发起自动请求的情况下开始`loading`. ⬇️
```
useAutoRunPlugin.onInit = ({ ready = true, manual }) => {
  return {
    loading: !manual && ready,
  };
}; 
```
同理, 在`onBefore`事件触发时, 如果`!ready`, 将立即返回`stopNow: true`停止请求.
```
  return {
    onBefore: () => {
      if (!ready) {
        return {
          stopNow: true,
        };
      }
    },
  };  
```
而插件真正跑起来后, 是否自动开始请求, 则只需要监听`ready`, 并判断`manual`即可. 
```
  useUpdateEffect(() => {
    if (!manual && ready) {
      hasAutoRun.current = true;
      fetchInstance.run(...defaultParams);
    }
  }, [ready]);
```
从代码可以看到, 的确也是这样做的, 依赖项为ready, 判断`!manual && ready`.

如果发起自动请求,  则会设置`hasAutoRun.current = true`(后面有用), 同时调用`run`方法. 这里不难发现, 如果`ready`反复在真假之间横跳, 那么就会多次发起`run`的调用. (注意, 这里不是`refresh`)
依赖更新
另外有一段依赖于`[...refreshDeps]`的`useUpdateEffect`代码, 看起来很有意思.
```
   useUpdateEffect(() => {
    if (hasAutoRun.current) {
      return;
    }
    if (!manual) {
      hasAutoRun.current = true;
      if (refreshDepsAction) {
        refreshDepsAction();
      } else {
        fetchInstance.refresh();
      }
    }
  }, [...refreshDeps]);
```
当依赖更新时的处理逻辑是这样:
Begin.
1. 判断`hasAutoRun.current`是否为真
     1. 是, 则直接结束, End.
1. 判断是否非手动请求`!manual` 
    1. 是, 则
        1. 将`hasAutoRun.current`置为`true`
        1. 判断是否有传入`refreshDepsAction`方法
            1. 是, 则调用
            1. 否, 则调用`Fetch`的刷新请求方法`fetchInstance.refresh()` 

End.

可见, 此处的主要作用是在依赖更新时调用刷新方法. 但单从这一代码块而言, 这一刷新过程只会被调用一次, 甚至一次都不被调用——事实也的确如此, 整个插件代码里除初始化时`hasAutoRun.current`为`false`外, 没有其他将其赋值为`false`的地方了, 且不论是因为`ready`刷新引起了`run`方法的调用, 还是`refreshDeps`刷新引起了`refresh`方法的调用, 只要准备发起请求了, `hasAutoRun.current`就会被置为`true`. 

现在, 可以来浅浅理解一下`useAutoRunPlugin`的逻辑了.
### 整体逻辑 
1. 📝 第一优先级: `manual` 
作为最重要的参数, 只有`!manual`即非手动请求时, 此插件才有意义.
1. 📝 同第一优先级: `ready` 
它主要本身需要结合`manual`发挥作用的, 即`!manual && ready`才会发起自动请求.
但在`onBefore`事件中, 它也被用来独立判断, 即在手动请求的状态下, `ready`是唯一一个可以发挥作用的参数——当用户传入了`ready = false`时, 即使用户手动也无法发起请求.
1. 📝 第二优先级: `refreshDeps`
当依赖更新且还没有自动发起过请求时, 将自动执行刷新, 此操作只会执行0或1次. 
1. 📝 其他: defaultParams, refreshDepsAction 
功能简单明了, 没啥需要重复说的点.
## 缓存 & SWR · `useCachePlugin`
复杂一点的东西来了.
### 源码
```
import { useRef } from 'react';
import useCreation from '../../../useCreation';
import useUnmount from '../../../useUnmount';
import type { Plugin } from '../types';
import * as cache from '../utils/cache';
import type { CachedData } from '../utils/cache';
import * as cachePromise from '../utils/cachePromise';
import * as cacheSubscribe from '../utils/cacheSubscribe';

const useCachePlugin: Plugin<any, any[]> = (
  fetchInstance,
  {
    cacheKey,
    cacheTime = 5 * 60 * 1000,
    staleTime = 0,
    setCache: customSetCache,
    getCache: customGetCache,
  },
) => {
  const unSubscribeRef = useRef<() => void>();

  const currentPromiseRef = useRef<Promise<any>>();

  const _setCache = (key: string, cachedData: CachedData) => {
    if (customSetCache) {
      customSetCache(cachedData);
    } else {
      cache.setCache(key, cacheTime, cachedData);
    }
    cacheSubscribe.trigger(key, cachedData.data);
  };

  const _getCache = (key: string, params: any[] = []) => {
    if (customGetCache) {
      return customGetCache(params);
    }
    return cache.getCache(key);
  };

  useCreation(() => {
    if (!cacheKey) {
      return;
    }

    // get data from cache when init
    const cacheData = _getCache(cacheKey);
    if (cacheData && Object.hasOwnProperty.call(cacheData, 'data')) {
      fetchInstance.state.data = cacheData.data;
      fetchInstance.state.params = cacheData.params;
      if (staleTime === -1 || new Date().getTime() - cacheData.time <= staleTime) {
        fetchInstance.state.loading = false;
      }
    }

    // subscribe same cachekey update, trigger update
    unSubscribeRef.current = cacheSubscribe.subscribe(cacheKey, (data) => {
      fetchInstance.setState({ data });
    });
  }, []);

  useUnmount(() => {
    unSubscribeRef.current?.();
  });

  if (!cacheKey) {
    return {};
  }

  return {
    onBefore: (params) => {
      const cacheData = _getCache(cacheKey, params);

      if (!cacheData || !Object.hasOwnProperty.call(cacheData, 'data')) {
        return {};
      }

      // If the data is fresh, stop request
      if (staleTime === -1 || new Date().getTime() - cacheData.time <= staleTime) {
        return {
          loading: false,
          data: cacheData?.data,
          returnNow: true,
        };
      } else {
        // If the data is stale, return data, and request continue
        return {
          data: cacheData?.data,
        };
      }
    },
    onRequest: (service, args) => {
      let servicePromise = cachePromise.getCachePromise(cacheKey);

      // If has servicePromise, and is not trigger by self, then use it
      if (servicePromise && servicePromise !== currentPromiseRef.current) {
        return { servicePromise };
      }

      servicePromise = service(...args);
      currentPromiseRef.current = servicePromise;
      cachePromise.setCachePromise(cacheKey, servicePromise);
      return { servicePromise };
    },
    onSuccess: (data, params) => {
      if (cacheKey) {
        // cancel subscribe, avoid trgger self
        unSubscribeRef.current?.();
        _setCache(cacheKey, {
          data,
          params,
          time: new Date().getTime(),
        });
        // resubscribe
        unSubscribeRef.current = cacheSubscribe.subscribe(cacheKey, (d) => {
          fetchInstance.setState({ data: d });
        });
      }
    },
    onMutate: (data) => {
      if (cacheKey) {
        // cancel subscribe, avoid trgger self
        unSubscribeRef.current?.();
        _setCache(cacheKey, {
          data,
          params: fetchInstance.state.params,
          time: new Date().getTime(),
        });
        // resubscribe
        unSubscribeRef.current = cacheSubscribe.subscribe(cacheKey, (d) => {
          fetchInstance.setState({ data: d });
        });
      }
    },
  };
};

export default useCachePlugin;
```
这块的依赖方法就比较多了, 预知插件逻辑如何, 还且先看看使用了哪些封装过的方法.
`useCreation`在官方文档介绍中, 是`useMemo`或`useRef`的替代品, 相对前者, 可以保证memo的值不会被重新计算, 相对后者, 可以创建复杂常量且不易出现潜在的性能隐患.

另外, 则是三项与缓存相关的util.
```
import * as cache from '../utils/cache';
import * as cachePromise from '../utils/cachePromise';
import * as cacheSubscribe from '../utils/cacheSubscribe';
```
下面先来一个个看一看.
### 缓存 · `utils/cache` 
```
type Timer = ReturnType<typeof setTimeout>;
type CachedKey = string | number;

export interface CachedData<TData = any, TParams = any> {
  data: TData;
  params: TParams;
  time: number;
}
interface RecordData extends CachedData {
  timer: Timer | undefined;
}

const cache = new Map<CachedKey, RecordData>();

const setCache = (key: CachedKey, cacheTime: number, cachedData: CachedData) => {
  const currentCache = cache.get(key);
  if (currentCache?.timer) {
    clearTimeout(currentCache.timer);
  }

  let timer: Timer | undefined = undefined;

  if (cacheTime > -1) {
    // if cache out, clear it
    timer = setTimeout(() => {
      cache.delete(key);
    }, cacheTime);
  }

  cache.set(key, {
    ...cachedData,
    timer,
  });
};

const getCache = (key: CachedKey) => {
  return cache.get(key);
};

const clearCache = (key?: string | string[]) => {
  if (key) {
    const cacheKeys = Array.isArray(key) ? key : [key];
    cacheKeys.forEach((cacheKey) => cache.delete(cacheKey));
  } else {
    cache.clear();
  }
};

export { getCache, setCache, clearCache };
```
这是一个比较核心的处理`cache`的模块. 其中包含有`cache`缓存本身、`getCache`获取缓存、`setCache`设置缓存、`clearCache`清空缓存三个配套方法. 且它们都是全局变量与全局方法, 所有请求之间是共用的, 因此只要`cacheKey`一致就可以共享缓存.

对插件而言, 则只透出了后面三个方法, `cache`变量本身是不能也没必要直接访问的. 我们只需简单了解`cache`的类型和几个方法的大致思路即可.

`cache`是什么: `Map`(带原始插入顺序的键值对). 函数签名: `const cache = new Map<CachedKey, RecordData>(); `

Map的属性和方法 ⬇️
```
// 属性
get Map[@@species]
Map.prototype[@@toStringTag]
Map.prototype.size
// 方法
Map.prototype[@@iterator]()
Map.prototype.clear()
Map.prototype.delete()
Map.prototype.entries()
Map.prototype.forEach()
Map.prototype.get()
Map.prototype.has()
Map.prototype.keys()
Map.prototype.set()
Map.prototype.values() 
```
随后来看`cache`的三个方法. 
`getCache`最为简略: 直接使用了`Map.get()`方法. 
```
const getCache: (key: CachedKey) => RecordData | undefined
```
`setCache`看起来比较复杂, 但其主体功能的实现也是使用了`Map.set()`方法. 而在此之前, 还有两步操作.
1. 清除传入`key`的计时器
1. 如果有数据回收时间`cacheTime`, 则给它传入一个新的定时清除方法
1. 最后, 重新set
```
const setCache: (key: CachedKey, cacheTime: number, cachedData: CachedData) => void 
```
`clearCache`则分为清除特定`key`的缓存和清除全部缓存, 前者使用`Map.delete()`, 后者则`Map.clear()`.
```
const clearCache: (key?: string | string[]) => void 
```
弄清楚了`cache`, 就懂了很重要的一小半了. 
### 请求缓存 · `utils/cachePromise`
```
type CachedKey = string | number;
const cachePromise = new Map<CachedKey, Promise<any>>();

const getCachePromise = (cacheKey: CachedKey) => {
  return cachePromise.get(cacheKey);
};

const setCachePromise = (cacheKey: CachedKey, promise: Promise<any>) => {
  // Should cache the same promise, cannot be promise.finally
  // Because the promise.finally will change the reference of the promise
  cachePromise.set(cacheKey, promise);

  // no use promise.finally for compatibility
  promise
    .then((res) => {
      cachePromise.delete(cacheKey);
      return res;
    })
    .catch(() => {
      cachePromise.delete(cacheKey);
    });
};

export { getCachePromise, setCachePromise };
```
其中有`getCachePromise`和`setCachePromise`两个方法, 且也有一个与`cache`类似的`Map`类型全局变量`cachePromise`.
- `getCachePromise`: 使用`Map.get()`方法返回特定`cacheKey`对应的值. 
- `setCachePromise`: 使用`Map.set()`方法设定传入的`(cacheKey, promise)`键值对, 并给`promise`添加`then`和`catch`方法, 触发时删除这一键值对. 

这里可以先粗略判断一下——只有通过`setCachePromise`发起过的未结束的请求可以从`getCachePromise`中获取. 

具体作用等到插件中之后再做分析.
### 预定缓存 · `utils/cacheSubscribe `
```
type Listener = (data: any) => void;
const listeners: Record<string, Listener[]> = {};

const trigger = (key: string, data: any) => {
  if (listeners[key]) {
    listeners[key].forEach((item) => item(data));
  }
};

const subscribe = (key: string, listener: Listener) => {
  if (!listeners[key]) {
    listeners[key] = [];
  }
  listeners[key].push(listener);

  return function unsubscribe() {
    const index = listeners[key].indexOf(listener);
    listeners[key].splice(index, 1);
  };
};

export { trigger, subscribe };
```
这里的全局变量`listeners`就与前文两个的`Map`类型有所不同了:
```
const listeners: Record<string, Listener[]> = {}; 
```
但基本结构仍然类似, 返回了两个全局方法:
- `trigger`: 根据传入的键`key`, 以`data`为参数, 遍历调用`listeners[key]`中的所有内容
- `subscribe`: 将传入的`listener`增量存入指定的`key`,  并返回其删除方法.

同理, 回到插件本身.
