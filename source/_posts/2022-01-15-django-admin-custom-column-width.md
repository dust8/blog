---
title: django admin 定制列宽
date: 2022-01-15 05:15:31
tags:
  - admin
  - django
---


起源是在列表页使用了 `list_editable` 来编辑字段,出现金额应该占用很小的列宽,实际上`input`导致很宽,字段多的时候就会出现横向的滚动条,导致不好查看.

## 官方文档
`list_display` 中的字段名也会以 CSS 类的形式出现在 HTML 输出中，在每个 <th> 元素上以 column-<field_name> 的形式出现。例如，这可以用来在 CSS 文件中设置列宽。

```python
class ArticleAdmin(admin.ModelAdmin):
    class Media:
        css = {
            "all": ("my_styles.css",)
        }
        js = ("my_code.js",)
```

## 实际使用修改input
```python
# admin.py
class ProductSKUAdmin(admin.ModelAdmin):
    class Media:
        css = {
            "all": ("css/productsku.css",)
        }
```
```css
# css/productsku.css
.results input{
    max-width:80px;
}
```
`reslult` 这个css类名可以通过浏览的开发者工具查看源代码获得.
同理你想其他的样式也可以.


## 参考链接
- [ModelAdmin 静态资源定义](https://docs.djangoproject.com/zh-hans/3.2/ref/contrib/admin/#modeladmin-asset-definitions)