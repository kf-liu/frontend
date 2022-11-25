# [jsoncrack.com](jsoncrack.com) — json可视化工具

![https://github.com/AykutSarac/jsoncrack.com/raw/main/public/assets/jsoncrack-screenshot.webp](https://github.com/AykutSarac/jsoncrack.com/raw/main/public/assets/jsoncrack-screenshot.webp)

这个项目的核心功能非常清晰———JSON的可视化。虽然整体界面很像一个编辑器，左侧有很多按钮，但它们并非大型的功能模块，而是一些快捷的配置项，如导入文件、树状图的旋转等。因此（我一开始觉得）专注看看它的核心功能即可。

作为前端常用的配置文件格式，JSON结构本身是清晰好用的，但也架不住链表太长、深度太大…… 有时想把握一个JSON文件的核心结构，属实有点费眼睛。jsoncrack.com就属于这种，平时觉得没啥用，JSON文件逼疯你后疯狂全网寻找的神器。

首先，还是想来整体看看它的项目结构。

```shell
.
├── CODE_OF_CONDUCT.md
├── CONTRIBUTING.md
├── Dockerfile
├── LICENSE
├── README.md
├── default.conf
├── jest.config.ts
├── jest.setup.ts
├── next-env.d.ts
├── next.config.js
├── package.json
├── tsconfig.json
├── yarn.lock
├── public
│   └── assets
└── src
    ├── components
    │   ├── Button
    │   ├── CarbonAds
    │   ├── CustomNode
    │   ├── ErrorContainer
    │   ├── GoogleAnalytics
    │   ├── Graph
    │   ├── Input
    │   ├── Loading
    │   ├── Modal
    │   ├── MonacoEditor
    │   ├── Producthunt
    │   ├── SearchInput
    │   ├── SeoTags
    │   ├── Sidebar
    │   ├── Sponsors
    │   ├── SupportButton
    │   ├── Toggle
    │   ├── Tooltip
    │   └── __tests__
    ├── constants
    ├── containers
    │   ├── Editor
    │   │   ├── JsonEditor
    │   │   └── LiveEditor
    │   ├── Home
    │   └── Modals
    │       ├── ClearModal
    │       ├── DownloadModal
    │       ├── GoalsModal
    │       ├── ImportModal
    │       ├── NodeModal
    │       ├── SettingsModal
    │       └── ShareModal
    ├── hooks
    ├── pages
    │   ├── Editor
    │   ├── Embed
    │   └── Widget
    ├── store
    ├── typings
    └── utils
```

### iframe套娃

从这个目录，可以看出它所使用的框架也是`Next.js`，而我们从首页进入的编辑器是`src/pages/Editor`。扒拉扒拉另外两个页面，我们发现`src/pages/Embed`下还有一个这样的页面 ⬇️


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9719dc8b43014d57b759f8c45fa83997~tplv-k3u1fbpfcp-watermark.image?)

是用iframe嵌入了一个演示demo用的[codepen](https://codepen.io/)页面（[地址](https://codepen.io/AykutSarac/embed/PoawZYo?default-tab=html%2Cresult)）

从源码反推，我才找到从首页到`/embed`页面的方法：首页 ——— 正中间 “GO TO EDITOR” ——— 左侧工具栏 “Share” ——— “Learn How to Embed”。（无聊的游戏🥱 但很精神👴）

更有趣的是另外一个页面：`scr/page/Widget`。

~~因为它打不开。~~

因为你已经见过它了，只是你还不知道！

就在你已经划过了的那个首页！

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/54c1229bf477457a8eefd6e411e5469b~tplv-k3u1fbpfcp-watermark.image?)

它是一个被iframe嵌入的迷你demo！

其实你可以先尝试在浏览器输入它的地址 https://jsoncrack.com/widget ，然后发现我也没有说错，因为确实打不开———它会重定向到首页———且不是因为“走错了”、“404”。

第一，这个地址真的存在。

第二，事实证明，走错了长这样：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/93874cd71c784229841342d6c27fe8c8~tplv-k3u1fbpfcp-watermark.image?)

看看代码：
```jsx
function inIframe() {
  try {
    return window.self !== window.top;
  } catch (e) {
    return true;
  }
}
// ...
if (!inIframe()) push("/");
```

果不其然哈。

判断是否被嵌入`iframe`的方法 +1。

有趣，俩页面，一个iframe嵌别人，一个被iframe嵌哈，正常人确实是找不着哈。

### css in js ?!

还是回到核心功能。

一进来就给孩子看傻眼了：
```tsx
export const StyledEditorWrapper = styled.div`
  width: 100%;
  height: 100%;
  overflow: hidden;
`;
```
这是个什么用法？（自问自答）这是一个叫[`styled-components`](https://styled-components.com/)的库。这是一个最小demo：
```tsx
import styled from 'styled-components'
// ...
const Button = styled.button``
```

其主要操作呢，是在`styled.button`后的\`\`中，添加所需要的css样式，就可以返回好看的组件。除了`width`、`height`这种普通的样式外，还支持css选择器，同时，也可以用`${}`内的函数返回一些动态的样式 ⬇️

```tsx
import styled, { css } from 'styled-components'

const Button = styled.button`
  background: transparent;
  border-radius: 3px;
  border: 2px solid palevioletred;
  color: palevioletred;
  margin: 0 1em;
  padding: 0.25em 1em;

  ${props =>
    props.primary &&
    css`
      background: palevioletred;
      color: white;
    `};
`
```

又是一个有趣的css打开方式。和昨天**能不写就不写的语义化css**完全相反，这是一个典型的`css in js`的写法。这一思想可以很好的避免传统css造成的样式污染问题，也让样式的维护和数据一样方便集中，但也造成了较大的“冗余”，实际上如今这样的打开方式似乎不是主流。

### 漂亮的模块封装 📦

```tsx
const EditorPage: React.FC = () => {
  return (
    <StyledEditorWrapper>
      <Head>
        <title>Editor | JSON Crack</title>
        <meta
          name="description"
          content="View your JSON data in graphs instantly."
        />
      </Head>
      <StyledPageWrapper>
        <Sidebar />
        <StyledEditorWrapper>
          <Panes />
        </StyledEditorWrapper>
      </StyledPageWrapper>
    </StyledEditorWrapper>
  );
};

export default EditorPage;
```

作为核心页面的主要代码，真是让人赏心悦目。回忆一下这个页面的结构 ⬇️

再看看上面那段代码 ⬆️

再继续往下读 ⬇️

![https://github.com/AykutSarac/jsoncrack.com/raw/main/public/assets/jsoncrack-screenshot.webp](https://github.com/AykutSarac/jsoncrack.com/raw/main/public/assets/jsoncrack-screenshot.webp)

是不是让你冷静冷静现场不用修BUG好像都蠢蠢欲动立马想好从哪里开刀了？（bushi

没错是的，就是这么漂亮，好想拉一刀。

这样的项目得加多少功能才能改成💩呢？

（开个玩笑）

---

小小的开源项目，简单的页面，简单的语言，还是有不少宝藏可挖挖的！

今天就看到这里，赶紧`✨Star`明天继续！
