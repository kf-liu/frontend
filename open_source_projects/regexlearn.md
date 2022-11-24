# [regexlearn.com](regexlearn.com) 游戏化方式学正则

regexlearn是一个学习正则的开源网站，用户可以通过类似闯关的方式学习正则表达式，亲测包教包会！

先来简单看一看它长什么样子 ⬇️

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/057a7e97c9d4451d9f9a9bb2632f40b7~tplv-k3u1fbpfcp-watermark.image?)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3867f66a0a6d4636945cdaa73dd19123~tplv-k3u1fbpfcp-watermark.image?)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/84645b443c754b398e5407451e6139f5~tplv-k3u1fbpfcp-watermark.image?)

目前站内的核心功能是学习正则表达式，它会通过题目和一句话教学引导你使用正则表达式匹配指定的词句，难度梯度把握的很好。

更重要的是，这个站，虽然很简单，但真的很好看😍 好喜欢😍 所以咱们先来看看它为什么给人感觉还不错？🤨

### UE Design

##### 黑绿配色

它的背景的黑色不是`#000`的纯黑，而是灰灰的`#282c34`，前景色则是不同透明度的`rgb(95 245 155)`，注意，这个绿不是普通的绿，而是~~王维诗里的绿~~带点赛博朋克的荧光却不死亡绿。

这两个颜色可谓双双深得程序员本猿的心❤️

##### 交互动效

各种炫酷CSS动效见多了，但它仍然是属于好看的那一挂。

![hover.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b14b6450935e4e148fb8846f8de48b1f~tplv-k3u1fbpfcp-watermark.image?)

仔细看看会发现其实也很水，就是一个条纹格的背景图hover后放大移出后缩小。可以看看CSS求证求证。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/19a5483f44544ccbbe66cf8c0ee224a5~tplv-k3u1fbpfcp-watermark.image?)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f1dad3d2e29f48dc88ac00dbcb3b97e3~tplv-k3u1fbpfcp-watermark.image?)

果不其然，class`.hover\:bg-\[length\:125\%_125\%\]`的`hover`事件作用就是放大背景尺寸：
```css
.hover\:bg-\[length\:125\%_125\%\]:hover {
    background-size: 125% 125%;
}
```

另外，配上一丢丢的动效———？居然没有css文件，嗯，less也没有，sass也没有，fine。

原来是配上一丢丢的魔法：
- .hover\:bg-\[length\:125\%_125\%\]
- .duration-300
- .transition-all
- .shadow-sm, .shadow-xl

———上面这些class的样式，实则都是由[`@tailwind`](https://www.tailwindcss.cn/)处理。

> Tailwind CSS 是一个功能类优先的 CSS 框架，它集成了诸如 `flex`, `pt-4`, `text-center` 和 `rotate-90` 这样的的类，它们能直接在脚本标记语言中组合起来，构建出任何设计。

在开发中只需要在`src/styles/globals.css`下添加以下几行代码，即可使用**语义化类名**（比如上面那四个class）来创造css。
```tsx
@tailwind base;
@tailwind components;
@tailwind utilities;
```

小小的交互挖出大大的宝藏！逃离手写css的知识又+1啦～

### 多语言

这其实是个很常见的操作了，许多大型网站都是N语言国际化的。但也正是因为它们往往都是大网站，所以想要看清楚它的多语言处理逻辑反而没那么清晰。这个项目体量小、语言多，正好合适～

先来瞅一眼`src/pages`下的目录结构。
```
src/pages
|____404.tsx
|_____document.tsx
|_____app.tsx
|____[lang]
| |____index.tsx
| |____learn
| | |____index.tsx
| | |____[lesson].tsx
| |____playground.tsx
| |____cheatsheet.tsx
```

作为一个`Next.js`项目，它的目录结构就是路由结构。中文首页`https://regexlearn.com/zh-cn`就对应着`[lang]/index.tsx`文件。

回到`src`下的一级文件。
```
src
|____migration.ts
|____context
|____types.d.ts
|____utils
|____styles
|____components
|____shortcuts.ts
|____data
|____pages
|____localization
```

除了`pages`，还有`localization`文件夹，下面是各语言的配置文件。以`zh-cn`为例，各个语言的文件结构都一样。
```
src/localization
|____zh-cn
| |____lessons.json
| |____general.json
| |____landing.json
| |____index.js
| |____learn.json
| |____cheatsheet.json
| |____lessons
| | |____regexForSeo.json
```

其中`index`是个入口文件。具体配置的`JSON`文件，可以浅看一眼`lessons.json`，都一样。
```json
{
  "lessons.regex101.title": "Regex 101 - ZH-CN",
  "lessons.regex101.description": "您可以在本教程中学习 Regex 的基础知识。 ",

  "lessons.regexForSeo.title": "Regex for SEO - ZH-CN",
  "lessons.regexForSeo.description": "在本教程中，您可以了解如何以及在何处使用 Regex 进行 SEO。"
}
```

在使用时，只需要上面文件的将`key`：`lessons.regex101.title`作为`id`传入`FormattedMessage`组件即可 ⬇️
```tsx
import { FormattedMessage } from 'react-intl';
// ...
<FormattedMessage id="lessons.regex101.title" />
// ...
```

`FormattedMessage`组件则来自国际化常用的[`react-intl`](https://react.i18next.com/)包，完全不需要你自己操心。如此看来，这一套操作下来国际化还是很简单方便的，MARK☑️。

---

小小的开源项目，简单的动效，简单的语言，还是有不少宝藏可挖挖的！

今天就看到这里，赶紧`✨Star`明天继续！
