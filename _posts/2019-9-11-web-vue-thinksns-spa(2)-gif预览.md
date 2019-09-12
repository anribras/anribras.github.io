---
layout: post
title:
modified:
categories: Tech
tags: [vue, thinksns, spa, web]
comments: true
---

<!-- TOC -->

- [v-gif-play 使用场景](#v-gif-play-使用场景)
- [gifPlay.js](#gifplayjs)
    - [IntersectionObserver](#intersectionobserver)
- [util/prototype.js](#utilprototypejs)
- [bus.js](#busjs)
- [gif 数据出处 v-async-image](#gif-数据出处-v-async-image)
    - [getFirstFrameOfGif.js](#getfirstframeofgifjs)

<!-- /TOC -->

通过自定义指令实现 gif 在自动播放(1 级页面 feed 列表里)

## v-gif-play 使用场景

```js
       <ul v-gif-play class="feed-list">
          <li
            v-for="(feed, index) in pinned"
            v-if="feed.id"
            :key="`pinned-feed-${feedType}-${feed.id}-${index}`"
            :data-feed-id="feed.id"
          >
            <FeedCard :feed="feed" :pinned="true" />
          </li>
          <li
            v-for="(card, index) in feeds"
            :key="`feed-${feedType}-${card.id}-${index}`"
            :data-feed-id="card.id"
          >
            <FeedCard v-if="card.user_id" :feed="card" />
            <FeedAdCard v-if="card.space_id" :ad="card" />
          </li>
        </ul>
```

## gifPlay.js

这个是用来实现`v-gif-play`指令的.具体是在 mixin.js 里引入:

```js
import directives from '@/directives;
export default {
  data () {
    return {
      ...
    }
  },
  directives,
```

directive 的只要实现了响应的钩子即可,如 gifPlay.js:

```js
export default {
  bind: onBind,
  inserted: onInserted,
  update: onInserted,
  unbind: onUnbind
};
```

最终是在 OnBind 里添加了 scroll 的回调，也就是滚动时，将查找 feedId,
并且将 bus.js 里的数据结构 elList 赋值:

```js
const onScroll = _.throttle(() => {
  //添加playing-feed标记
  const playEl = document.querySelector(".playing-feed") || {};

  const { feedId } = playEl.dataset || {};
  if (feedId && feedId !== gif.feedId) {
    //更新feedId
    gif.feedId = feedId;
    //所有包含data-gif-duration属性，但是不含need-pay的class
    const playingEls = playEl.querySelectorAll(
      "[data-gif-duration]:not(.need-pay)"
    );
    //更新elList
    gif.elList = playingEls;
  }
}, 200);
```

feedId 变化时，elList 更新.

bus.js 里的 gif 对象数据跟随 feedId 而变化. 需求应该就是就是要实现随着滚动，自动检测到 feed 里的 gif 并播放

一旦 feedId 切换，上 1 篇的全部停止，下 1 篇的自动播放，大概这个意思.

feedId 切换取决于`.playing-feed`何时添加.

`OnInserted`里，将指令 bind 的元素的子元素，用`IntersectionObserver`来监控.

`playing-feed`就是在这个里面 add 和 remove 的.

### IntersectionObserver

所有的改变都是通过它观察的，之前自己的做法是监控`documentElement.scrollTop`的值.

用了 1 个`IntersectionObserver`,ios 还需要做 polyfill:

```js
const observer = new IntersectionObserver(entries => {
  entries.forEach(entry => {
    if (entry.isIntersecting && entry.intersectionRatio > 0.75) {
      if (entry.target.querySelectorAll(".gif:not(.need-pay)").length) {
        entry.target.classList.add("playing-feed");
      }
    } else {
      entry.target.classList.remove("playing-feed");
    }
  });
}, options);
```

## util/prototype.js

给 div tag 添加了 play 和 stop 函数,专门用来播放 gif 和停止 gif;

```
并在父级元素上添加`playing`.
```

dataset 的数据怎么来的看最后.

git.index++是在第 1 张 gifDuration 到时候后才发生，此时才能触发下 1 个 play,
也就是 gif 并不是同一时间一起播放，而是按顺序来的?

```js
// 给 Div Element 增加 播放和停止的功能（用于播放GIF图像）
if (!HTMLDivElement.prototype.play) {
  HTMLDivElement.prototype.play = function() {
    if (!this.dataset.gifBlobUrl) {
      return console.warn("节点不含有 GIF 信息, 无法播放", this); // eslint-disable-line no-console
    }
    this.parentElement.classList.add("playing");
    this.style.backgroundImage = `url(${this.dataset.gifBlobUrl})`;

    bus.$data.gif.timer = setTimeout(() => {
      this.stop();
      //顺序play gif?
      bus.$data.gif.index++;
    }, +this.dataset.gifDuration);
  };
  HTMLDivElement.prototype.stop = function() {
    if (!this.dataset.gifBlobUrl) {
      return console.warn("节点不含有 GIF 信息, 无法停止", this); // eslint-disable-line no-console
    }
    this.parentElement.classList.remove("playing");
    this.style.backgroundImage = `url(${this.dataset.gifFirstFrame})`;
  };
}
```

## bus.js

另外控制数据来自 bus.js:

```js
  data () {
    return {
      gif: {
        feedId: null, //id改变，stop index=0
        elList: [],
        index: null, //当前播放的index
        timer: null, //播放期间运行，到时则停止
      },
    }
```

设置 timer 增加 list index,index 变化被 watch.不断的调用 play,播放所有 gif 图片

```js
  watch: {
    'gif.feedId' (val, oldId) {
      document.querySelectorAll('.playing > div').forEach(el => el.stop())
      clearTimeout(this.gif.timer)
      this.gif.index = null
      this.gif.timer = null
      this.$nextTick(() => {
        this.gif.index = 0 //这里触发第1张图的play()
      })
    },
    'gif.index' (index) {
      if (index === null) return
      if (index >= this.gif.elList.length) {
        this.gif.index = 0
        return
      }
      this.gif.elList[index].play()
    },
  },
}
```

gif 数据来自 this.dataset:

## gif 数据出处 v-async-image

还有最后 1 个关键：gif 的数据来自哪里?

懒加载:

```sh
1 图片是否在视口
2 开始访问url,(xx/file?json=1),避免重定向问题
3  通过返回的url后缀`.gif`判断是gif图片,
4 api拉取blob数据，并存放在el.dataset里.
5 Promise.all 并发2个动作:
    handleFile:
    getFirsetFrameOfGif:
```

handleFile:把 gifinfo resolve 出去

```js
const handleFile = blob => {
  return new Promise(resolve => {
    const reader = new FileReader();
    reader.onload = function(event) {
      const arrayBuffer = reader.result;
      const gifInfo = window.gify.getInfo(arrayBuffer);
      resolve(gifInfo);
    };
    reader.readAsArrayBuffer(blob);
  });
};
```

### getFirstFrameOfGif.js

getFirsetFrameOfGif: 把第 1 帧的 blob 拿到，为了预览方便.

```js
const URL = window.URL || window.webkitURL;

function dataURLtoBlob(dataURL) {
  const arr = dataURL.split(",");
  const mime = arr[0].match(/:(.*?);/)[1];
  const bstr = atob(arr[1]);
  let n = bstr.length;
  const u8arr = new Uint8Array(n);
  while (n--) {
    u8arr[n] = bstr.charCodeAt(n);
  }
  return new Blob([u8arr], { type: mime });
}

export default (file, type = "dataURL") => {
  return new Promise(resolve => {
    const image = new Image();

    image.onload = () => {
      const width = image.width;
      const height = image.height;

      const canvas = document.createElement("canvas");

      canvas.width = width;
      canvas.height = height;
      // 绘制图片帧（第一帧）
      canvas.getContext("2d").drawImage(image, 0, 0, width, height);
      const dataURL = canvas.toDataURL("image/jpeg", 0.5);
      switch (type) {
        case "dataURL":
          return resolve(dataURL);
        case "blob":
          return resolve(dataURLtoBlob(dataURL));
      }
    };

    image.src = URL.createObjectURL(file);
  });
};
```
