---
title: vue之videojs
date: 2022-11-24 06:15:31
tags:
  - vue
  - videojs
---


## 起因
同时在`vue3`中播放多个视频, 还能翻页.

## 问题
### 问题1.多个播放器
解决: 按照`videojs`官方文档的`vue`集成教程熟悉下, 在封装成组件, 一个组件有一个播放器, 循环就支持多个. 通过`ref`来自动生成`id`.
```javascript
<template>
  <div>
    <a href="#" class="group" v-for="item in items">
        <div class="aspect-w-1 aspect-h-1 w-full overflow-hidden">
          <video-player :options="item.videoOptions" />
        </div>
        <h3 class="mt-1 text-lg font-medium text-gray-900">{{ item.name }}</h3>
    </a>
  </div>
</template>

<script setup>
import { ref } from "vue";
import VideoPlayer from '@/components/video-player.vue'

const items = [
    {
        name:"湖南卫视",
        videoOptions: {
          autoplay: true,
          muted: true,
          controls: true,
          sources: [
            {
              src: 'http://219.151.31.38/liveplay-kk.rtxapp.com/live/program/live/hnwshd/4000000/mnf.m3u8',
              type: 'application/x-mpegURL'
            }
          ]
        }
    }]
</script>
```


### 问题2.翻页播放
翻页后`videojs`生成的id未变, 视频还是更改前的. vue组件传进去的参数已经发生改变, 但`videojs`实例`player`的播放地址没有改变. 看官方文档有个`player.src()`方法可以改变播放地址. 却出现一直请求`.m3u8`文件, 未请求`.ts`视频文件.

解决:
1.在组件内通过`watch`来监听参数变化, 有变动通过`player.src()`来设置新的播放地址.
2.由于`vue3` 默认使用代理来声明变量, `player`的`src()`就出现了问题, 可以把`player`声明为普通变量,  可以参考`video.js/issues/7418`.
```javascript
// video-player.vue
<script>
import videojs from 'video.js';
import "video.js/dist/video-js.css";
import zh from "video.js/dist/lang/zh-CN.json";
import { onBeforeUnmount, onMounted, watch, ref } from 'vue';

videojs.addLanguage('zh-CN', zh);

export default {
  name: 'LjVideoPlayer',
  props: {
    options: {
      type: Object,
      default() {
        return {};
      }
    }
  },
  setup(props) {
    let videoPlayer = ref(null)
    let player =null;
  
    const onChange = () => {
      if (player) {
        player.src(props.options.sources)
      }
    }

    const dispose = () => {
      if (player) {
        player.dispose()
      }
    }
    onMounted(() => {
      player = videojs(videoPlayer.value, props.options, () => {
        // player.log('onPlayerReady', this);
      });
    })

    onBeforeUnmount(() => {
      if (player) {
        player.dispose();
      }
    })

    watch(props, () => {
      onChange()
    })

    return { dispose, videoPlayer,}
  },
}
</script>

<template>
  <div>
    <video ref="videoPlayer" class="video-js vjs-fluid vjs-fill" webkit-playsinline="true" playsinline="true"></video>
  </div>
</template>

<style scoped>
video {
  height: 100%;
  width: 100%;
  min-width: 100px;
  min-height: 100px;
  display: block;
}
</style>
```

## 问题3. vue3没有$refs
`videojs`初始化需要元素, 通过`ref`来指定后, 在取值时需要声明和导出.
```javascript
<script>
export default {
    setup(){
        // 声明
        let videoPlayer = ref(null)
        let player =null
        
        onMounted(() => {
          player = videojs(videoPlayer.value, props.options, () => {
            // player.log('onPlayerReady', this);
          });
        })
        // 导出
        return {videoPlayer,}
    }
}
</script>

<template>
  <div>
    <video ref="videoPlayer"></video>
  </div>
</template>
```

## 问题4. 中文语言
官方文档只说了设置语言, 却没说如何引入中文语言包. 没有说明`vue`如何引入中文包, 在`https://github.com/videojs/video.js/issues/7986`可以知道.
```javascript
// 引入中文
import zh from "video.js/dist/lang/zh-CN.json";
```
```javascript
// 初始化的时候设置中文
{
    autoplay: true,
    muted: true,
    controls: true,
    language: 'zh-CN',
    sources: [
      {
        src: src,
        type: 'application/x-mpegURL'
      }
    ]
  }
```

## 问题5. 重复使用一个弹出框来播放不同视频
一种是弹出生成播放实例, 关闭后销毁. 一种是重新设置视频地址.
下面说说不销毁的, 如果不销毁, 那么需要使用暂停, 会一直请求视频地址(m3u8)来缓存. 可以设置视频地址为空字符串, 来禁止一直请求.
```javascript
player.pause()
player.src('')
player.load()
```

## 参考链接
- [Cant not play when use player.src to change media's url](https://github.com/videojs/video.js/issues/7418)
- [friendly-carson-2rmnnh](https://codesandbox.io/s/friendly-carson-2rmnnh?file=/src/App.vue)
- [i failed to set language zh-CN in react project](https://github.com/videojs/video.js/issues/7986)