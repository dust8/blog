---
title: django admin 大数据量的关联外键
date: 2022-02-20 05:15:31
tags:
  - admin
  - django
---

## 方法1. ModelAdmin.raw_id_fields
会弹出该外键的模型的列表选择
```python
class ArticleAdmin(admin.ModelAdmin):
    raw_id_fields = ("newspaper",)
```

## 方法2. ModelAdmin.autocomplete_fields
会让你输入搜索的关键字来选择搜索结果
```python
class QuestionAdmin(admin.ModelAdmin):
    ordering = ['date_created']
    search_fields = ['question_text']

class ChoiceAdmin(admin.ModelAdmin):
    autocomplete_fields = ['question']
```

## 参考链接
- [ModelAdmin.raw_id_fields](https://docs.djangoproject.com/zh-hans/3.2/ref/contrib/admin/#django.contrib.admin.ModelAdmin.raw_id_fields)
- [ModelAdmin.autocomplete_fields](https://docs.djangoproject.com/zh-hans/3.2/ref/contrib/admin/#django.contrib.admin.ModelAdmin.autocomplete_fields)