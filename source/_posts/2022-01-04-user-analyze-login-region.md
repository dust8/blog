---
title: 用户分析之登录地域
date: 2022-01-04 05:15:31
tags:
  - 分析
  - python
  - echarts
  - django
---


最近要分析下用户登录地域.了解到`ECharts`可以实现,可以看官方的例子
[香港18区人口密度](https://echarts.apache.org/examples/zh/editor.html?c=map-HK).


## 数据
### 用户数据
在登陆的时候用`GPS`定位,或者`ip`地址获取到地理位置,例如省的名称, 记录到数据库. 然后把数据按省来分组用户去重就可以得到echarts地图需要的数据了.
```python
qs = (UserLoginRecord.objects.filter(record_time__gte=start_date)
          .filter(record_time__lt=end_date)
          .values('province')
          .annotate(unique_user=Count('user_id', distinct=True))
          .order_by('unique_user'))
# 结果    
[
    { province: '北京', unique_user: 20 },
    { province: '台湾', unique_user: 1 },
]

# 转换为
[
    { name: '北京', value: 20 },
    { name: '台湾', value: 1 },
]
```

### 地图数据
2种方式,用别人处理好的, 或者自己画(看参考链接).    
v5 移除了内置的 geoJSON（原先在 echarts/map 文件夹下）。这些 geoJSON 文件本就一直来源于第三方。如果使用者仍然需要他们，可以去从老版本中得到，或者自己寻找更合适的数据然后通过 registerMap 接口注册到 ECharts 中。   
这里我从第三方的cdn里面找到老版本的json文件.
```
# utils/china.js, 把json地图文件内容变成变量好引入,就不用发请求了
export const chinaJson = { "type": "FeatureCollection", ...}
```
```
# 导入地图数据并注册
import { chinaJson } from '/@/utils/china'
echarts.registerMap('china', chinaJson)
```

数据处理好了,其他的看例子就好了

## 参考链接
- [去除内置的 geoJSON](https://echarts.apache.org/handbook/zh/basics/release-note/v5-upgrade-guide/#%E5%8E%BB%E9%99%A4%E5%86%85%E7%BD%AE%E7%9A%84-geojson)
- [香港18区人口密度](https://echarts.apache.org/examples/zh/editor.html?c=map-HK)
- [echarts搞定各种地图（想怎么画就怎么画）](https://www.jianshu.com/p/7337c2f56876)