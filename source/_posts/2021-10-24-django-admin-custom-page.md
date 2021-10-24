---
title: django admin 定制菜单和页面
date: 2021-10-24 05:15:31
tags:
  - django
  - django-admin
---


## 原理
django admin 后台管理的菜单层级是  `app` -> `model`, 所以如果要加顶级菜单就创建一个应用, 要加二级菜单就创建一个模型.

## 示例
### 顶级菜单
创建新应用`z_app`
```
python manage.py startapp z_app
```
修改app显示信息
```python
# apps.py
class ZAppConfig(AppConfig):
    ...
    verbose_name="自定义菜单"
```
在把新的app放入 `INSTALLED_APPS`, 这样就会在后台显示了
```python
# settings.py
INSTALLED_APPS = [
    ...
    z_app.apps.ZAppConfig,
]
```

### 二级菜单
二级菜单就创建一个模型, 这里要注意设置 `managed = False`, 这样就不会在数据库里面创建表.    
但是默认把模型的4个权限插入了权限表, 后面可以用来做权限校验支持.
```
# models.py
class UploadMaterial(models.Model):
    class Meta:
        # 注意设置
        managed = False
        verbose_name = "上传素材"
        verbose_name_plural = verbose_name
```
因为二级菜单点击的链接对应的是 `changelist_view` 方法, 所以我们修改该方法. 
通过查看`admin.ModelAdmin` 的这个方法实现, 保留了权限的校验代码, 这样就可以继承菜单权限的校验.
```
# admin.py
from .models import UploadMaterial

@admin.register(UploadMaterial)
class UploadMaterialAdmin(admin.ModelAdmin):
    def changelist_view(self, request, extra_context=None):
        # 权限校验
        if not self.has_view_or_change_permission(request):
            raise PermissionDenied
            
        # 定制页面
        return render(request, 'z_app/uploadmaterial.html')
```
创建自定义页面
```
# z_app/templates/z_app/uploadmaterial.html
xxxxx
```

## 效果
就用文字来简单描述下效果
```
自定义菜单    
    上传素材
```