---
title: django ckeditor 插件上传视频
date: 2021-11-16 05:15:31
tags:
  - django
  - django-admin
  - ckeditor
  - video
---

[django-ckeditor/django-ckeditor](https://github.com/django-ckeditor/django-ckeditor)是django后台的富文本插件,没有默认带视频编辑功能,而我要用到,找了下资料实现了.

## 原理
1.下载ckeditor的插件    
2.django里面配置启用插件    

这里面资料少的是如何把他们合理的结合在一起.


## 示例
### 下载插件
`django-ckeditor` 使用的是 `ckeditor4`, 去它的 `github` 可以看到 `Features` 里面说有超过 500 个插件,并给出了插件地址 [https://ckeditor.com/cke4/addons/plugins/all](https://ckeditor.com/cke4/addons/plugins/all).
在里面搜索 `video` 可以看到相关的一些插件. 这里我选用了 [html5video](https://ckeditor.com/cke4/addon/html5video) 这个插件, 把它下载并解压出来.

### 启用插件
`django-ckeditor` 的文档里面有说明
```
CKEDITOR_CONFIGS = {
    'default': {
        'extraPlugins': ','.join([
            ...
            'html5video', 
        ]),
    }
}
```

### 结合
这里是重点,下载的插件放到哪里?     
先启用插件,去后台看下,因为还不知道放哪里就没放,所以会报找不到文件的错误,在浏览器开发控制台看到是
在`static`的目录里面,也有具体的路径了,是`/static/ckeditor/ckeditor/plugins/html5video`.
我们也不能直接放到`static`目录里面,因为开发环境是用不了的,所以建了一个和它同级的目录`static_common`,
所以最后的目录是`/static_common/ckeditor/ckeditor/plugins/html5video`.
然后修改配置
```python
STATIC_URL = "/static/"
STATIC_ROOT = os.path.join(BASE_DIR, "static")
STATICFILES_DIRS = [
    os.path.join(BASE_DIR, "static_common"),
]
```
这样就可以了, 这几个参数的作用可以看文末的参考链接.    
网上有些资料是把插件放到python包的路径,虽然能实现,但是是不可取的.

## 参考链接
 - [STATIC_URL、STATIC_ROOT、STATICFILES_DIRS三者区别](https://www.cnblogs.com/shangping/p/13563439.html)