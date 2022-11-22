---
title: vue之组合式函数
date: 2022-11-22 06:15:31
tags:
  - vue
---

## 起因
前端有个树形结构需要显示, 层级比较多, 达到7层, 还是需要自适应大小. 现有的组件要不层级不够, 例如像省市县这样的级联, 只有3层, 如果做成7层, 在移动端显示效果不好.
在网上发现`yangjingyu/vs-tree`比较符合要求, 上面用面包屑显示层级, 下面只显示最后一层的列表. 不过在vue3上有些问题. 刚好最近接触了`hooks`, 还了解了`Headless UI`. 把逻辑和`ui`进行分离, 以前看过(用积木理论设计一个灵活好用的Carousel走马灯组件), 也想过, 只是不知道这样一个统一的名称. 
目前常见的组件通过`slot`来定制化还是不够灵活. 
灵活性: 
```
手写逻辑和ui > hooks + 手写ui > ui框架组件
```
当然灵活性越大, 工作量也越大.


## 实例
输入: `data` 是一个扁平的树, id为"_"的是根节点.
```json
{
	"_": {
		"id": "_",
		"name": "_",
		"children": ["1", "2"]
	},
	"1": {
		"id": "1",
		"name": "1",
		"children": []
	},
	"2": {
		"id": "2",
		"name": "2",
		"children": []
	}
}
```
输出: 导出4个变量, `breadcrumbNodes`,`breadcrumbNodeClick`,是上面面包屑的节点列表, 和点击函数, `currentNodes`, `currentNodeClick` 是下面的节点列表和点击函数.

```javascript
# useTree.js
import { ref } from "vue";

const useTree = (data) => {
  const nodesMap = ref(data);
  const breadcrumbNodes = ref([]);
  const currentNodes = ref([]);

  const updateCurrentNodes = (node) => {
    const nodes = [];
    nodesMap.value[node.id].children.forEach((element) => {
      nodes.push(nodesMap.value[element]);
    });
    currentNodes.value = nodes;
  };
  const breadcrumbNodeClick = (index) => {
    // 更新头
    breadcrumbNodes.value = breadcrumbNodes.value.slice(0, index + 1);

    // 更新列表
    const node = breadcrumbNodes.value[index];
    updateCurrentNodes(node);
  };

  const currentNodeClick = (index) => {
    const node = currentNodes.value[index];
    if (!node.children || !node.children.length) {
      console.log("no children");
      return;
    }

    // 更新头
    breadcrumbNodes.value.push(node);

    // 更新列表
    updateCurrentNodes(node);
  };

  breadcrumbNodes.value.push(nodesMap.value["_"]);
  updateCurrentNodes(nodesMap.value["_"]);

  return {
    breadcrumbNodes,
    currentNodes,
    breadcrumbNodeClick,
    currentNodeClick,
  };
};
export default useTree;
```
使用
```javascript
<script setup>
import useTree from './useTree'

const data2 = {}
const { breadcrumbNodes, currentNodes, breadcrumbNodeClick, currentNodeClick } = useTree(data2)

const listclick2 = (index) => {
  const node = currentNodes.value[index]
  console.log('listclick2', index, node.name)
  currentNodeClick(index)
}

</script>
<template>
  <div>
    <div class="tree-box">
      <div class="tree-breadcrumb">
        <span v-for="(item, index) in breadcrumbNodes" :key="item.id" @click="breadcrumbNodeClick(index)">
          <span class="tree-breadcrumb-start" v-if="index == 0">@</span>
          <span class="tree-breadcrumb-link">{{ item.name }}</span>
          <span class="tree-breadcrumb-separator" v-if="index != breadcrumbNodes.length - 1">/</span>
        </span>
      </div>

      <ul class="tree-list">
        <li v-for="(item, index) in currentNodes" :key="item.id" @click="listclick2(index)">
          {{ item.name }}
        </li>
      </ul>
    </div>
  </div>
</template>
```

## 参考链接
- [组合式函数](https://cn.vuejs.org/guide/reusability/composables.html)
- [这样封装列表 hooks,一天可以开发 20 个页面](https://juejin.cn/post/7165467345648320520)
- [用积木理论设计一个灵活好用的Carousel走马灯组件](https://juejin.cn/post/7051370356585529357)
- [yangjingyu/vs-tree](https://github.com/yangjingyu/vs-tree)
- [我被骂了，但我学会了如何构造高性能的树状结构](https://juejin.cn/post/7142649750402121742)
- [reactjs - avoid-deeply-nested-state](https://beta.reactjs.org/learn/choosing-the-state-structure#avoid-deeply-nested-state)
- [vue - 树状视图](https://cn.vuejs.org/examples/#tree)
