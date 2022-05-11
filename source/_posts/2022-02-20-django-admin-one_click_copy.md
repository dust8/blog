---
title: django admin 一键复制
date: 2022-02-20 06:15:31
tags:
  - admin
  - django
---

后台有时会有需要复制一些内容,所以需要一键复制功能来方便复制.

```python
from django.contrib import admin
from django.utils.html import format_html

from .models import Note


@admin.register(Note)
class NoteAdmin(admin.ModelAdmin):
    list_display = [..., "one_click_copy" ]


    @admin.display(description='操作')
    def one_click_copy(self, obj):
        # 需要https
        return format_html(f"""
            <a id="{obj.id}" data-value="{obj.serial_no}" href="javascript:;"
            onclick=";let text=document.getElementById('{obj.id}').getAttribute('data-value');
            navigator.clipboard.writeText(text);">一键复制</a>
        """)
```

```python
@admin.register(Note)
class NoteAdmin(admin.ModelAdmin):
    list_display = [..., "one_click_copy" ]

    @admin.display(description='操作')
    def one_click_copy(self, obj):
        return format_html(f"""
            <a id="{obj.id}" data-value="{obj.serial_no}" href="javascript:;"
            onclick=";
            const oInput = document.createElement('textarea');
            const text=window.location.protocol+'//'+window.location.host+'/core/note/detail/'
                      +document.getElementById('{obj.id}').getAttribute('data-value');
            oInput.value = text;
            oInput.readOnly = 'readOnly';
            document.body.appendChild(oInput);
            oInput.select();
            document.execCommand('copy');
            oInput.blur();
            document.body.removeChild(oInput);">一键复制</a>
        """)
```

## 参考链接

- [display 装饰器](https://docs.djangoproject.com/zh-hans/3.2/ref/contrib/admin/#the-display-decorator)
